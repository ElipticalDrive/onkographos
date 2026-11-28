# GPU Voxelization via Compute Shaders

Also called: AHD GPU Voxelizer, AfterHumanDeclined, AfterHumanDeclined (AHD) GPU Voxelizer, GPU Boxelizer, GPUVoxelizer, VoxelizerX12, VoxelizerX

## Overview
GPU Voxelization via Compute Shaders converts a triangle mesh into a voxel grid by offloading geometric intersection and occupancy logic to parallel compute kernels [S1, unofficial][S3]. It distinguishes itself from [GPU Voxelization](2633-gpu-voxelization.md) by separating surface generation (front/back facing) and interior filling into distinct compute dispatches, allowing for precise control over surface attributes and solid filling [S3].

## How it works
The method implements a pipeline of resource preparation, surface voxel generation via compute shaders, and optional interior filling [S3].

### Inputs and Outputs
**Inputs:**
*   Triangle mesh data: Vertex positions (Vector3), optional normals, optional colors/textures, and index buffer (UInt16 or UInt32) [S1, unofficial][S3].
*   Grid parameters: Resolution (target voxel count per axis) and mesh bounding box [S3].
*   View/Projection matrices: For rasterization-based variants (AHD) [S1, unofficial]; or spatial bounds for compute-based variants (Boxelizer) [S3].

**Outputs:**
*   Voxel Grid: A 1D or 3D array of `Voxel_t` structures representing occupied voxels [S3]. Each voxel typically stores position, color, and surface flags (front/back) [S1, unofficial][S3].

### Data Structures

**1. Voxel Resource (AHD Variant) [S1, unofficial]:**
*   `mVertexBuffer`: ID3D11Buffer containing vertex data.
*   `mIndexBuffer`: ID3D11Buffer containing index data.
*   `mAABB`: Axis-Aligned Bounding Box of the mesh.
*   `mVertexStride`: Size of a single vertex in bytes.
*   `mIndexStride`: Size of a single index (2 or 4 bytes) [S1, unofficial].

**2. Voxel_t (Boxelizer Variant) [S3]:**
*   Stored as a 1D `ComputeBuffer` of size `width * height * depth` [S3].
*   Index calculation: `index = x + y * width + z * width * height` [S3].
*   Fields: Position, Color, Front/Back flags.

**3. Constant Buffers (AHD Variant) [S1, unofficial]:**
*   `EffectParameter`: Contains `bcount` (bool, pass selector), `world`, `width`, `height`, `depth`, and `views` (array of 4 projection matrices) [S1, unofficial].

**4. Constant Buffers (Boxelizer Variant) [S3]:**
*   `_Start`, `_End`, `_Size`: Vector3 uniforms defining the voxel grid spatial bounds [S3].
*   `_Unit`, `_InvUnit`, `_HalfUnit`: Float uniforms for voxel size and inverse/half-size [S3].
*   `_Width`, `_Height`, `_Depth`: Int uniforms for grid dimensions [S3].
*   `_TriangleCount`: Int uniform for the number of triangles [S3].

### Numbered Steps

1.  **Prepare Resources:**
    *   Calculate the global AABB of the input mesh [S1, unofficial].
    *   Compute the center of the AABB and a translation matrix to center it at the origin: $M_{translation} = \text{XMMatrixTranslation}(-center.x, -center.y, -center.z)$ [S1, unofficial].
    *   For compute-based variants, calculate voxel unit size: $unit = \frac{maxLength}{resolution}$, where $maxLength$ is the max dimension of the bounding box [S3].
    *   Calculate grid dimensions: $width = \lceil \frac{size.x}{unit} \rceil$, $height = \lceil \frac{size.y}{unit} \rceil$, $depth = \lceil \frac{size.z}{unit} \rceil$ [S3].
    *   Expand bounds by one unit to ensure coverage: $start = bounds.min - (unit, unit, unit)$, $end = bounds.max + (unit, unit, unit)$ [S3].

2.  **Allocate Output Buffer:**
    *   Create a 1D Compute Buffer (or UAV buffer in AHD) of size $width \times height \times depth$ [S3].
    *   Initialize all voxels to empty/unoccupied [S3].

3.  **Generate Surface Voxels (Compute Shader Dispatch):**
    *   **Dispatch SurfaceFront:** Launch a compute shader kernel (`SurfaceFront`) with thread group size $(\lceil \frac{triangleCount}{ThreadX} \rceil, 1, 1)$ [S3].
        *   Each thread processes one triangle index [S3].
        *   Retrieve triangle vertices and calculate normal [S3].
        *   Determine front/back facing based on normal dot product with view direction (e.g., $(0,0,1)$) [S3].
        *   If front-facing, calculate the triangle's AABB in voxel space [S3].
        *   Iterate through all voxels within the triangle's AABB [S3].
        *   Perform geometric intersection test to determine if the voxel intersects the triangle [S3].
        *   If intersected, write voxel data (position, color, front flag) to the output buffer [S3]. Use atomic operations or mutexes if multiple threads may write to the same voxel [S2, unofficial][S3].
    *   **Dispatch SurfaceBack:** Launch a compute shader kernel (`SurfaceBack`) with the same thread group dimensions [S3].
        *   Process only back-facing triangles.
        *   This order ensures that if a voxel is hit by both front and back faces, the back face can override or merge with the front face data, resolving ambiguity [S3].

4.  **Fill Interior Voxels (Optional/Compute Variant) [S3]:**
    *   **Dispatch Volume:** Launch a compute shader kernel (`Volume`) with thread group size $(\lceil \frac{width}{ThreadX} \rceil, \lceil \frac{height}{ThreadY} \rceil, \lceil \frac{depth}{ThreadZ} \rceil)$ [S3].
        *   Each thread processes one voxel coordinate $(x,y,z)$ [S3].
        *   Check if the voxel is on the surface (already occupied) [S3].
        *   If empty, perform a ray-cast or distance check from the voxel center to the mesh surface to determine if it is inside the mesh [G].
        *   If inside, mark the voxel as occupied and fill with appropriate color/normal [S3].

5.  **Read Back Results:**
    *   Copy the GPU buffer data to a staging buffer and read back to CPU memory [S1, unofficial][S3].

### Pseudocode (Boxelizer Variant)

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def surface_front(tri_buffer, vert_buffer, voxels, triangle_count, width, height, depth, start, inv_unit, half_unit):
    for idx in range(triangle_count):
        i0 = tri_buffer[idx * 3]
        i1 = tri_buffer[idx * 3 + 1]
        i2 = tri_buffer[idx * 3 + 2]
        
        va = vert_buffer[i0]
        vb = vert_buffer[i1]
        vc = vert_buffer[i2]
        
        normal = normalize(cross(vb - va, vc - va))
        if dot(normal, (0, 0, 1)) <= 0:
            continue
            
        min_v = min(va, min(vb, vc))
        max_v = max(va, max(vb, vc))
        
        ix_min = clamp(floor((min_v.x - start.x) * inv_unit), 0, width - 1)
        ix_max = clamp(floor((max_v.x - start.x) * inv_unit), 0, width - 1)
        iy_min = clamp(floor((min_v.y - start.y) * inv_unit), 0, height - 1)
        iy_max = clamp(floor((max_v.y - start.y) * inv_unit), 0, height - 1)
        iz_min = clamp(floor((min_v.z - start.z) * inv_unit), 0, depth - 1)
        iz_max = clamp(floor((max_v.z - start.z) * inv_unit), 0, depth - 1)
        
        for z in range(iz_min, iz_max + 1):
            for y in range(iy_min, iy_max + 1):
                for x in range(ix_min, ix_max + 1):
                    center = start + (x, y, z) * inv_unit + half_unit
                    if intersect_triangle(center, va, vb, vc):
                        voxels[x + y * width + z * width * height] = Voxel(center, color, True)

def volume_fill(voxels, width, height, depth, start, unit, half_unit):
    for z in range(depth):
        for y in range(height):
            for x in range(width):
                idx = x + y * width + z * width * height
                if voxels[idx].occupied:
                    continue
                center = start + (x, y, z) * unit + half_unit
                if is_inside_mesh(center):
                    voxels[idx] = Voxel(center, interior_color, False)
```

### Parameters
*   `resolution`: Target voxel grid resolution (e.g., 128, 256, 512) [S3].
*   `USE_MUTEX`: Compile-time flag. If true, uses multiple mutexed grids; if false, uses a single grid with atomic operations [S2, unofficial].
*   `scale`: Scaling factor applied to voxel grid dimensions (AHD) [S1, unofficial].
*   `FrameCount`: Number of frames for double/triple buffering of constant buffers (D12) [S2, unofficial].

### Edge Cases and Pitfalls
*   **Race Conditions:** Multiple threads may access the same voxel data simultaneously [S3].
*   **Index Format:** Index buffer stride must be 2 or 4 bytes. If 2, format is R16_UINT; if 4, format is R32_UINT. Other strides throw an exception [S1, unofficial].
*   **Texture Handling:** If a resource has a texture, it must be added to the Voxelizer via `addTexture` before voxelization. The texture is bound as a Shader Resource View (SRV) with anisotropic filtering and wrap addressing [S1, unofficial].
*   **Precision:** Floating-point precision issues may arise when mapping continuous triangle coordinates to discrete voxel indices. Clamping and careful AABB calculation are required [S3].

## Subtopics

### [GPU Mesh Voxelization (Schwarz & Seidel 2010)](1633-gpu-mesh-voxelization-schwarz-seidel-2010.md)
GPU Mesh Voxelization converts a triangle mesh into a set of occupied voxel coordinates within a 3D grid by performing explicit geometric intersection tests on the GPU. Unlike rasterization-based methods that project geometry into a grid, this approach iterates over voxels (or triangles) and uses Separating Axis Theorem (SAT) or plane/projection tests to determine occupancy. It is distinguished from its sibling [Multi-channel Surface Voxelization](1899-multi-channel-surface-voxelization.md) by using explicit geometric intersection logic rather than hardware rasterization, and from [GPU Rasterization-based Octree Building](2406-gpu-rasterization-based-octree-building.md) by producing a dense or sparse voxel grid directly without hierarchical tree construction.

### [Multi-channel Surface Voxelization](1899-multi-channel-surface-voxelization.md)
Multi-channel surface voxelization is a real-time GPU technique that rasterizes 3D geometry into a uniform voxel grid while simultaneously computing and storing multiple surface attributes (such as albedo, normals, and spherical harmonics coefficients) per voxel. It distinguishes itself from binary occupancy voxelization methods by supporting partial occupancy, transparency, and rich multi-channel data in a single pass, albeit at a higher computational cost.

### [GPU Rasterization-based Octree Building](2406-gpu-rasterization-based-octree-building.md)
GPU Rasterization-based Octree Building is a technique for constructing sparse voxel octrees directly on the GPU by exploiting the rasterization pipeline to identify occupied leaf nodes. It solves the problem of efficiently generating hierarchical voxel data for fully dynamic scenes by rasterizing geometry from multiple orthogonal directions to eliminate boundary holes inherent in single-pass methods. Unlike sibling methods that may require pre-processing or specific primitive types, this approach handles any hardware-supported rasterizable geometry in a single geometry submission pass, leveraging 32-bit integer textures and bitwise operations for high performance. It differs from [GPU Voxelization via Compute Shaders](2549-gpu-voxelization-via-compute-shaders.md) by using rasterization fragments to drive the octree traversal rather than a purely compute-based atomic accumulation, and from [GPU Mesh Voxelization (Schwarz & Seidel 2010)](1633-gpu-mesh-voxelization-schwarz-seidel-2010.md) by explicitly building the hierarchical octree structure during the rasterization phase rather than just filling a flat voxel grid.

### [GPU Voxelizer using Nearest Vertex and Normal Dot Product](17779-gpu-voxelizer-using-nearest-vertex-and-normal-dot-product.md)
This method is a real-time, brute-force voxelization technique that determines the interiority of a voxel relative to a mesh by identifying the nearest mesh vertex and analyzing the geometric relationship between the vertex normal and the vector to the voxel. It differs from sibling methods like [GPU Mesh Voxelization (Schwarz & Seidel 2010)](1633-gpu-mesh-voxelization-schwarz-seidel-2010.md) which rely on ray-casting or projection, and [Multi-channel Surface Voxelization](1899-multi-channel-surface-voxelization.md) which uses multi-sampling, by avoiding rasterization entirely in favor of direct vertex proximity checks. The technique is primarily distinguished by its reliance on a dot-product threshold for interiority, although the source implementation notes this specific logic is potentially buggy and often superseded by distance-only checks in practice.

## Operating characteristics
*   **Time Complexity:** $O(T \cdot V_{local})$ for surface generation, where $T$ is the number of triangles and $V_{local}$ is the number of voxels in the triangle's AABB. Interior filling is $O(V_{total})$ [S3].
*   **Memory Complexity:** $O(V_{total})$ for the voxel grid, where $V_{total} = width \times height \times depth$ [S3].
*   **Performance:** Real-time performance is achievable on modern GPUs (DirectX 11/12) for resolutions up to 512^3, depending on mesh complexity [S1, S2, unofficial].
*   **Strengths:** Explicit control over surface attributes, suitable for complex geometry [S3].
*   **Limitations:** Race conditions require atomic operations or mutexes, which can reduce performance. Interior filling adds significant computational cost [S3].

## Background
GPU Voxelization via Compute Shaders originated from the need for real-time voxelization in graphics applications. The AHD (AfterHumanDeclined) variant was developed for DirectX 11 and Shader Model 5.0, leveraging compute shaders for explicit geometric processing [S1, unofficial]. The VoxelizerX12 variant extends this to DirectX 12, incorporating tessellation and depth peeling for solid voxelization [S2, unofficial]. The Boxelizer variant is part of a Unity-based educational series on GPU programming, demonstrating the use of compute shaders for parallel geometric intersection tests [S3].

## Sources
[S1] nustxujun/AfterHumanDeclined, https://github.com/nustxujun/AfterHumanDeclined, fetched 2026-10-26 (unofficial)
[S2] StarsX/VoxelizerX12, https://github.com/StarsX/VoxelizerX12, fetched 2026-10-27 (unofficial)
[S3] Real-Time GPU-Based Voxelizer - GitHub Pages, https://freder.github.io/UnityGraphicsProgrammingBook1/html-translated/vol2/Chapter%201%20_%20Real-Time%20GPU-Based%20Voxelizer.html, fetched 2026-10-27
