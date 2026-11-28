# Gaussian Splat Data Packing and Compression

Also called: Gaussian Splat Encoding, Runtime Splat Processing, SPZ Processing

## Overview
This method converts raw `InputSplatData` arrays (from SPZ or PLY files) into compressed GPU-ready byte buffers stored in `RuntimeSplatData` [S1, unofficial]. All jobs are Burst-compiled and have zero Editor dependencies, safe to call at runtime in builds [S1, unofficial]. It sets itself apart from sibling methods by integrating Morton space-filling curve reordering for cache coherence with aggressive per-attribute quantization and chunk-based storage for spherical harmonics [S1, unofficial].

## How it works
**Inputs:**
- Raw splat data array (`NativeArray<InputSplatData>`), containing position, scale, rotation, color, and spherical harmonic (SH) coefficients [S1, unofficial].
- Format specifications for each attribute: `posFormat`, `scaleFormat`, `colorFormat`, `shFormat` [S1, unofficial].

**Outputs:**
- Compressed GPU-ready byte buffers stored in `RuntimeSplatData` [S1, unofficial].
- Reordered splat indices based on Morton codes [S1, unofficial].

**Data Structures:**
1. **InputSplatData**: The raw input structure [S1, unofficial].
2. **GaussianSplatAsset.ChunkInfo**: Used when SH data is not stored in full float32 [S1, unofficial].
3. **ReorderMortonJob**: A job struct for parallel Morton reordering [S1, unofficial].
    - `const float kScaler = (1<<21)-1`
    - `float3 m_BoundsMin`
    - `float3 m_InvBoundsSize`
    - `ReadOnly NativeArray<InputSplatData> m_SplatData`
    - `NativeArray<(ulong, int)> m_Order` (stores Morton code and original index)
4. **CalcChunkDataJob**: A job struct for computing SH chunks [S1, unofficial].
    - `NativeArray<InputSplatData> splatData`
    - `NativeArray<GaussianSplatAsset.ChunkInfo> chunks`

**Parameters (Defaults):**
- `posFormat`: `GaussianSplatAsset.VectorFormat.Norm11` [S1, unofficial]
- `scaleFormat`: `GaussianSplatAsset.VectorFormat.Norm11` [S1, unofficial]
- `colorFormat`: `GaussianSplatAsset.ColorFormat.Norm8x4` [S1, unofficial]
- `shFormat`: `GaussianSplatAsset.SHFormat.Norm6` [S1, unofficial]

**Steps:**

1. **Input Validation**: Check if the input splat array is empty. If empty, throw `ArgumentException` [S1, unofficial].
2. **Scene Bounds Calculation**: Iterate through all splats to find the axis-aligned bounding box (AABB) [S1, unofficial].
    - Initialize `bMin` to positive infinity and `bMax` to negative infinity [S1, unofficial].
    - For each splat position `p`: `bMin = min(bMin, p)` and `bMax = max(bMax, p)` [S1, unofficial].
3. **Morton Reordering**:
    - Calculate inverse bounds size: `invSize = 1.0f / (bMax - bMin)` [S1, unofficial].
    - Normalize position to `[0, 1]`: `pos = ((input_pos - boundsMin) * invSize) * kScaler` [S1, unofficial].
    - Scale to integer range for Morton encoding: `ipos = cast_to_uint3(pos)` [S1, unofficial].
    - Generate Morton code: `code = MortonEncode3(ipos)` [S1, unofficial].
    - Sort splats by `code` and reorder the `InputSplatData` array accordingly [S1, unofficial].
4. **Chunk Data Creation (Conditional)**:
    - If any format is not full float32 (`posFormat != Float32` OR `scaleFormat != Float32` OR `colorFormat != Float32x4` OR `shFormat != Float32`), call `CreateChunkData` [S1, unofficial].
    - Ensure chunk bounds are clamped to `splatData.Length` to avoid out-of-bounds access [S1, unofficial].
5. **Position Encoding**: Call `CreatePositionsData` with the reordered splats and `posFormat` [S1, unofficial].
    - If `posFormat` is `Norm11`: `enc = (v.x * 2047.5f) | (v.y * 1023.5f << 11) | (v.z * 2047.5f << 21)`, stored as 4 bytes (uint) [S1, unofficial].
    - If `posFormat` is `Norm16`: `enc = (v.x * 65535.5f) | (v.y * 65535.5f << 16) | (v.z * 65535.5f << 32)`, stored as 4 bytes (uint) + 2 bytes (ushort) [S1, unofficial].
6. **Scale and SH Index Encoding**: Call `CreateOtherData` with splats, `scaleFormat`, and `splatSHIndices` [S1, unofficial].
7. **Color Encoding**: Call `CreateColorData` with splats [S1, unofficial].
8. **SH Data Encoding**: Call `CreateSHData` with splats, `shFormat`, and `clusteredSHs` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
from typing import List, Tuple, Any

class float3:
    def __init__(self, x: float, y: float, z: float):
        self.x = x
        self.y = y
        self.z = z
    def __sub__(self, other): return float3(self.x - other.x, self.y - other.y, self.z - other.z)
    def __mul__(self, scalar): return float3(self.x * scalar, self.y * scalar, self.z * scalar)
    def min(self, other): return float3(min(self.x, other.x), min(self.y, other.y), min(self.z, other.z))
    def max(self, other): return float3(max(self.x, other.x), max(self.y, other.y), max(self.z, other.z))

class uint3:
    def __init__(self, x: int, y: int, z: int):
        self.x = x
        self.y = y
        self.z = z

class Splat:
    def __init__(self, position: float3, scale: float3, color: Tuple[float, float, float, float], sh_index: int, sh_coefficients: List[float]):
        self.position = position
        self.scale = scale
        self.color = color
        self.sh_index = sh_index
        self.sh_coefficients = sh_coefficients

class RuntimeSplatData:
    def __init__(self, pos_buffer: bytes, scale_buffer: bytes, sh_indices_buffer: bytes, color_buffer: bytes, sh_buffer: bytes, chunks: List[Any]):
        self.pos_buffer = pos_buffer
        self.scale_buffer = scale_buffer
        self.sh_indices_buffer = sh_indices_buffer
        self.color_buffer = color_buffer
        self.sh_buffer = sh_buffer
        self.chunks = chunks

class ArgumentException(Exception):
    pass

def morton_encode_3(v: uint3) -> int:
    # Standard 3D Morton code interleaving (21 bits per axis max)
    x, y, z = v.x, v.y, v.z
    x = (x | (x << 16)) & 0x0000003F0000FF
    x = (x | (x << 8)) & 0x000300000F0000F0
    x = (x | (x << 4)) & 0x03000000F000F000
    x = (x | (x << 2)) & 0x0000000000000003
    y = (y | (y << 16)) & 0x0000003F0000FF
    y = (y | (y << 8)) & 0x000300000F0000F0
    y = (y | (y << 4)) & 0x03000000F000F000
    y = (y | (y << 2)) & 0x0000000000000003
    z = (z | (z << 16)) & 0x0000003F0000FF
    z = (z | (z << 8)) & 0x000300000F0000F0
    z = (z | (z << 4)) & 0x03000000F000F000
    z = (z | (z << 2)) & 0x0000000000000003
    return (x << 0) | (y << 1) | (z << 2)

def create_chunk_data(splat_data: List[Splat]) -> List[Any]:
    # Not given in the sources. Placeholder logic for chunking.
    # In practice, this groups splats into chunks for SH storage.
    return []

def pack_gaussian_splats(splat_data: List[Splat], pos_format: str = "Norm11", scale_format: str = "Norm11", color_format: str = "Norm8x4", sh_format: str = "Norm6") -> RuntimeSplatData:
    if not splat_data:
        raise ArgumentException("Input splat array is empty")

    b_min = float3(float('inf'), float('inf'), float('inf'))
    b_max = float3(float('-inf'), float('-inf'), float('-inf'))
    for splat in splat_data:
        b_min = b_min.min(splat.position)
        b_max = b_max.max(splat.position)

    inv_bounds_size = float3(1.0 / (b_max.x - b_min.x), 1.0 / (b_max.y - b_min.y), 1.0 / (b_max.z - b_min.z))
    k_scaler = (1 << 21) - 1

    indexed_splats = []
    for i, splat in enumerate(splat_data):
        pos_norm = (splat.position - b_min) * inv_bounds_size
        ipos = uint3(int(pos_norm.x * k_scaler), int(pos_norm.y * k_scaler), int(pos_norm.z * k_scaler))
        morton_code = morton_encode_3(ipos)
        indexed_splats.append((morton_code, i))

    indexed_splats.sort(key=lambda x: x[0])
    reordered_splat_data = [splat_data[idx] for _, idx in indexed_splats]

    needs_chunks = (pos_format != "Float32" or scale_format != "Float32" or 
                    color_format != "Float32x4" or sh_format != "Float32")
    chunks = []
    if needs_chunks:
        chunks = create_chunk_data(reordered_splat_data)
        for chunk in chunks:
            chunk.end = min(chunk.end, len(reordered_splat_data))

    pos_buffer = bytearray()
    for splat in reordered_splat_data:
        if pos_format == "Norm11":
            enc = int((splat.position.x * 2047.5) | (splat.position.y * 1023.5 << 11) | (splat.position.z * 2047.5 << 21))
            pos_buffer += enc.to_bytes(4, byteorder='little')
        elif pos_format == "Norm16":
            enc = int((splat.position.x * 65535.5) | (splat.position.y * 65535.5 << 16) | (splat.position.z * 65535.5 << 32))
            pos_buffer += (enc & 0xFFFFFFFF).to_bytes(4, byteorder='little')
            pos_buffer += ((enc >> 32) & 0xFFFF).to_bytes(2, byteorder='little')
        else:
            pass

    scale_buffer = bytearray()
    sh_indices_buffer = bytearray()
    for splat in reordered_splat_data:
        if scale_format == "Norm11":
            enc = int((splat.scale.x * 2047.5) | (splat.scale.y * 1023.5 << 11) | (splat.scale.z * 2047.5 << 21))
            scale_buffer += enc.to_bytes(4, byteorder='little')
        sh_indices_buffer += splat.sh_index.to_bytes(4, byteorder='little')

    color_buffer = bytearray()
    for splat in reordered_splat_data:
        if color_format == "Norm8x4":
            r = int(splat.color[0] * 255)
            g = int(splat.color[1] * 255)
            b = int(splat.color[2] * 255)
            a = int(splat.color[3] * 255)
            color_buffer += bytes([r, g, b, a])

    sh_buffer = bytearray()
    for splat in reordered_splat_data:
        if sh_format == "Norm6":
            for coeff in splat.sh_coefficients:
                q = int(coeff * 31.5)
                sh_buffer += q.to_bytes(1, byteorder='little')

    return RuntimeSplatData(pos_buffer, scale_buffer, sh_indices_buffer, color_buffer, sh_buffer, chunks)
```

## Operating characteristics
- **Time Complexity**: Dominated by the Morton sort, which is $O(N \log N)$ where $N$ is the number of splats [S1, unofficial]. Bounds calculation is $O(N)$ [S1, unofficial]. Encoding is $O(N)$ [S1, unofficial].
- **Memory Complexity**: Significantly reduced compared to raw float32 storage [S1, unofficial]. Position and scale drop from 12 bytes each to 4 bytes (Norm11) or 6 bytes (Norm16) [S1, unofficial]. Color drops from 16 bytes to 4 bytes (Norm8x4) [S1, unofficial]. SH data is heavily compressed via quantization and chunking [S1, unofficial].
- **Performance**: The method is designed for runtime execution using Burst compilation, ensuring zero Editor dependencies and safe usage in builds [S1, unofficial].
- **Strengths**: High compression ratio, cache-friendly ordering via Morton codes, runtime-safe [S1, unofficial].
- **Limitations**: Quantization introduces precision loss [S1, unofficial]. Morton reordering requires sorting, which can be costly for very large datasets if not parallelized effectively [S1, unofficial]. Empty input arrays cause exceptions [S1, unofficial].

## Background
This method is part of the WorldLabs Unity integration package, specifically designed for processing SPZ (Splat Zone) files generated by WorldLabs' AI world generation API [S1, unofficial]. It enables efficient runtime rendering of Gaussian Splat scenes in Unity, particularly for VR and mobile platforms [S1, unofficial]. The approach reflects the broader trend in 3D Gaussian Splatting towards optimizing data layouts for real-time rendering engines [S1, unofficial].

## Sources
[S1] nigelhartman/worldlabs_unity, https://github.com/nigelhartman/worldlabs_unity, fetched 2026-10-27 (unofficial)
