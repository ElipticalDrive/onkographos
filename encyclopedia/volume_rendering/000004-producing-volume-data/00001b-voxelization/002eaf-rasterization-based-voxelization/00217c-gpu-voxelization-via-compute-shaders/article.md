# GPU Voxelization via Compute Shaders

Also called: AHD GPU Voxelizer, AfterHumanDeclined, AfterHumanDeclined (AHD) GPU Voxelizer, GPU Boxelizer, GPUVoxelizer, VoxelizerX12

## Overview
GPU Voxelization via Compute Shaders converts 3D triangle meshes into a uniform 3D voxel grid by leveraging GPU parallel processing [S1, unofficial][S3]. It solves the problem of generating volumetric data for real-time ray marching or volume rendering with high throughput [S1, unofficial]. Unlike [Rasterization-based Voxelization](908-rasterization-based-voxelization.md), which projects triangles via the fixed-function graphics pipeline, compute shader methods decouple intersection logic from rasterization, allowing custom geometric tests and atomic operations on 3D buffers or textures [S1, unofficial][S3].

## How it works
The implementation varies between two primary architectural patterns found in the sources: a Direct3D 11/12 rasterization-assisted approach (AHD/VoxelizerX12) and a pure compute shader geometric approach (GPU Boxelizer) [S1, unofficial][S3]. Both share common preprocessing steps but diverge in the core voxelization kernel [S1, unofficial][S3].

### Inputs and Outputs
**Inputs:**
*   Triangle mesh data: Vertex positions (3 floats), optional normals, indices (16 or 32-bit integers) [S1, unofficial][S3].
*   Grid parameters: Resolution ($N_x, N_y, N_z$), bounding box bounds ($\mathbf{Start}, \mathbf{End}$), and voxel unit size ($u$) [S3].
*   Optional: Texture maps for color/normal mapping [S1, unofficial].

**Outputs:**
*   A 3D grid of voxels. Depending on the variant, this is stored as:
    *   A 1D `ComputeBuffer` or `RWTexture3D` of `Voxel_t` structures [S3].
    *   A UAV buffer of `Voxel` structs with an initial count pass [S1, unofficial].
    *   Multiple `Texture3D` grids if mutexing is required [S2, unofficial].

### Data Structures

**1. Voxel Resource (AHD/D3D11)**
*   `mVertexBuffer`: `ID3D11Buffer` containing vertex data [S1, unofficial].
*   `mIndexBuffer`: `ID3D11Buffer` containing index data [S1, unofficial].
*   `mAABB`: Axis-Aligned Bounding Box of the mesh [S1, unofficial].
*   `mVertexStride`: Byte size of a vertex [S1, unofficial].
*   `mIndexStride`: 2 or 4 bytes [S1, unofficial].

**2. Voxel_t Structure (GPU Boxelizer)**
*   Stored in a 1D `ComputeBuffer` of size $W \times H \times D$ [S3].
*   Fields: Occupancy flag, surface attributes (color, normal) [S3].
*   Index calculation: $idx = x + y \cdot W + z \cdot W \cdot H$ [S3].

**3. Constant Buffers**
*   **AHD `EffectParameter`**: `bcount` (bool), `world` (matrix), `width/height/depth` (float), `views` (array of 4 matrices) [S1, unofficial].
*   **GPU Boxelizer Uniforms**: `_Start`, `_End`, `_Size` (Vector3); `_Unit`, `_InvUnit`, `_HalfUnit` (float); `_Width`, `_Height`, `_Depth` (int) [S3].

### Preprocessing (Common)
1.  **Bounding Box Calculation**: Compute the AABB of the input mesh. Determine the center $\mathbf{c}$ and max dimension $L_{max}$ [S1, unofficial][S3].
2.  **Grid Dimensions**:
    *   Calculate voxel unit size: $u = L_{max} / \text{resolution}$ [S3].
    *   Expand bounds by one unit to ensure coverage: $\mathbf{Start} = \mathbf{min} - u$, $\mathbf{End} = \mathbf{max} + u$ [S3].
    *   Calculate grid dimensions: $W = \lceil (\mathbf{End}_x - \mathbf{Start}_x) / u \rceil$, similarly for $H, D$ [S3].
3.  **Buffer Allocation**: Create GPU buffers for vertices, indices, and the voxel grid [S1, unofficial][S3].

### Variant 1: Two-Pass Rasterization-Assisted (AHD / VoxelizerX12)
This method uses the graphics pipeline to count voxels and then writes them, often using depth peeling for solid voxelization [S1, unofficial][S2, unofficial].

1.  **Pass 1: Count**:
    *   Render the scene to a UAV counter buffer with `bcount=true` [S1, unofficial].
    *   Use 4 orthographic views along principal axes (X, Y, Z) to project triangles [S1, unofficial].
    *   Read back the total voxel count `numVoxels` to CPU [S1, unofficial].
2.  **Allocation**:
    *   Allocate a UAV buffer of size `numVoxels * sizeof(Voxel)` [S1, unofficial].
3.  **Pass 2: Write**:
    *   Render the scene again with `bcount=false` to write voxel data into the allocated buffer [S1, unofficial].
    *   For solid voxelization, use depth peeling to fill interior voxels [S2, unofficial].

### Variant 2: Pure Compute Shader Geometric (GPU Boxelizer)
This method uses explicit geometric containment tests within compute kernels [S3].

1.  **Dispatch SurfaceFront Kernel**:
    *   Launch compute shader with thread group size $(\lceil \text{triangleCount} / \text{ThreadX} \rceil, 1, 1)$ [S3].
    *   Each thread processes one triangle [S3].
2.  **SurfaceFront Kernel Logic**:
    *   Retrieve triangle vertices $\mathbf{v}_0, \mathbf{v}_1, \mathbf{v}_2$ from buffers [S3].
    *   Determine front/back facing: $\mathbf{n} = \text{normalize}((\mathbf{v}_1-\mathbf{v}_0) \times (\mathbf{v}_2-\mathbf{v}_1))$. If $\mathbf{n} \cdot (0,0,1) > 0$, it is front-facing [S3].
    *   If back-facing, return [S3].
    *   Calculate triangle AABB in voxel space [S3].
    *   Iterate through all voxels in the AABB [S3].
    *   Perform plane intersection test: Check if the voxel center lies on the correct side of the triangle plane [S3].
    *   Write to voxel buffer using atomic operations if necessary [S3].
3.  **Dispatch SurfaceBack Kernel**:
    *   Launch compute shader with thread group size $(\lceil \text{triangleCount} / \text{ThreadX} \rceil, 1, 1)$ [S3].
    *   Process only back-facing triangles [S3]. This order (Front then Back) allows back-facing flags to override front-facing flags in case of collision, resolving race conditions [S3].
4.  **Dispatch Volume Kernel**:
    *   Launch with thread group size $(\lceil W/\text{ThreadX} \rceil, \lceil H/\text{ThreadY} \rceil, \lceil D/\text{ThreadZ} \rceil)$ [S3].
    *   Iterate through all lattice coordinates $(x,y,z)$ [S3].
    *   Fill interior voxels based on surface occupancy [S3].

### Pseudocode (GPU Boxelizer Variant)

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def SurfaceFront(triangle_idx, vert_buffer, tri_buffer, voxel_buffer, _Start, _InvUnit, _Unit, _Width, _Height, _Depth, _TriangleCount):
    if triangle_idx >= _TriangleCount:
        return
    i0, i1, i2 = tri_buffer[triangle_idx * 3 : triangle_idx * 3 + 3]
    va, vb, vc = vert_buffer[i0], vert_buffer[i1], vert_buffer[i2]
    normal = normalize(cross(vb - va, vc - va))
    if dot(normal, (0, 0, 1)) <= 0:
        return
    min_v = min(va, vb, vc)
    max_v = max(va, vb, vc)
    ix_min = int(floor((min_v.x - _Start.x) * _InvUnit))
    iy_min = int(floor((min_v.y - _Start.y) * _InvUnit))
    iz_min = int(floor((min_v.z - _Start.z) * _InvUnit))
    ix_max = int(floor((max_v.x - _Start.x) * _InvUnit))
    iy_max = int(floor((max_v.y - _Start.y) * _InvUnit))
    iz_max = int(floor((max_v.z - _Start.z) * _InvUnit))
    ix_min = max(0, ix_min); iy_min = max(0, iy_min); iz_min = max(0, iz_min)
    ix_max = min(_Width - 1, ix_max); iy_max = min(_Height - 1, iy_max); iz_max = min(_Depth - 1, iz_max)
    for z in range(iz_min, iz_max + 1):
        for y in range(iy_min, iy_max + 1):
            for x in range(ix_min, ix_max + 1):
                voxel_center = _Start + (x + 0.5, y + 0.5, z + 0.5) * _Unit
                dist = dot(normal, voxel_center - va)
                if dist > 0:
                    atomic_add(voxel_buffer, x + y * _Width + z * _Width * _Height, 1)

def SurfaceBack(triangle_idx, vert_buffer, tri_buffer, voxel_buffer, _Start, _InvUnit, _Unit, _Width, _Height, _Depth, _TriangleCount):
    if triangle_idx >= _TriangleCount:
        return
    i0, i1, i2 = tri_buffer[triangle_idx * 3 : triangle_idx * 3 + 3]
    va, vb, vc = vert_buffer[i0], vert_buffer[i1], vert_buffer[i2]
    normal = normalize(cross(vb - va, vc - va))
    if dot(normal, (0, 0, 1)) > 0:
        return
    min_v = min(va, vb, vc)
    max_v = max(va, vb, vc)
    ix_min = int(floor((min_v.x - _Start.x) * _InvUnit))
    iy_min = int(floor((min_v.y - _Start.y) * _InvUnit))
    iz_min = int(floor((min_v.z - _Start.z) * _InvUnit))
    ix_max = int(floor((max_v.x - _Start.x) * _InvUnit))
    iy_max = int(floor((max_v.y - _Start.y) * _InvUnit))
    iz_max = int(floor((max_v.z - _Start.z) * _InvUnit))
    ix_min = max(0, ix_min); iy_min = max(0, iy_min); iz_min = max(0, iz_min)
    ix_max = min(_Width - 1, ix_max); iy_max = min(_Height - 1, iy_max); iz_max = min(_Depth - 1, iz_max)
    for z in range(iz_min, iz_max + 1):
        for y in range(iy_min, iy_max + 1):
            for x in range(ix_min, ix_max + 1):
                voxel_center = _Start + (x + 0.5, y + 0.5, z + 0.5) * _Unit
                dist = dot(normal, voxel_center - va)
                if dist > 0:
                    atomic_add(voxel_buffer, x + y * _Width + z * _Width * _Height, 1)
```

### Parameters and Typical Values
*   **Resolution**: Typically $64^3$ to $256^3$ for real-time applications [S3].
*   **Thread Group Size**: $(8, 1, 1)$ for triangle-based dispatch [S3].
*   **Index Format**: 16-bit (`R16_UINT`) or 32-bit (`R32_UINT`) [S1, unofficial].
*   **USE_MUTEX**: Compile-time flag. If true, uses 4 `Texture3D` grids and a mutex texture; if false, uses a single `Texture3D` with atomic overrides [S2, unofficial].

### Edge Cases and Pitfalls
*   **Race Conditions**: Multiple threads may access the same voxel simultaneously. The Front-then-Back processing order helps, but atomic operations are required for correctness [S3].
*   **Index Stride**: Must be exactly 2 or 4 bytes. Other strides cause exceptions [S1, unofficial].
*   **Texture Handling**: Textures must be added to the voxelizer before voxelization and bound as SRVs with anisotropic filtering and wrap addressing [S1, unofficial].
*   **Precision**: Floating point comparisons in plane tests should account for epsilon errors near triangle edges [G].

## Subtopics

### [GPU Mesh Voxelization (Schwarz & Seidel 2010)](1633-gpu-mesh-voxelization-schwarz-seidel-2010.md)
Uses explicit geometric containment tests per voxel within a triangle's bounding box, providing exact surface occupancy. It differs from rasterization-based methods by avoiding projection artifacts and is chosen when geometric precision is critical.

### [Multi-channel Surface Voxelization](1899-multi-channel-surface-voxelization.md)
Extends surface voxelization to store multiple attributes (albedo, normals, SH coefficients) per voxel in a single pass. It is chosen when rich surface data is needed for rendering, at the cost of higher memory bandwidth.

### [GPU Rasterization-based Octree Building](2406-gpu-rasterization-based-octree-building.md)
Builds an octree structure directly on the GPU using rasterization techniques to determine node occupancy. It is chosen for adaptive resolution scenes where memory efficiency is prioritized over uniform grid simplicity.

## Operating characteristics
*   **Time Complexity**: $O(T \cdot B)$ for compute shader variants, where $T$ is the number of triangles and $B$ is the average number of voxels in a triangle's bounding box [S3]. For rasterization-assisted variants, complexity is proportional to the number of fragments generated across 4 views [S1, unofficial].
*   **Memory Complexity**: $O(W \cdot H \cdot D)$ for the voxel grid [S3]. Additional memory for vertex/index buffers and temporary UAVs [S1, unofficial].
*   **Performance**: Real-time performance is achievable on modern GPUs (DX11/DX12) for resolutions up to $256^3$ [S1, unofficial][S2, unofficial][S3].
*   **Strengths**: High parallelism, flexible intersection logic, supports solid voxelization via depth peeling [S2, unofficial].
*   **Limitations**: Race conditions require careful synchronization; bounding box iteration can be inefficient for sparse meshes [S3].

## Background
The technique originated from the need for fast, GPU-accelerated voxelization for real-time ray marching and volume rendering [S1, unofficial]. The "AfterHumanDeclined" (AHD) implementation introduced the D3D11 two-pass approach [S1, unofficial]. VoxelizerX12 extended this to DirectX 12 with tessellation and depth peeling [S2, unofficial]. The "GPU Boxelizer" approach, detailed in Unity Graphics Programming, demonstrated the feasibility of pure compute shader geometric tests [S3].

## Sources
[S1] nustxujun/AfterHumanDeclined, https://github.com/nustxujun/AfterHumanDeclined, fetched 2026-10-26 (unofficial)
[S2] StarsX/VoxelizerX12, https://github.com/StarsX/VoxelizerX12, fetched 2026-10-27 (unofficial)
[S3] Real-Time GPU-Based Voxelizer - GitHub Pages, https://freder.github.io/UnityGraphicsProgrammingBook1/html-translated/vol2/Chapter%201%20_%20Real-Time%20GPU-Based%20Voxelizer.html, fetched 2026-10-27
