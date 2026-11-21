# 3D Texture Representation (CPU/GPU Indexing)

Also called: Volume Header, Voxel Data Structure and Indexing, Voxel Grid Volumes

## Overview
3D Texture Representation stores voxel data in a dense 3D texture or equivalent linear array where the texture dimensions match the voxel grid dimensions exactly [S1, unofficial]. It solves the problem of mapping continuous 3D coordinates to discrete voxel values by using simple linear indexing, leveraging hardware acceleration for caching and filtering [S1, unofficial]. Unlike tree-based structures like Octrees, it offers high runtime performance and simplicity but suffers from low memory efficiency due to the allocation of empty space in sparse volumes [S1, unofficial].

## How it works
**Inputs:**
- Grid dimensions: $Size = (S_x, S_y, S_z)$ [S3].
- Voxel coordinates: $(X, Y, Z)$ for CPU access [S3] or normalized texture coordinates $(u, v, w) \in [0, 1]^3$ for GPU access [S3].
- Voxel data: Values stored in a contiguous 1D array or 3D texture [S1, unofficial][S3].

**Outputs:**
- Linear index $I$ for 1D array access [S3].
- Interpolated voxel value $V$ [G].

**Data Structures:**
1.  **CPU Side:**
    -   `Int3`: Struct with fields `X`, `Y`, `Z` of type `int` [S3].
    -   `VolumeHeader`: Contains `Size` (`Int3`) and methods `CubicToLinearIndex` and `LinearToCubicIndex` [S3].
    -   `VolumeBuffer<T>`: Contains `Header` (`VolumeHeader`) and `DataArray` (`T[]`) [S3].
2.  **GPU Side:**
    -   3D Texture (`GL_TEXTURE_3D`) with format `GL_R8` and dimensions `dimension x dimension x dimension` [S2, unofficial].
    -   Uniform buffer containing `_VolBufferSize` (`float3`) [S3].

**Numbered Steps:**

1.  **Coordinate Conversion (GPU):** Convert normalized texture coordinates to integer voxel indices. $\mathbf{intIndex3} = \lfloor \mathbf{coord} \times \mathbf{VolBufferSize} \rfloor$ [S3].
2.  **Linear Indexing (CPU/GPU):** Map 3D integer coordinates $(X, Y, Z)$ to a 1D linear index $I$ using row-major ordering. $I = X + (Y \times S_x) + (Z \times S_x \times S_y)$ [S3].
3.  **Inverse Mapping (CPU):** Convert linear index $I$ back to cubic coordinates $(X, Y, Z)$. $X = (I \mod S_x)$, $Y = ((I / S_x) \mod S_y)$, $Z = ((I / (S_x \times S_y)) \mod S_z)$ [S3].
4.  **Data Access:** CPU uses `VolumeBuffer.GetVoxel(pos)` which internally calls `Header.CubicToLinearIndex(pos)` to access `DataArray` [S3]. GPU samples the 3D texture or accesses the structured buffer using the calculated linear index [S2, unofficial][S3]. For `GL_R8` textures, density is retrieved as `(float)pTexels[index].a / 128.0f` [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

class Int3:
    def __init__(self, x: int, y: int, z: int):
        self.x = x
        self.y = y
        self.z = z

class VolumeHeader:
    def __init__(self, size: Int3):
        self.size = size

    def cubic_to_linear_index(self, index: Int3) -> int:
        # Formula: X + (Y * Size.X) + (Z * Size.X * Size.Y)
        return index.x + (index.y * self.size.x) + (index.z * (self.size.x * self.size.y))

    def linear_to_cubic_index(self, linear_index: int) -> Int3:
        # Formula: X = (I % Size.X), Y = ((I / Size.X) % Size.Y), Z = ((I / (Size.X * Size.Y)) % Size.Z)
        x = linear_index % self.size.x
        y = (linear_index // self.size.x) % self.size.y
        z = (linear_index // (self.size.x * self.size.y)) % self.size.z
        return Int3(x, y, z)

class VolumeBuffer:
    def __init__(self, header: VolumeHeader, data_array):
        self.header = header
        self.data_array = data_array

    def get_voxel(self, pos: Int3):
        index = self.header.cubic_to_linear_index(pos)
        return self.data_array[index]

    def set_voxel(self, pos: Int3, val):
        index = self.header.cubic_to_linear_index(pos)
        self.data_array[index] = val

def sample_vol_gpu(coord3: tuple, vol_buffer_size: tuple, vol_buffer: list) -> float:
    # 1. Convert normalized coordinates to integer indices (floor)
    int_index3 = (
        math.floor(coord3[0] * vol_buffer_size[0]),
        math.floor(coord3[1] * vol_buffer_size[1]),
        math.floor(coord3[2] * vol_buffer_size[2])
    )
    
    # 2. Calculate linear index
    size_x, size_y, size_z = vol_buffer_size
    linear_index = int_index3[0] + (int_index3[1] * size_x) + (int_index3[2] * (size_x * size_y))
    
    # 3. Return value
    return vol_buffer[linear_index]
```

**Parameters:**
-   **Texture Format:** `GL_R8` is common for density [S2, unofficial].
-   **Filtering:** `GL_LINEAR` for magnification, `GL_LINEAR_MIPMAP_LINEAR` for minification [S2, unofficial].
-   **Mipmaps:** Can be generated [S2, unofficial].

**Edge Cases and Pitfalls:**
-   **Sparse Memory:** The structure allocates memory for the entire bounding box, even if most voxels are empty [S1, unofficial]. This limits achievable image quality on memory-limited platforms [S1, unofficial].
-   **Texture Compression:** While BC1-7 compression can reduce memory usage, it adds a runtime performance cost to all pipeline stages, not just voxelization [S1, unofficial].

## Subtopics

### 3D Texture Population for Gradient and Intensity
Encodes voxel gradients and normalized intensities into a 3D RGBA texture for GPU raycasting [S4, unofficial]. The process iterates over all voxels $(k, j, i)$, calculates the linear index $I = i + j \times res.x + k \times res.x \times res.y$, and normalizes gradient components $(x, y, z)$ by `maxGra` (255 for 8-bit, 65535 for 16-bit) before storing them in the RGB channels [S4, unofficial]. Intensity is stored in the A channel using either linear normalization $xpos = ((I - minInt) \times 255.0) / (maxInt - minInt)$ or histogram equalization $xpos = (accCounter[I] \times 255.0) / sum$, followed by $A = (xpos + 1.0) / 257.0$ [S4, unofficial]. The resulting `tex3D` is a contiguous float array of size $res.x \times res.y \times res.z \times 4$ with [R, G, B, A] layout per voxel [S4, unofficial].

## Operating characteristics
-   **Time Complexity:** $O(1)$ for indexing and access [S1, unofficial].
-   **Memory Complexity:** $O(S_x \times S_y \times S_z)$, regardless of sparsity [S1, unofficial].
-   **Performance:** High runtime performance due to hardware acceleration (caching, filtering) [S1, unofficial].
-   **Limitations:** Low memory efficiency for sparse volumes [S1, unofficial]. Texture compression mitigates memory issues but incurs a performance penalty across the entire pipeline [S1, unofficial].
-   **When to Use:** When the volume is dense, memory is sufficient, and maximum rendering speed is required [S1, unofficial].
-   **When Not to Use:** When the volume is sparse or memory is constrained [S1, unofficial].

## Background
This approach was taken in the implementation of *Real-Time Ray Traced Voxel Global Illumination* due to its runtime performance and simplicity, despite not optimizing memory usage [S1, unofficial].

## Sources
[S1] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
[S2] jaafersheriff/Cloud-Renderer, https://github.com/jaafersheriff/Cloud-Renderer, fetched 2026-10-27 (unofficial)
[S3] Volume rendering - Mixed Reality | Microsoft Learn, https://learn.microsoft.com/en-us/windows/mixed-reality/develop/native/volume-rendering, fetched 2026-10-27
[S4] extrovert7986/Scientific-visualization-Slicing-volume-rendering, https://github.com/extrovert7986/Scientific-visualization-Slicing-volume-rendering, fetched 2026-10-26 (unofficial)
