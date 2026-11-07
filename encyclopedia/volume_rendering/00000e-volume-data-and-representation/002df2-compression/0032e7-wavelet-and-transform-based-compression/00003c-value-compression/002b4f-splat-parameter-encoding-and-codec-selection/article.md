# Splat parameter encoding and codec selection

Also called: Luxar store compression, quantised splat encoding

## Overview
This method defines the binary layout and compression pipeline for 3D Gaussian Splat parameters to minimize file size while maintaining rendering fidelity. It solves the problem of excessive storage and memory bandwidth required by raw float32 splat data by applying specific quantization schemes per parameter type and selecting an optimal codec configuration. It is distinguished from general volume compression by its focus on the specific statistical properties of Gaussian splat attributes (position, scale, rotation, amplitude) and its optimization for both native and WebAssembly-based browser decoding [S1, unofficial].

## How it works
**Inputs and Outputs**
Input: 3D Gaussian Splat parameters (centres, Cholesky factors for scale/rotation, amplitudes) originally in float32.
Output: A compressed `.ksplat` file containing quantized and encoded parameter arrays [S2, unofficial].

**Data Structures and Layout**
The store consists of four parameter arrays, each encoded according to its dynamic range and statistical properties [S1, unofficial]:
1.  **Centres**: Quantized to `uint16`.
2.  **Cholesky Factors**: Quantized to `uint8` (certified precision).
3.  **Amplitudes**: Selected based on the array's dynamic range:
    *   Bounded-linear `uint8` or `uint16`.
    *   Geometric-log `uint16` grid.

**Codec and Filtering Policy**
The compression uses Blosc with the zstd compressor at level 9 [S1, unofficial]. A width-aware per-dtype compressor policy is applied:
1.  **Byte Shuffle**: Applied to multi-byte integer codes (e.g., `uint16` centres).
2.  **No Byte Shuffle**: Applied to single-byte codes (`uint8` Cholesky/amplitudes) and floating-point arrays.
3.  **Optimization**: In the optimized configuration, log-companded amplitude codes are left unshuffled to reduce size by 1.44% [S1, unofficial].

**Pitfalls and Edge Cases**
*   **Shuffle Anomaly**: A level-dependent Blosc shuffle anomaly occurs on 64 KiB chunks. This is mitigated by using atom-aligned chunks and applying columnar delta prefiltering specifically to the centres array [S1, unofficial].
*   **Chunking**: Re-chunking to 64 KiB without the prefilter increases size to 10.09 bytes/splat [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def encode_splats(splats_float32):
    """
    Encodes splat parameters into the Luxar store format.
    splats_float32: List of dicts with keys 'center' (vec3), 'cholesky' (mat3), 'amplitude' (float).
    """
    # 1. Quantize Centres to uint16
    centres_uint16 = quantize_uint16(splats_float32.centers)
    
    # 2. Quantize Cholesky Factors to uint8
    cholesky_uint8 = quantize_uint8(splats_float32.cholesky)
    
    # 3. Select Amplitude Encoding based on dynamic range
    amp_dynamic_range = calculate_dynamic_range(splats_float32.amplitudes)
    if amp_dynamic_range < threshold_linear:
        amplitudes_encoded = quantize_bounded_linear_uint8(splats_float32.amplitudes)
        is_multi_byte = False
    else:
        amplitudes_encoded = quantize_bounded_linear_uint16(splats_float32.amplitudes)
        is_multi_byte = True
        
    # Check for geometric-log suitability
    if amp_dynamic_range > threshold_geometric:
        amplitudes_encoded = quantize_geometric_log_uint16(splats_float32.amplitudes)
        is_multi_byte = True
        use_shuffle = False # Optimized config leaves these unshuffled
    
    # 4. Apply Blosc Compression with specific policies
    # Centres: Multi-byte integer -> Byte Shuffle enabled
    centres_compressed = blosc_compress(
        data=centres_uint16, 
        clevel=9, 
        cname='zstd', 
        shuffle=True
    )
    
    # Cholesky: Single-byte integer -> Byte Shuffle disabled
    cholesky_compressed = blosc_compress(
        data=cholesky_uint8, 
        clevel=9, 
        cname='zstd', 
        shuffle=False
    )
    
    # Amplitudes: Depends on encoding
    # If multi-byte (uint16 or bounded-linear uint16), shuffle enabled unless optimized
    # If single-byte (uint8), shuffle disabled
    # If geometric-log (uint16) in optimized config, shuffle disabled
    amplitudes_compressed = blosc_compress(
        data=amplitudes_encoded, 
        clevel=9, 
        cname='zstd', 
        shuffle=use_shuffle
    )
    
    # 5. Write to .ksplat file
    write_ksplat_file(centres_compressed, cholesky_compressed, amplitudes_compressed)
```

## Subtopics

### [Ksplat Decompression](6899-ksplat-decompression.md)
Ksplat Decompression reads and decompresses `.ksplat` files, which contain spatially bucketed and quantized 3D Gaussian Splat data, into a structured `DataTable` format suitable for rendering. It handles position quantization, scale/rotation compression, and spherical harmonics decoding. Unlike raw PLY formats, it supports three distinct compression modes (Float32, Float16, Uint8) to balance file size and precision, and uses spatial bucketing to optimize position storage.

## Operating characteristics
**Performance**
*   **Store Size**: 8.8–12.7 bytes/splat, down from 30–37 bytes/splat for float32 [S1, unofficial].
*   **Shipped Rule Storage**: 9.63 bytes/splat. If re-chunked to 64 KiB without prefilter, 10.09 bytes/splat (14.3% below uniform Blosc-zstd level-3 bit-shuffle baseline). Optimized unshuffled config: 9.50 bytes/splat (15.6% below baseline at 64 KiB) [S1, unofficial].
*   **Quantization Fidelity**: PSNR loss vs original is at most −0.679 dB (median −0.012 dB). Quantised render PSNR vs float32 render is 59–88 dB [S1, unofficial].
*   **Decoding Throughput**: 138–215 MB/s in-browser (WebAssembly), 326–592 MB/s natively on identical chunk sets [S1, unofficial].

**Strengths and Limitations**
*   **Strengths**: Significant reduction in file size and memory footprint; optimized for both native and browser environments; high rendering fidelity (PSNR > 59 dB) [S1, unofficial].
*   **Limitations**: Requires specific codec configuration (Blosc-zstd level 9) and prefiltering to avoid anomalies; not compatible with progressive loading optimizations of standard `.ply` or `.splat` files [S2, unofficial].

## Background
This method originates from the Luxar project, which focuses on Gaussian splatting for microscopy and scalable interactive web visualization of multidimensional scientific data. The approach was developed to address the limitations of standard `.ply` and `.splat` formats in web-based viewers, particularly regarding loading times and cache optimization [S1, S2, unofficial]. The `.ksplat` format was introduced as a trimmed-down and compressed version of original `.ply` files, designed to match the internal format for splat data, thereby enabling the fastest loading times [S2, unofficial].

## Sources
[S1] Luxar: Gaussian splatting for microscopy and scalable interactive web visualisation of multidimensional scientific data, https://doi.org/10.64898/2026.09.22.753676, fetched 2026-10-26 (unofficial)
[S2] mkkellogg/GaussianSplats3D, https://github.com/mkkellogg/GaussianSplats3D, fetched 2026-10-27 (unofficial)
