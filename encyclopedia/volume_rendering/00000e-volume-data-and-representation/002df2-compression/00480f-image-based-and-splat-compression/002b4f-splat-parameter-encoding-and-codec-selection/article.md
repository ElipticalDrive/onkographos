# Splat parameter encoding and codec selection

Also called: CreateOtherDataJob, Luxar store compression, Other Data Encoding, Other Data Encoding (Rotation, Scale, SH Index), quantised splat encoding

## Overview
This method compresses the non-position attributes of 3D Gaussian splats (rotation, scale, and spherical harmonics indices) into a compact byte array, applying a specific codec policy to minimize storage size while maintaining rendering fidelity. It solves the problem of high memory bandwidth and storage costs associated with raw floating-point splat data by quantizing parameters and applying lossless compression. It differs from raw storage by introducing discrete compression levels and from position encoding by handling multi-component vectors and optional metadata indices. It builds on [Image-Based and Splat Compression](12178-image-based-and-splat-compression.md).

## How it works
**Inputs and Outputs:**
- **Input:** A collection of splat parameters including rotation quaternions ($q \in \mathbb{R}^4$), scale vectors ($s \in \mathbb{R}^3$), and optionally spherical harmonics (SH) indices ($idx \in \mathbb{Z}$).
- **Output:** A contiguous byte array containing the encoded parameters, ready for codec application.

**Data Structures and Layout:**
The output buffer is structured as a sequence of fixed-size records per splat. The layout depends on the selected formats:
1.  **Rotation:** 4 bytes (32-bit integer).
2.  **Scale:** Variable bytes, determined by `m_ScaleFormat` (e.g., 6 bytes for Float16, 12 bytes for Float32).
3.  **SH Index:** 2 bytes (16-bit unsigned integer), present only if `m_SplatSHIndices` is provided.

**Parameters:**
-   `m_ScaleFormat`: The encoding format for scale vectors (e.g., Float32, Float16, Uint8).
-   `m_SplatSHIndices`: An optional array of integers containing SH indices. If null/empty, no SH index bytes are written.
-   **Codec:** Blosc-zstd level 9.
-   **Shuffle Policy:** Byte shuffle is applied to multi-byte integer codes. Single-byte codes and floating-point arrays are left unshuffled.

**Steps:**
1.  **Initialize Buffer:** Calculate the total output buffer length. For each splat, the size is the sum of the rotation size (4 bytes), the scale size (determined by `m_ScaleFormat`), and the SH index size (2 bytes if present). Round the total length up to the next multiple of 8 bytes for alignment.
2.  **Encode Rotation:** For each splat, normalize the rotation quaternion $q = (x, y, z, w)$. Encode it into a 32-bit integer using `EncodeQuatToNorm10(q)`. This maps the normalized components to a 10-bit integer representation. Write the 4-byte integer to the buffer.
3.  **Encode Scale:** For each splat, encode the scale vector $s$ using the specified `m_ScaleFormat`. This typically involves quantizing the float values to the target bit-depth (e.g., 16-bit or 8-bit) and packing them into the buffer.
4.  **Encode SH Index (Optional):** If `m_SplatSHIndices` is provided, write the 16-bit unsigned integer index for the current splat to the buffer.
5.  **Apply Shuffle:** Apply byte shuffle to the multi-byte integer codes (rotation and SH index) to improve compression efficiency. Do not apply shuffle to single-byte codes or floating-point arrays.
6.  **Compress:** Apply the Blosc-zstd codec (level 9) to the resulting byte array.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def encode_splat_parameters(splats, scale_format, sh_indices=None):
    """
    Encodes splat parameters into a compressed byte array.
    
    Args:
        splats: List of splat objects with attributes:
            - rot: Quaternion (x, y, z, w)
            - scale: Vector (sx, sy, sz)
        scale_format: Enum or int specifying scale encoding (e.g., 'float16', 'float32')
        sh_indices: Optional list of integers for SH indices
        
    Returns:
        bytes: Compressed byte array
    """
    # Step 1: Calculate buffer size
    # Rotation is always 4 bytes (32-bit int)
    rotation_size = 4
    # Scale size depends on format
    scale_size = get_vector_size(scale_format)
    # SH index is 2 bytes if present
    sh_index_size = 2 if sh_indices is not None else 0
    
    bytes_per_splat = rotation_size + scale_size + sh_index_size
    total_bytes = len(splats) * bytes_per_splat
    # Round up to multiple of 8
    buffer_size = next_multiple_of(total_bytes, 8)
    
    buffer = bytearray(buffer_size)
    
    # Step 2-4: Encode each splat
    for i, splat in enumerate(splats):
        offset = i * bytes_per_splat
        
        # Encode Rotation
        # Normalize quaternion
        q = normalize_quaternion(splat.rot)
        # Encode to 32-bit int (10-bit per component)
        rot_int = encode_quat_to_norm10(q)
        # Write 4 bytes
        write_int32(buffer, offset, rot_int)
        
        # Encode Scale
        # Quantize and pack scale vector based on format
        scale_bytes = encode_vector(splat.scale, scale_format)
        write_bytes(buffer, offset + rotation_size, scale_bytes)
        
        # Encode SH Index
        if sh_indices is not None:
            sh_idx = sh_indices[i]
            # Write 2 bytes
            write_uint16(buffer, offset + rotation_size + scale_size, sh_idx)
            
    # Step 5: Apply Shuffle
    # Apply byte shuffle to multi-byte integer codes
    # Note: In practice, this is often done by the codec or a pre-processor
    # For multi-byte integers (rotation, sh_index), shuffle bytes
    # For floats (scale), do not shuffle
    apply_byte_shuffle(buffer, bytes_per_splat, scale_format)
    
    # Step 6: Compress
    # Use Blosc-zstd level 9
    compressed = blosc_compress(buffer, clevel=9, cname='zstd')
    
    return compressed

def encode_quat_to_norm10(q):
    """
    Encodes a normalized quaternion into a 32-bit integer.
    Each component (x, y, z, w) is mapped to a 10-bit integer.
    """
    # Map [-1, 1] to [0, 1023]
    def map_to_10bit(val):
        return int((val + 1.0) * 511.5)
    
    x = map_to_10bit(q.x)
    y = map_to_10bit(q.y)
    z = map_to_10bit(q.z)
    w = map_to_10bit(q.w)
    
    # Pack into 32-bit integer
    return (x << 22) | (y << 12) | (z << 2) | (w >> 8) # Simplified packing, actual implementation may vary

def apply_byte_shuffle(buffer, bytes_per_splat, scale_format):
    """
    Applies byte shuffle to multi-byte integer codes.
    """
    # Determine which parts are integers vs floats
    # Rotation is always integer (4 bytes)
    # SH index is integer (2 bytes) if present
    # Scale is float if format is float, integer if quantized
    
    # For simplicity, assume rotation and sh_index are shuffled
    # Scale is shuffled only if it is integer-based
    for i in range(0, len(buffer), bytes_per_splat):
        # Shuffle rotation (bytes 0-3)
        shuffle_bytes(buffer, i, i + 4)
        
        # Shuffle SH index if present
        if 2 in [bytes_per_splat - 4 - get_vector_size(scale_format)]: # Check if sh_index exists
            sh_offset = 4 + get_vector_size(scale_format)
            shuffle_bytes(buffer, i + sh_offset, i + sh_offset + 2)
            
        # Shuffle scale if integer-based
        if not is_float_format(scale_format):
            scale_offset = 4
            scale_end = 4 + get_vector_size(scale_format)
            shuffle_bytes(buffer, i + scale_offset, i + scale_end)

def shuffle_bytes(buffer, start, end):
    """
    Reverses the byte order of a segment in the buffer.
    """
    segment = buffer[start:end]
    buffer[start:end] = segment[::-1]
```

**Edge Cases and Pitfalls:**
-   **Precision Loss:** Quantization of rotation and scale introduces error. The quantization PSNR loss vs original is at most $-0.679$ dB (median $-0.012$ dB) [S1, unofficial].
-   **Codec Anomaly:** A level-dependent Blosc shuffle anomaly occurs on 64 KiB chunks. This is mitigated by using atom-aligned chunks and columnar delta prefiltering on centres [S1, unofficial].
-   **SH Index Absence:** If `m_SplatSHIndices` is not provided, the SH index bytes are omitted, reducing the per-splat size.

## Subtopics

### [Ksplat Decompression](6899-ksplat-decompression.md)
Ksplat Decompression reads and decompresses `.ksplat` files, which contain spatially bucketed and quantized 3D Gaussian Splat data, into a structured `DataTable` format suitable for rendering. It handles position quantization, scale/rotation compression, and spherical harmonics decoding. Unlike raw PLY formats, it supports three distinct compression modes (Float32, Float16, Uint8) to balance file size and precision, and uses spatial bucketing to optimize position storage.

### [Splat Quality Format Mapping](17045-splat-quality-format-mapping.md)
Splat Quality Format Mapping is a codec selection strategy that maps a target quality level to specific data storage formats for position, scale, color, and spherical harmonics (SH) coefficients. It balances visual fidelity against memory usage and download size by selecting from a predefined set of compression levels. It differs from raw storage by introducing a discrete set of presets (VeryHigh to VeryLow) that dictate the bit-depth and encoding scheme for each splat attribute.

### Ksplat Format
A trimmed-down and compressed version of the original `.ply` files, designed for fastest loading times as its format matches the internal format for splat data [S3, unofficial]. Converting scenes to `.ksplat` results in fastest loading times because the format matches the internal splat data format, whereas progressive loading of `.ply`/`.splat` skips cache-optimized ordering [S3, unofficial].

### Position Data Encoding
Encodes the position vectors of splats into a byte array using a specified vector format, processed in parallel via a Burst job [S2, unofficial]. The total output buffer length is calculated, rounding up to the next multiple of 8 bytes: `NextMultipleOf(inputSplats.Length * GaussianSplatAsset.GetVectorSize(formatPos), 8)` [S2, unofficial]. The job is executed with a chunk size of 8192 splats [S2, unofficial]. The encoding format for the vector data is defined in `GaussianSplatAsset.VectorFormat` [S2, unofficial].

## Operating characteristics
-   **Time Complexity:** Encoding is linear with respect to the number of splats $N$. Decompression is also linear, with throughput measured at 138–215 MB/s in-browser and 326–592 MB/s natively on identical chunk sets on a benchmark workstation [S1, unofficial].
-   **Memory Complexity:** The compressed store size is 8.8–12.7 bytes/splat, down from 30–37 bytes/splat for raw data [S1, unofficial]. The shipped rule storage is 9.63 bytes/splat (10.09 bytes/splat if re-chunked to 64 KiB without prefilter) [S1, unofficial]. An optimized unshuffled configuration achieves 9.50 bytes/splat [S1, unofficial].
-   **Performance:** The codec policy (Blosc-zstd level 9) provides a 14.3% reduction compared to a uniform Blosc-zstd level-3 bit-shuffle baseline [S1, unofficial]. The optimized unshuffled config is 15.6% below the baseline at 64 KiB [S1, unofficial].
-   **Strengths:** Significant reduction in storage and memory bandwidth. Fast decoding speeds.
-   **Limitations:** Quantization introduces small PSNR loss. Codec shuffle anomaly on 64 KiB chunks requires mitigation [S1, unofficial].
-   **When to Use:** When storage size and loading speed are critical, and slight quantization error is acceptable.
-   **When Not to Use:** When maximum floating-point precision is required for scientific accuracy.

## Background
This method originates from the Luxar project's work on scalable interactive web visualization of multidimensional scientific data, specifically focusing on Gaussian splatting for microscopy [S1, unofficial]. The implementation details, including the specific encoding functions and job structures, are derived from the `nigelhartman/worldlabs_unity` repository [S2, unofficial]. The `.ksplat` format and its benefits for loading times are documented in the `mkkellogg/GaussianSplats3D` project [S3, unofficial].

## Sources
[S1] Luxar: Gaussian splatting for microscopy and scalable interactive web visualisation of multidimensional scientific data, https://doi.org/10.64898/2026.09.22.753676, fetched 2026-10-26 (unofficial)
[S2] nigelhartman/worldlabs_unity, https://github.com/nigelhartman/worldlabs_unity, fetched 2026-10-27 (unofficial)
[S3] mkkellogg/GaussianSplats3D, https://github.com/mkkellogg/GaussianSplats3D, fetched 2026-10-27 (unofficial)
