# SPZ v4 Format Parsing

Also called: SPZ format decoder, SPZ v4, SPZ v4 Reader, ZSTD streams in KMP splat parser, compressed capture reader, parseSplatSpz

## Overview
SPZ v4 format parsing is the method for decoding binary SPZ v4 files containing compressed 3D Gaussian splat data into a usable `SplatCloud` structure. It solves the problem of loading high-fidelity Gaussian splat captures in environments lacking native system-level decompression libraries, such as web browsers. It is distinguished from earlier SPZ versions (1–3) by using ZSTD compression per attribute instead of a single gzip stream, and from generic PLY parsers by its specific handling of quantized fields and coordinate system transformations [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A binary file with the `.spz` extension. The file must start with the magic number `0x5053474e` (ASCII "SPZN") [S2].
*   **Output:** A `SplatCloud` object containing decoded Gaussian attributes (position, covariance, color, spherical harmonics). The output coordinate system is either the file's native "Right-Up-Back" (RUB) or a requested "Right-Down-Front" (RDF) compatible with PLY standards [S2].

**Data Structures:**
The SPZ v4 format stores attributes in separate ZSTD-compressed streams. The parser must maintain a buffer for each attribute stream. Quantized fields (e.g., positions, SH coefficients) are stored as integers and require specific scaling constants to be converted to floating-point values. These constants are provided by the reference library [S2].

**Numbered Steps:**
1.  **Header Validation:** Read the first 4 bytes. Verify they equal `0x5053474e`. If not, abort.
2.  **Version Detection:** Determine the SPZ version. For v4, proceed to step 3. (For v1–3, use gzip decompression [S2]).
3.  **Stream Initialization:** Identify the number of attribute streams defined in the header. Initialize a ZSTD decompressor instance for each stream. The KTX2 ZSTD decoder is recommended for web compatibility [S2].
4.  **Decompression:** For each attribute stream:
    a. Read the compressed data block.
    b. Decompress using the ZSTD decoder to obtain raw quantized bytes.
5.  **Quantization Decoding:** For each quantized field:
    a. Read the quantization parameters (offset, scale, bits) from the reference library constants [S2].
    b. Convert integer values to float: $v_{float} = (v_{int} - offset) \times scale$.
6.  **Coordinate System Handling:**
    a. By default, return the cloud in the file's native Right-Up-Back (RUB) axes [S2].
    b. If a PLY coordinate system is requested:
        i. Transform axes from RUB to Right-Down-Front (RDF). This typically involves flipping the Y and Z axes.
        ii. Flip the signs of the spherical harmonics coefficients to match the coordinate change [S2].
7.  **Validation:** Compare decoded reference positions against expected values using a relative tolerance to account for precision differences inherent in the quantization process [S1, unofficial].
8.  **Extension Check:** If the file name includes an extension indicating a coordinate axis move, refuse to parse the file by name rather than rendering it mirrored, to prevent incorrect coordinate mapping [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def parse_spz_v4(file_path: str, request_ply_coords: bool = False) -> SplatCloud:
    # 1. Header Validation
    with open(file_path, 'rb') as f:
        magic = f.read(4)
        if magic != b'SPZN':
            raise ValueError("Invalid SPZ magic number")
        
        # 2. Version Detection (Assumed v4 based on context)
        # Read header to determine attribute count and stream sizes
        header = parse_spz_header(f)
        
    # 3. Stream Initialization
    # Initialize ZSTD decoders for each attribute stream
    # Using KTX2 ZSTD decoder for web compatibility [S2]
    decoders = [ZstdDecoder() for _ in header.attribute_streams]
    
    splat_cloud = SplatCloud()
    
    # 4. Decompression
    for i, stream_info in enumerate(header.attribute_streams):
        f.seek(stream_info.offset)
        compressed_data = f.read(stream_info.size)
        raw_bytes = decoders[i].decompress(compressed_data)
        
        # 5. Quantization Decoding
        # Use reference library constants for quantization params [S2]
        quant_params = get_quantization_params(stream_info.attribute_name)
        
        if stream_info.attribute_name == 'position':
            # Decode positions
            positions = decode_quantized_field(raw_bytes, quant_params)
            splat_cloud.positions = positions
            
        elif stream_info.attribute_name == 'sh':
            # Decode Spherical Harmonics
            sh_coeffs = decode_quantized_field(raw_bytes, quant_params)
            splat_cloud.sh_coeffs = sh_coeffs
            
        # ... handle other attributes (covariance, opacity, etc.)

    # 6. Coordinate System Handling
    if request_ply_coords:
        # Transform from Right-Up-Back (RUB) to Right-Down-Front (RDF)
        # Flip Y and Z axes
        splat_cloud.positions[:, 1] *= -1
        splat_cloud.positions[:, 2] *= -1
        
        # Flip signs of spherical harmonics to match coordinate change [S2]
        splat_cloud.sh_coeffs = flip_sh_signs(splat_cloud.sh_coeffs)
        
    # 7. Validation
    # Compare with reference positions using relative tolerance [S1, unofficial]
    validate_positions(splat_cloud.positions, relative_tolerance=1e-5)
    
    # 8. Extension Check
    # If file name indicates axis move, refuse [S2]
    if has_axis_move_extension(file_path):
        raise ValueError("File with axis-move extension refused by name")
        
    return splat_cloud

def decode_quantized_field(raw_bytes: bytes, params: QuantParams) -> np.ndarray:
    # Convert quantized integers to float using reference constants [S2]
    # v_float = (v_int - offset) * scale
    ints = np.frombuffer(raw_bytes, dtype=params.dtype)
    return (ints - params.offset) * params.scale
```

**Parameters:**
*   `relative_tolerance`: Used for position validation. Value not explicitly given, but "relative tolerance" is required [S1, unofficial].
*   `request_ply_coords`: Boolean flag. Default `False` (returns RUB). If `True`, returns RDF [S2].

**Edge Cases and Pitfalls:**
*   **Precision:** SPZ v4 reference positions must be compared with a relative tolerance due to quantization precision differences [S1, unofficial].
*   **Coordinate Mapping:** Files with extensions indicating axis moves are refused by name, not rendered mirrored, to prevent incorrect mapping [S2].
*   **Web Compatibility:** Native system decoders are not required; the parser must use web-compatible ZSTD (e.g., KTX2) [S2].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of splats and attributes, dominated by ZSTD decompression.
*   **Memory Complexity:** Proportional to the size of the decompressed `SplatCloud`.
*   **Strengths:** Enables web-compatible decoding without native system libraries [S2]. Supports high-fidelity data with SH degree 4 [S2].
*   **Limitations:** Requires ZSTD implementation. Coordinate system conversion is non-trivial for SH coefficients [S2].
*   **When to use:** When loading SPZ v4 files in web environments or when PLY coordinate compatibility is required.
*   **When not to use:** When native gzip decompression is sufficient (SPZ v1–3) or when coordinate system conversion is not needed and RUB is acceptable.

## Background
The SPZ format was developed to provide a compressed, efficient storage format for 3D Gaussian splats. SPZ v4 specifically introduces ZSTD compression per attribute, improving compression ratios and enabling web-based parsing. The parser is implemented in the `flutter3d` package to enable web-compatible SPZ decoding [S2]. The format is identified by the magic number `0x5053474e` [S2].

## Sources
[S1] Explore: Gaussian splatting, OpenUSD, AI world generation, volumetric video — study, https://github.com/sceneview/sceneview/issues/4023, fetched 2026-10-27 (unofficial)
[S2] KHR_gaussian_splatting_compression_spz_2, https://github.com/KhronosGroup/glTF/pull/2531, fetched 2026-10-27
