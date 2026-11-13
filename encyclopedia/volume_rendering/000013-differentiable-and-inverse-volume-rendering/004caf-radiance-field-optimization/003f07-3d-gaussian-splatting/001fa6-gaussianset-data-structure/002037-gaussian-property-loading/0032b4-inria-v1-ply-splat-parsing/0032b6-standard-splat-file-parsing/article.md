# Standard Splat File Parsing

Also called: SplatParser.parseStandardSplatToUncompressedSplatArray, decodeSplatToDataTable, readSplat

## Overview
Standard Splat File Parsing decodes the binary `.splat` file format into an uncompressed splat array or data table, handling the specific binary layout of position, scale, color, and rotation data [S1, unofficial][S2]. It serves as the foundational parser for loading 3D Gaussian Splatting assets, converting raw binary records into structured geometric and appearance properties [S1, unofficial][S2]. This method provides both eager decoding for validation and lazy chunk-based reading for memory efficiency [S1, unofficial][S2].

## How it works
**Inputs and Outputs:**
- **Input:** A binary buffer containing the `.splat` file data.
- **Output:** An uncompressed data structure containing splat properties. This can be a custom `SplatArray` object (as in [S1, unofficial]) or a `DataTable` with typed columns (as in [S2]).

**Data Structures and Layout:**
The standard `.splat` format uses a fixed row size of 32 bytes per splat [S1, unofficial][S2]. The fields are packed in the following order:
1. **Center (Position):** 12 bytes. Three `Float32` values (x, y, z) [S1, unofficial][S2].
2. **Scale:** 12 bytes. Three `Float32` values (scale_x, scale_y, scale_z) [S1, unofficial][S2].
3. **Color:** 4 bytes. Four `Uint8` values [S1, unofficial][S2].
4. **Rotation:** 4 bytes. Four `Uint8` values representing a quaternion [S1, unofficial][S2].

**Parameters:**
- `SplatParser.RowSizeBytes`: 32 [S1, unofficial].
- `BYTES_PER_SPLAT`: 32 [S2].
- `SplatParser.CenterSizeBytes`: 12 [S1, unofficial].
- `SplatParser.ScaleSizeBytes`: 12 [S1, unofficial].
- `SplatParser.ColorSizeBytes`: 4 [S1, unofficial].

**Numbered Steps:**
1. **Validate File Size:** Calculate the total file size in bytes. If `fileSize % 32 !== 0`, throw an error [S2]. Calculate `numSplats = fileSize / 32`. If `numSplats === 0`, throw an error [S2].
2. **Initialize Output Structure:** For eager decoding [S2], initialize `Float32Array` columns for x, y, z, scale_0, scale_1, scale_2, f_dc_0, f_dc_1, f_dc_2, opacity, rot_0, rot_1, rot_2, rot_3, each of size `numSplats`.
3. **Iterate and Decode:** Loop `i` from 0 to `numSplats - 1`. Calculate the byte offset `o = i * 32`.
   - **Position:** Read 12 bytes starting at `o` as three `Float32` values. Assign to x, y, z [S2].
   - **Scale:** Read 12 bytes starting at `o + 12` as three `Float32` values. Assign to scale_0, scale_1, scale_2 [S2].
   - **Color:** Read 4 bytes starting at `o + 24` as four `Uint8` values. Assign the first three to f_dc_0, f_dc_1, f_dc_2 and the fourth to opacity [S2].
   - **Rotation:** Read 4 bytes starting at `o + 28` as four `Uint8` values. Let these be `u0, u1, u2, u3` [S1, unofficial].
4. **Quaternion Normalization:** Convert the raw uint8 rotation bytes to a normalized quaternion [S1, unofficial]:
   - Map the uint8 values to floats in the range [-1, 1]:
     $$ w = \frac{u_0 - 128}{128} $$
     $$ x = \frac{u_1 - 128}{128} $$
     $$ y = \frac{u_2 - 128}{128} $$
     $$ z = \frac{u_3 - 128}{128} $$
   - Construct a quaternion $(x, y, z, w)$ and normalize it to ensure it is a unit quaternion [S1, unofficial].
   - Assign the normalized components to rot_0, rot_1, rot_2, rot_3 [S2].
5. **Finalize:** Return the populated data structure [S1, unofficial][S2].

**Pseudocode (Eager Decoding):**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import struct
import math

class DataTable:
    def __init__(self, columns, transform=None):
        self.columns = columns
        self.transform = transform

class Transform:
    PLY = "PLY"

def decode_splat_to_datatable(file_buffer: bytes) -> DataTable:
    file_size = len(file_buffer)
    if file_size % 32 != 0:
        raise ValueError("Invalid .splat file: file size is not a multiple of 32 bytes")
    
    num_splats = file_size // 32
    if num_splats == 0:
        raise ValueError("Invalid .splat file: no splats found")

    # Initialize arrays
    x = [0.0] * num_splats
    y = [0.0] * num_splats
    z = [0.0] * num_splats
    scale_0 = [0.0] * num_splats
    scale_1 = [0.0] * num_splats
    scale_2 = [0.0] * num_splats
    f_dc_0 = [0.0] * num_splats
    f_dc_1 = [0.0] * num_splats
    f_dc_2 = [0.0] * num_splats
    opacity = [0.0] * num_splats
    rot_0 = [0.0] * num_splats
    rot_1 = [0.0] * num_splats
    rot_2 = [0.0] * num_splats
    rot_3 = [0.0] * num_splats

    for i in range(num_splats):
        o = i * 32
        
        # Position (Float32, little-endian)
        x[i] = struct.unpack_from('<f', file_buffer, o)[0]
        y[i] = struct.unpack_from('<f', file_buffer, o + 4)[0]
        z[i] = struct.unpack_from('<f', file_buffer, o + 8)[0]
        
        # Scale (Float32, little-endian)
        scale_0[i] = struct.unpack_from('<f', file_buffer, o + 12)[0]
        scale_1[i] = struct.unpack_from('<f', file_buffer, o + 16)[0]
        scale_2[i] = struct.unpack_from('<f', file_buffer, o + 20)[0]
        
        # Color and Opacity (Uint8)
        r = file_buffer[o + 24]
        g = file_buffer[o + 25]
        b = file_buffer[o + 26]
        op = file_buffer[o + 27]
        
        f_dc_0[i] = r
        f_dc_1[i] = g
        f_dc_2[i] = b
        opacity[i] = op
        
        # Rotation (Uint8 -> Normalized Quaternion)
        u0 = file_buffer[o + 28]
        u1 = file_buffer[o + 29]
        u2 = file_buffer[o + 30]
        u3 = file_buffer[o + 31]
        
        # Map to [-1, 1] per source mapping
        w = (u0 - 128) / 128.0
        qx = (u1 - 128) / 128.0
        qy = (u2 - 128) / 128.0
        qz = (u3 - 128) / 128.0
        
        # Normalize quaternion
        norm = math.sqrt(qx**2 + qy**2 + qz**2 + w**2)
        if norm == 0:
            qx, qy, qz, w = 0.0, 0.0, 0.0, 1.0
        else:
            qx /= norm
            qy /= norm
            qz /= norm
            w /= norm
            
        rot_0[i] = qx
        rot_1[i] = qy
        rot_2[i] = qz
        rot_3[i] = w

    return DataTable({
        'x': x, 'y': y, 'z': z,
        'scale_0': scale_0, 'scale_1': scale_1, 'scale_2': scale_2,
        'f_dc_0': f_dc_0, 'f_dc_1': f_dc_1, 'f_dc_2': f_dc_2,
        'opacity': opacity,
        'rot_0': rot_0, 'rot_1': rot_1, 'rot_2': rot_2, 'rot_3': rot_3
    }, transform=Transform.PLY)
```

**Edge Cases and Pitfalls:**
- **Invalid File Size:** The file size must be a multiple of 32 bytes [S2].
- **Zero Norm Quaternion:** If the normalized quaternion has a zero norm, it is degenerate. The implementation should handle this, potentially by defaulting to an identity quaternion or zeroing components [G].

## Subtopics

### [Antimatter15 .splat File Parsing](6902-antimatter15-splat-file-parsing.md)
Implements a lazy `ChunkSource` for reading `.splat` files, supporting contiguous and scatter-gather reads without loading the entire file into memory. It decodes DC-only color and geometric data on demand. Choose this when memory efficiency is critical for large scenes.

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the number of splats [G].
- **Memory Complexity:** $O(N)$ for eager decoding, storing all properties in contiguous arrays [G].
- **Measured Performance:** Not given in the sources.
- **Strengths:** Simple binary layout allows for fast, straightforward parsing. No header parsing overhead.
- **Limitations:** No spherical harmonic data, limiting color accuracy to DC only. No compression, leading to large file sizes for high-resolution scenes.
- **When to use:** When loading standard `.splat` files for immediate visualization or further processing.
- **When not to use:** When spherical harmonic color variation is required, or when memory constraints prevent loading the full dataset.

## Background
The standard `.splat` file format was developed as a compact, header-less binary format for storing 3D Gaussian Splatting data. The format simplifies parsing by removing the complexity of PLY headers and SH coefficients, focusing on the essential geometric and appearance properties. Not given in the sources.

## Sources
[S1] mkkellogg/GaussianSplats3D, https://github.com/mkkellogg/GaussianSplats3D, fetched 2026-10-27 (unofficial)
[S2] playcanvas/splat-transform, https://github.com/playcanvas/splat-transform, fetched 2026-10-27
