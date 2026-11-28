# Chunked Bounding Box Computation and Normalization

Also called: ChunkInfo Generation, Gaussian Splat Chunking

## Overview
This method preprocesses a list of 3D Gaussian splats by dividing them into fixed-size chunks, computing per-chunk bounding boxes for position, scale, color (DC0 + opacity), and spherical harmonics (SH) coefficients, and normalizing the data within each chunk to the range $[0, 1]$ [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A list of Gaussian splats (`splatData`), where each splat contains position (float3), scale (float3), rotation (quaternion), color DC0 (float3), opacity (float), and SH coefficients (sh1 through shF) [S1, unofficial].
- **Output:** Normalized splat data and a `ChunkInfo` structure containing the bounding box bounds for each chunk [S1, unofficial].

**Data Structures:**
- `ChunkInfo`: Contains bounds for each chunk [S1, unofficial].
  - `posX`, `posY`, `posZ`: Each is a `float2` storing `(min, max)` for the respective axis [S1, unofficial].
  - `sclX`, `sclY`, `sclZ`: Each is a `uint` packing two 16-bit floats (f16) representing min and max scale [S1, unofficial].
  - `colR`, `colG`, `colB`, `colA`: Each is a `uint` packing two f16 representing min and max color/opacity [S1, unofficial].
  - `shR`, `shG`, `shB`: Each is a `uint` packing two f16 representing min and max SH coefficients for the respective color channel [S1, unofficial].

**Numbered Steps:**
1. Initialize min/max accumulators for position (float3), scale (float3), color (float4, combining dc0.xyz and opacity.w), and SH coefficients (float3, accumulating sh1 through shF) to positive/negative infinity respectively [S1, unofficial].
2. Calculate the start (`splatBegin`) and end (`splatEnd`) indices for the current chunk `chunkIdx`, clamping to the total `splatData.Length` [S1, unofficial].
3. Iterate through each splat `i` from `splatBegin` to `splatEnd` [S1, unofficial].
   - Apply a power transformation to the scale: `s.scale = math.pow(s.scale, 1.0f / 8.0f)` [S1, unofficial].
   - Apply a square centered 0-1 normalization to the opacity: `s.opacity = GaussianUtils.SquareCentered01(s.opacity)` [S1, unofficial].
   - Update chunk min/max bounds for position, scale, color (float4 of dc0.xyz and opacity.w), and SH coefficients (sh1 through shF). Note: SH bounds are updated individually for sh1-shF [S1, unofficial].
4. Enforce a minimum spread of `1.0e-5f` on the max bounds to prevent division by zero during normalization: `chunkMaxX = math.max(chunkMaxX, chunkMinX + 1.0e-5f)` for pos, scl, col, and shs [S1, unofficial].
5. Store the computed bounds in a `ChunkInfo` struct. Position bounds are stored as `float2` pairs (min, max) for X, Y, Z. Scale, Color, and SH bounds are packed into 32-bit integers using 16-bit float conversion (`math.f32tof16`) and bitwise OR/shift operations [S1, unofficial].
6. Normalize the splat data in place for the current chunk. Position, scale, DC0 color, and opacity are normalized using the formula: `(value - min) / (max - min)` [S1, unofficial].
7. Normalize SH coefficients (sh1 through shF) using the same min/max bounds computed for SHs: `(sh - chunkMinshs) / (chunkMaxshs - chunkMinshs)` [S1, unofficial].

**Parameters:**
- `kChunkSize`: The number of splats per chunk. The specific value is not defined in the sources, but it is used to calculate `splatBegin` and `splatEnd` indices [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import struct

class ChunkInfo:
    def __init__(self):
        self.posX = (0.0, 0.0)
        self.posY = (0.0, 0.0)
        self.posZ = (0.0, 0.0)
        self.sclX = 0
        self.sclY = 0
        self.sclZ = 0
        self.colR = 0
        self.colG = 0
        self.colB = 0
        self.colA = 0
        self.shR = 0
        self.shG = 0
        self.shB = 0

def f32_to_f16_bits(val):
    # Standard IEEE 754 float32 to float16 bit conversion [G]
    return struct.unpack('<H', struct.pack('<f', val))[0]

def pack_f16_pair(min_val, max_val):
    # Pack two f16 values into a uint32 using bitwise OR and shifts [G]
    min_bits = f32_to_f16_bits(min_val)
    max_bits = f32_to_f16_bits(max_val)
    return (min_bits << 16) | max_bits

def square_centered_01(opacity):
    # Clamp and normalize opacity to [0, 1] range [G]
    return max(0.0, min(1.0, opacity))

def compute_chunk_bounds_and_normalize(splat_data, chunk_size):
    chunks = []
    num_splats = len(splat_data)

    for chunk_idx in range(0, num_splats, chunk_size):
        splat_begin = chunk_idx
        splat_end = min(chunk_idx + chunk_size, num_splats)

        # 1. Initialize min/max accumulators
        pos_min = [float('inf')] * 3
        pos_max = [float('-inf')] * 3
        scl_min = [float('inf')] * 3
        scl_max = [float('-inf')] * 3
        col_min = [float('inf')] * 4  # RGB + Opacity
        col_max = [float('-inf')] * 4
        sh_min = [float('inf')] * 3   # Accumulating sh1 through shF
        sh_max = [float('-inf')] * 3

        # 3. Iterate through splats
        for i in range(splat_begin, splat_end):
            s = splat_data[i]

            # Apply power transformation to scale
            s.scale = [math.pow(val, 1.0 / 8.0) for val in s.scale]

            # Apply square centered 0-1 normalization to opacity
            s.opacity = square_centered_01(s.opacity)

            # Update position bounds
            for axis in range(3):
                pos_min[axis] = min(pos_min[axis], s.position[axis])
                pos_max[axis] = max(pos_max[axis], s.position[axis])

            # Update scale bounds
            for axis in range(3):
                scl_min[axis] = min(scl_min[axis], s.scale[axis])
                scl_max[axis] = max(scl_max[axis], s.scale[axis])

            # Update color bounds (RGB + Opacity)
            col_min[0] = min(col_min[0], s.color[0])
            col_max[0] = max(col_max[0], s.color[0])
            col_min[1] = min(col_min[1], s.color[1])
            col_max[1] = max(col_max[1], s.color[1])
            col_min[2] = min(col_min[2], s.color[2])
            col_max[2] = max(col_max[2], s.color[2])
            col_min[3] = min(col_min[3], s.opacity)
            col_max[3] = max(col_max[3], s.opacity)

            # Update SH bounds individually
            for axis in range(3):
                sh_min[axis] = min(sh_min[axis], s.sh[axis])
                sh_max[axis] = max(sh_max[axis], s.sh[axis])

        # 4. Enforce minimum spread
        min_spread = 1.0e-5
        for axis in range(3):
            pos_max[axis] = max(pos_max[axis], pos_min[axis] + min_spread)
            scl_max[axis] = max(scl_max[axis], scl_min[axis] + min_spread)
            col_max[axis] = max(col_max[axis], col_min[axis] + min_spread)
            sh_max[axis] = max(sh_max[axis], sh_min[axis] + min_spread)
        col_max[3] = max(col_max[3], col_min[3] + min_spread)

        # 5. Store bounds in ChunkInfo
        chunk_info = ChunkInfo()
        chunk_info.posX = (pos_min[0], pos_max[0])
        chunk_info.posY = (pos_min[1], pos_max[1])
        chunk_info.posZ = (pos_min[2], pos_max[2])
        chunk_info.sclX = pack_f16_pair(scl_min[0], scl_max[0])
        chunk_info.sclY = pack_f16_pair(scl_min[1], scl_max[1])
        chunk_info.sclZ = pack_f16_pair(scl_min[2], scl_max[2])
        chunk_info.colR = pack_f16_pair(col_min[0], col_max[0])
        chunk_info.colG = pack_f16_pair(col_min[1], col_max[1])
        chunk_info.colB = pack_f16_pair(col_min[2], col_max[2])
        chunk_info.colA = pack_f16_pair(col_min[3], col_max[3])
        chunk_info.shR = pack_f16_pair(sh_min[0], sh_max[0])
        chunk_info.shG = pack_f16_pair(sh_min[1], sh_max[1])
        chunk_info.shB = pack_f16_pair(sh_min[2], sh_max[2])
        chunks.append(chunk_info)

        # 6 & 7. Normalize splat data in place
        for i in range(splat_begin, splat_end):
            s = splat_data[i]
            for axis in range(3):
                s.position[axis] = (s.position[axis] - pos_min[axis]) / (pos_max[axis] - pos_min[axis])
                s.scale[axis] = (s.scale[axis] - scl_min[axis]) / (scl_max[axis] - scl_min[axis])
                s.sh[axis] = (s.sh[axis] - sh_min[axis]) / (sh_max[axis] - sh_min[axis])
            s.color[0] = (s.color[0] - col_min[0]) / (col_max[0] - col_min[0])
            s.color[1] = (s.color[1] - col_min[1]) / (col_max[1] - col_min[1])
            s.color[2] = (s.color[2] - col_min[2]) / (col_max[2] - col_min[2])
            s.opacity = (s.opacity - col_min[3]) / (col_max[3] - col_min[3])

    return chunks
```

**Edge Cases and Pitfalls:**
- **Division by Zero:** Enforcing a minimum spread of `1.0e-5f` prevents division by zero when min and max bounds are equal [S1, unofficial].
- **Precision Loss:** Using 16-bit floats for packing bounds may introduce precision loss, especially for large ranges [S1, unofficial].
- **Degenerate Input:** If a chunk contains only one splat, the min and max bounds will be identical, requiring the minimum spread enforcement [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Not given in the sources.
- **Strengths:** Not given in the sources.
- **Limitations:** Not given in the sources.
- **When to Use:** Not given in the sources.
- **When Not to Use:** Not given in the sources.

## Background
Not given in the sources.

## Sources
[S1] nigelhartman/worldlabs_unity, https://github.com/nigelhartman/worldlabs_unity, fetched 2026-10-27 (unofficial)
