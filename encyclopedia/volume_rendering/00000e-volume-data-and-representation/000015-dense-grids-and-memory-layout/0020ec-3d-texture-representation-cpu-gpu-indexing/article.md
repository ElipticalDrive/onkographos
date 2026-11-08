# 3D Texture Representation (CPU/GPU Indexing)

Also called: 3D Textures, Volume Header, Voxel Data Structure and Indexing, Voxel Grid Volumes

## Overview
3D Texture Representation stores voxel data in a dense 3D texture (or equivalent linear array) with dimensions matching the voxel grid, providing the simplest storage mechanism for volume rendering [S1, unofficial]. Unlike tree-based structures like octrees, it fully leverages hardware acceleration for 3D textures, including caching and filtering, offering high runtime performance but low memory efficiency due to its inability to address sparse memory [S1, unofficial].

## How it works
The method takes 3D coordinates (cubic or normalized) and volume dimensions ($Size.X, Size.Y, Size.Z$) as inputs, and produces a linear array index and the corresponding voxel value (e.g., density or normal) as outputs [S3][S2, unofficial]. 

**Data Structures:**
*   **CPU (C#-like):**
    *   `Int3`: Struct with fields `X`, `Y`, `Z` of type `int` [S3].
    *   `VolumeHeader`: Contains `Size` (`Int3`) and methods `CubicToLinearIndex` and `LinearToCubicIndex` [S3].
    *   `VolumeBuffer<T>`: Contains `Header` (`VolumeHeader`) and `DataArray` (`T[]`) [S3].
*   **GPU (HLSL-like):**
    *   `_VolBufferSize`: `float3` representing volume dimensions [S3].
    *   `_VolBuffer`: `StructuredBuffer<float>` storing voxel data [S3].
*   **OpenGL (Contextual):**
    *   Format: `GL_R8` [S2, unofficial].
    *   Dimensions: `dimension x dimension x dimension` [S2, unofficial].
    *   Filters: `GL_LINEAR_MIPMAP_LINEAR` (min), `GL_LINEAR` (mag) [S2, unofficial].

**Indexing Formulas:**
The linear index $I$ from cubic coordinates $(X, Y, Z)$ is calculated as [S3]:
$$I = X + (Y \times Size.X) + (Z \times Size.X \times Size.Y)$$
A cube-specific variant uses a single dimension $dim$: $index = x + y \times dim + z \times dim \times dim$ [S2, unofficial].

The cubic coordinates from a linear index $I$ are [S3]:
$$X = (I \div 1) \bmod Size.X$$
$$Y = (I \div Size.X) \bmod Size.Y$$
$$Z = (I \div (Size.X \times Size.Y)) \bmod Size.Z$$
where $\div$ denotes integer division.

**Steps:**
1.  **Coordinate Conversion:** Convert input coordinates to integer voxel indices [S3][S2, unofficial]. On CPU, use `VolumeHeader.CubicToLinearIndex` [S3]. On GPU, convert normalized coordinates $coord$ to integer indices: $intIndex3 = \lfloor coord \times _VolBufferSize \rfloor$ [S3]. Handle negative coordinates by wrapping: if $x < 0$, $x = (x + dim) \bmod dim$ [S2, unofficial].
2.  **Linear Index Calculation:** Compute the 1D index using the forward formula [S3].
3.  **Data Access:** Read or write `DataArray[index]` (CPU) or `_VolBuffer[index]` (GPU) [S3]. Density can be extracted from alpha channels, e.g., `density = (float)pTexels[index].a / 128.0f` [S2, unofficial]. Normals can be stored in RGB channels, e.g., `pTexels[index].r = (char)(normal.r * 128.0f)` [S2, unofficial]. During voxelization, calculate the voxel cell index and write directly into it without extra steps or tree traversals [S1, unofficial].

**Pseudocode (CPU):**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Int3:
    def __init__(self, x, y, z):
        self.X = x
        self.Y = y
        self.Z = z

class VolumeHeader:
    def __init__(self, size: Int3):
        self.Size = size

    def CubicToLinearIndex(self, index: Int3) -> int:
        return index.X + (index.Y * self.Size.X) + (index.Z * (self.Size.X * self.Size.Y))

    def LinearToCubicIndex(self, linearIndex: int) -> Int3:
        x = linearIndex % self.Size.X
        y = (linearIndex // self.Size.X) % self.Size.Y
        z = (linearIndex // (self.Size.X * self.Size.Y)) % self.Size.Z
        return Int3(x, y, z)

class VolumeBuffer:
    def __init__(self, header: VolumeHeader, data_array):
        self.Header = header
        self.DataArray = data_array

    def GetVoxel(self, pos: Int3):
        return self.DataArray[self.Header.CubicToLinearIndex(pos)]

    def SetVoxel(self, pos: Int3, val):
        self.DataArray[self.Header.CubicToLinearIndex(pos)] = val
```

**Pseudocode (GPU):**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
VolBufferSize = (width, height, depth)
VolBuffer = [...]  # StructuredBuffer<float>

def unit_volume_to_int_volume(coord):
    return (int(coord[0] * VolBufferSize[0]),
            int(coord[1] * VolBufferSize[1]),
            int(coord[2] * VolBufferSize[2]))

def int_volume_to_linear_index(coord, size):
    return coord[0] + (coord[1] * size[0]) + (coord[2] * (size[0] * size[1]))

def sample_vol(coord3):
    int_index3 = unit_volume_to_int_volume(coord3)
    index1d = int_volume_to_linear_index(int_index3, VolBufferSize)
    return VolBuffer[index1d]
```

## Operating characteristics
*   **Time Complexity:** $O(1)$ for index calculation and data access [S1, unofficial].
*   **Memory Complexity:** $O(N)$ where $N$ is the total number of voxels in the grid, regardless of sparsity [S1, unofficial].
*   **Performance:** High runtime performance due to hardware acceleration (caching, filtering) [S1, unofficial].
*   **Limitations:** Low memory efficiency; significant portion of the grid may be empty but still allocated, limiting achievable image quality on memory-limited platforms [S1, unofficial].
*   **Trade-offs:** Texture compression (BC1-7) could reduce sparse memory impact but adds a runtime performance cost to all pipeline stages, not just voxelization [S1, unofficial].

## Background
This approach was taken in the implementation of "Real-Time Ray Traced Voxel Global Illumination" due to its runtime performance and simplicity, despite not optimizing memory usage [S1, unofficial]. It is also documented in Microsoft's Mixed Reality volume rendering guides [S3] and open-source projects like Cloud-Renderer [S2, unofficial].

## Sources
[S1] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
[S2] jaafersheriff/Cloud-Renderer, https://github.com/jaafersheriff/Cloud-Renderer, fetched 2026-10-27 (unofficial)
[S3] Volume rendering - Mixed Reality | Microsoft Learn, https://learn.microsoft.com/en-us/windows/mixed-reality/develop/native/volume-rendering, fetched 2026-10-27
