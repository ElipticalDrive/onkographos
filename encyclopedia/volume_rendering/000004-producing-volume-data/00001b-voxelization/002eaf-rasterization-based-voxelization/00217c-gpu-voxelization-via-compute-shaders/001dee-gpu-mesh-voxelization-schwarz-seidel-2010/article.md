# GPU Mesh Voxelization (Schwarz & Seidel 2010)

Also called: Compute Shader Voxelization, GPU Mesh Voxelization via Compute Shaders, GPU Surface Voxelization, GPU Voxelizer Part 3, GPU-based Mesh Voxelization via Triangle-AABB Intersection, VoxelizeMesh, VoxelizeViaCompute

## Overview
GPU Mesh Voxelization converts a triangle mesh into a set of occupied voxel coordinates within a 3D grid by performing explicit geometric intersection tests on the GPU [S1, S2, S3, unofficial]. Unlike rasterization-based methods that project geometry into a grid, this approach iterates over voxels (or triangles) and uses Separating Axis Theorem (SAT) or plane/projection tests to determine occupancy [S1, S3, S4, unofficial]. It is distinguished from its sibling [Multi-channel Surface Voxelization](1899-multi-channel-surface-voxelization.md) by using explicit geometric intersection logic rather than hardware rasterization, and from [GPU Rasterization-based Octree Building](2406-gpu-rasterization-based-octree-building.md) by producing a dense or sparse voxel grid directly without hierarchical tree construction [S1, S6, unofficial].

## How it works
The method operates in two primary variants: a brute-force voxel-centric approach using SAT [S2, S3, unofficial] and a triangle-centric approach using plane and projection tests [S1, S5, unofficial]. Both variants require preprocessing the mesh and launching compute kernels.

### Inputs and Outputs
**Inputs:**
*   Triangle mesh defined by vertices $V$ and indices $I$.
*   Voxel grid dimensions $(G_x, G_y, G_z)$ and voxel half-size $h$ [S2, S3, S5, unofficial].
*   Mesh bounding box bounds $B_{min}, B_{max}$ [S1, S2, S3, unofficial].

**Outputs:**
*   **Variant A (SAT):** A buffer of occupied voxels, typically stored as a list of coordinates or a bit-packed table [S1, S2, S3, unofficial].
*   **Variant B (Plane/Projection):** A bit-packed voxel table (vtable) where each bit represents a voxel's occupancy, followed by a sparse extraction pass to retrieve coordinates [S1, S5, unofficial].

### Data Structures
**1. Triangle Data Buffer:**
*   Flat array of floats.
*   Layout: 9 floats per triangle ($v_{0x}, v_{0y}, v_{0z}, v_{1x}, v_{1y}, v_{1z}, v_{2x}, v_{2y}, v_{2z}$) [S1, S4, unofficial].
*   Vertices are often normalized to the $[0, 1]$ range relative to the mesh bounding box [S1, S5, unofficial].

**2. Voxel Grid Parameters:**
*   `gridSize`: Dimensions $(G_x, G_y, G_z)$ calculated as $\lceil \frac{B_{max} - B_{min}}{2h} \rceil$ [S1, S2, S3, S5, unofficial].
*   `unit` or `cellSize`: Physical size of one voxel, $2h$ [S1, S3, S5, unofficial].

**3. Voxel Table (vtable) - Variant B:**
*   1D array of 32-bit unsigned integers.
*   Size: $\lceil G_x G_y G_z / 32 \rceil$ integers [S1, S5, unofficial].
*   Each bit corresponds to a voxel index $i = x + y G_x + z G_x G_y$ [S1, S5, unofficial].

**4. Compute Shader Buffers (Variant A):**
*   `_MeshVertices`: `StructuredBuffer<float3>` [S2, S3, unofficial].
*   `_MeshTriangleIndices`: `StructuredBuffer<int>` [S2, S3, unofficial].
*   `_VoxelGridPoints`: `RWStructuredBuffer<float4>` storing $(x, y, z, w)$ where $w=1.0$ if occupied, $0.0$ otherwise [S2, S3, unofficial].

### Algorithm Steps

#### Preprocessing
1.  Calculate grid dimensions:
    $$ G_x = \lceil \frac{B_{max,x} - B_{min,x}}{2h} \rceil, \quad G_y = \lceil \frac{B_{max,y} - B_{min,y}}{2h} \rceil, \quad G_z = \lceil \frac{B_{max,z} - B_{min,z}}{2h} \rceil $$
    [S2, S3, S5, unofficial]
2.  Compute unit vector (voxel size): $u = 2h$ [S1, S5, unofficial].
3.  Normalize triangle vertices to local coordinates relative to $B_{min}$:
    $$ v'_i = v_i - B_{min} $$
    [S1, S5, unofficial]
4.  Compute triangle bounding box in grid coordinates:
    $$ t_{grid}.min = \text{clamp}(\lfloor \frac{v'_{min}}{u} \rfloor, 0, G-1) $$
    $$ t_{grid}.max = \text{clamp}(\lceil \frac{v'_{max}}{u} \rceil, 0, G-1) $$
    [S1, S5, unofficial]

#### Variant A: SAT Intersection (Voxel-Centric)
Each compute thread processes one voxel. It checks for intersection with all mesh triangles.

1.  **Thread ID:** Map $(id.x, id.y, id.z)$ to voxel index.
2.  **Voxel AABB:** Define voxel AABB centered at $C = id \cdot u + h + B_{min}$ with extents $h$ [S2, S3, unofficial].
3.  **Triangle Loop:** Iterate through all triangles $T_j$.
4.  **SAT Test:** Check if Voxel AABB intersects Triangle $T_j$ along 13 axes:
    *   3 AABB face normals: $(1,0,0), (0,1,0), (0,0,1)$ [S2, S3, unofficial].
    *   3 Triangle face normals: $n = (v_1-v_0) \times (v_2-v_1)$ [S2, S3, unofficial].
    *   9 Edge-Axis cross products: $e_{tri} \times e_{aabb}$ [S2, S3, unofficial].
5.  **Projection Check:** For each axis $a$, calculate projections:
    $$ p_{tri} = \{ \text{dot}(v_k, a) \mid k \in \{0,1,2\} \} $$
    $$ p_{aabb} = \text{dot}(C, a) \pm \text{dot}(h, |a|) $$
    Separation exists if $\max(-\max(p_{tri}), \min(p_{tri})) > r$, where $r$ is the projected radius of the AABB [S2, S3, unofficial].
6.  **Mark Occupied:** If intersection found, write to `_VoxelGridPoints` and break loop [S2, S3, unofficial].

#### Variant B: Plane/Projection Tests (Triangle-Centric)
Each compute thread processes one triangle. It iterates over voxels in the triangle's bounding box.

1.  **Thread ID:** Map to triangle index $j$.
2.  **Triangle Geometry:**
    *   Edges: $e_0 = v_1 - v_0, e_1 = v_2 - v_1, e_2 = v_0 - v_2$ [S1, S5, unofficial].
    *   Normal: $n = \text{normalize}(e_0 \times e_1)$ [S1, S5, unofficial]. Handle degenerate triangles ($|n| < 10^{-20}$) by skipping normalization [S1, S5, unofficial].
3.  **Bounding Box:** Calculate $t_{grid}.min, t_{grid}.max$ in grid coordinates [S1, S5, unofficial].
4.  **Plane Test Constants:**
    *   Define offset vector $c$: $c_i = u_i$ if $n_i > 0$, else $0$ [S1, S5, unofficial].
    *   $d_1 = \text{dot}(n, c - v_0)$ [S1, S5, unofficial].
    *   $d_2 = \text{dot}(n, (u - c) - v_0)$ [S1, S5, unofficial].
5.  **Projection Test Constants:**
    *   For each plane (XY, YZ, ZX), compute 2D edge normals.
    *   Example XY: $n_{xy\_e0} = (-e_{0y}, e_{0x})$. If $n_z < 0$, negate all XY edge normals [S1, S5, unofficial].
    *   Compute $d_{xy\_ei} = -\text{dot}(n_{xy\_ei}, v_{i,xy}) + \max(0, u_x n_{xy\_ei,x}) + \max(0, u_y n_{xy\_ei,y})$ [S1, S5, unofficial].
    *   Repeat for YZ and ZX planes [S1, S5, unofficial].
6.  **Voxel Iteration:** Loop $z \in [t_{grid}.min.z, t_{grid}.max.z]$, $y \in [t_{grid}.min.y, t_{grid}.max.y]$, $x \in [t_{grid}.min.x, t_{grid}.max.x]$.
    *   Voxel center: $p = (x u_x, y u_y, z u_z)$ [S1, S5, unofficial].
    *   **Plane Check:** If $(\text{dot}(n, p) + d_1)(\text{dot}(n, p) + d_2) > 0$, voxel is outside slab, continue [S1, S5, unofficial].
    *   **Projection Checks:**
        *   XY: If $\text{dot}(n_{xy\_ei}, p_{xy}) + d_{xy\_ei} < 0$ for any edge, continue [S1, S5, unofficial].
        *   YZ: If $\text{dot}(n_{yz\_ei}, p_{yz}) + d_{yz\_ei} < 0$ for any edge, continue [S1, S5, unofficial].
        *   ZX: If $\text{dot}(n_{zx\_ei}, p_{zx}) + d_{zx\_ei} < 0$ for any edge, continue [S1, S5, unofficial].
    *   **Mark Bit:** If all tests pass, set bit in `vtable` at index $x + y G_x + z G_x G_y$ using `atomicOr` [S1, S5, unofficial].
7.  **Sparse Extraction (Post-Processing):**
    *   Launch kernel to iterate over `vtable`.
    *   If bit set, atomically increment counter and store $(x,y,z)$ in output array [S1, S5, unofficial].

### Pseudocode (Variant B: Plane/Projection)

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxelize_mesh_gpu(triangles, bounds_min, bounds_max, half_size):
    # Preprocessing
    voxel_size = 2.0 * half_size
    grid_size = ceil((bounds_max - bounds_min) / voxel_size)
    unit = voxel_size
    vtable_size = (grid_size.x * grid_size.y * grid_size.z + 31) // 32
    vtable = [0] * vtable_size
    
    # Kernel Launch: One thread per triangle
    threads_per_block = 256
    num_blocks = ceil(len(triangles) / threads_per_block)
    
    for block_idx in range(num_blocks):
        for thread_idx in range(threads_per_block):
            tri_idx = block_idx * threads_per_block + thread_idx
            if tri_idx >= len(triangles):
                break
            
            v0, v1, v2 = triangles[tri_idx]
            # Normalize to local coords
            v0 = v0 - bounds_min
            v1 = v1 - bounds_min
            v2 = v2 - bounds_min
            
            # Edges and Normal
            e0 = v1 - v0
            e1 = v2 - v1
            e2 = v0 - v2
            n = cross(e0, e1)
            n_len = length(n)
            if n_len > 1e-20:
                n = n / n_len
            else:
                continue # Degenerate triangle
            
            # Bounding Box in Grid Coords
            t_min = clamp(floor(min(v0, v1, v2) / unit), 0, grid_size - 1)
            t_max = clamp(ceil(max(v0, v1, v2) / unit), 0, grid_size - 1)
            
            # Plane Test Setup
            c = float3(0, 0, 0)
            if n.x > 0: c.x = unit.x
            if n.y > 0: c.y = unit.y
            if n.z > 0: c.z = unit.z
            
            d1 = dot(n, c - v0)
            d2 = dot(n, (float3(unit.x, unit.y, unit.z) - c) - v0)
            
            # Projection Tests Setup
            # XY Plane
            n_xy_e0 = float2(-e0.y, e0.x)
            n_xy_e1 = float2(-e1.y, e1.x)
            n_xy_e2 = float2(-e2.y, e2.x)
            if n.z < 0:
                n_xy_e0 = -n_xy_e0
                n_xy_e1 = -n_xy_e1
                n_xy_e2 = -n_xy_e2
            
            d_xy_e0 = -dot(n_xy_e0, float2(v0.x, v0.y)) + max(0, unit.x * n_xy_e0.x) + max(0, unit.y * n_xy_e0.y)
            d_xy_e1 = -dot(n_xy_e1, float2(v1.x, v1.y)) + max(0, unit.x * n_xy_e1.x) + max(0, unit.y * n_xy_e1.y)
            d_xy_e2 = -dot(n_xy_e2, float2(v2.x, v2.y)) + max(0, unit.x * n_xy_e2.x) + max(0, unit.y * n_xy_e2.y)
            
            # YZ Plane
            n_yz_e0 = float2(-e0.z, e0.y)
            n_yz_e1 = float2(-e1.z, e1.y)
            n_yz_e2 = float2(-e2.z, e2.y)
            if n.x < 0:
                n_yz_e0 = -n_yz_e0
                n_yz_e1 = -n_yz_e1
                n_yz_e2 = -n_yz_e2
                
            d_yz_e0 = -dot(n_yz_e0, float2(v0.y, v0.z)) + max(0, unit.y * n_yz_e0.x) + max(0, unit.z * n_yz_e0.y)
            d_yz_e1 = -dot(n_yz_e1, float2(v1.y, v1.z)) + max(0, unit.y * n_yz_e1.x) + max(0, unit.z * n_yz_e1.y)
            d_yz_e2 = -dot(n_yz_e2, float2(v2.y, v2.z)) + max(0, unit.y * n_yz_e2.x) + max(0, unit.z * n_yz_e2.y)
            
            # ZX Plane
            n_zx_e0 = float2(-e0.x, e0.z)
            n_zx_e1 = float2(-e1.x, e1.z)
            n_zx_e2 = float2(-e2.x, e2.z)
            if n.y < 0:
                n_zx_e0 = -n_zx_e0
                n_zx_e1 = -n_zx_e1
                n_zx_e2 = -n_zx_e2
                
            d_zx_e0 = -dot(n_zx_e0, float2(v0.z, v0.x)) + max(0, unit.x * n_zx_e0.x) + max(0, unit.z * n_zx_e0.y)
            d_zx_e1 = -dot(n_zx_e1, float2(v1.z, v1.x)) + max(0, unit.x * n_zx_e1.x) + max(0, unit.z * n_zx_e1.y)
            d_zx_e2 = -dot(n_zx_e2, float2(v2.z, v2.x)) + max(0, unit.x * n_zx_e2.x) + max(0, unit.z * n_zx_e2.y)
            
            # Voxel Iteration
            for z in range(t_min.z, t_max.z + 1):
                for y in range(t_min.y, t_max.y + 1):
                    for x in range(t_min.x, t_max.x + 1):
                        p = float3(x * unit.x, y * unit.y, z * unit.z)
                        
                        # Plane Test
                        n_dot_p = dot(n, p)
                        if (n_dot_p + d1) * (n_dot_p + d2) > 0.0:
                            continue
                            
                        # XY Projection Test
                        p_xy = float2(p.x, p.y)
                        if dot(n_xy_e0, p_xy) + d_xy_e0 < 0.0: continue
                        if dot(n_xy_e1, p_xy) + d_xy_e1 < 0.0: continue
                        if dot(n_xy_e2, p_xy) + d_xy_e2 < 0.0: continue
                        
                        # YZ Projection Test
                        p_yz = float2(p.y, p.z)
                        if dot(n_yz_e0, p_yz) + d_yz_e0 < 0.0: continue
                        if dot(n_yz_e1, p_yz) + d_yz_e1 < 0.0: continue
                        if dot(n_yz_e2, p_yz) + d_yz_e2 < 0.0: continue
                        
                        # ZX Projection Test
                        p_zx = float2(p.z, p.x)
                        if dot(n_zx_e0, p_zx) + d_zx_e0 < 0.0: continue
                        if dot(n_zx_e1, p_zx) + d_zx_e1 < 0.0: continue
                        if dot(n_zx_e2, p_zx) + d_zx_e2 < 0.0: continue
                        
                        # Mark Voxel
                        loc = x + y * grid_size.x + z * grid_size.x * grid_size.y
                        int_idx = loc // 32
                        bit_pos = 31 - (loc % 32)
                        atomic_or(vtable[int_idx], 1 << bit_pos)

    # Sparse Extraction
    occupied_voxels = []
    counter = 0
    for i in range(grid_size.x * grid_size.y * grid_size.z):
        int_idx = i // 32
        bit_pos = 31 - (i % 32)
        if vtable[int_idx] & (1 << bit_pos):
            x = i % grid_size.x
            y = (i // grid_size.x) % grid_size.y
            z = i // (grid_size.x * grid_size.y)
            occupied_voxels.append((x, y, z))
            
    return occupied_voxels
```

### Parameters
*   **Resolution/Grid Size:** Determined by mesh bounds and voxel half-size [S1, S2, S3, S5, unofficial].
*   **Half-Size ($h$):** Typical value $0.05$ [S2, S3, unofficial]. Range $[0.005, 0.05]$ [S3, unofficial].
*   **Thread Block Size:** Fixed at 256 threads per block in some implementations [S4, S5, unofficial].

### Edge Cases and Pitfalls
*   **Degenerate Triangles:** If triangle normal length $\le 10^{-20}$, do not normalize to avoid division by zero [S1, S5, unofficial].
*   **Boundary Clamping:** Grid coordinates must be clamped to $[0, G-1]$ to prevent out-of-bounds memory access [S1, S5, unofficial].
*   **Precision Issues:** SAT implementation may suffer from precision issues with very small or degenerate triangles if edges are normalized before cross products [S2, S3, unofficial].
*   **Memory Limits:** Check available GPU memory before allocating voxel tables [S5, unofficial].

## Subtopics

### [Sparse Voxel DAG Voxelization via Bounding Box Grid and Plane Projections](1680-sparse-voxel-dag-voxelization-via-bounding-box-grid-and-plan.md)
This method voxelizes a triangle mesh into a Sparse Voxel DAG by rasterizing triangles into a Morton-coded voxel subgrid. It iterates over a 2D grid (XY) within a triangle's bounding box, determines the Z-range of voxels intersecting the triangle plane, and fills only those voxels that pass containment tests in the XY, ZX, and YZ projections.

### [Surface Voxelization](1683-surface-voxelization.md)
Surface Voxelization determines which voxels in a 3D grid are intersected by the surface of a triangle mesh. It operates on individual triangles independently, supporting non-watertight meshes without requiring the input geometry to be closed. Unlike naive pipeline-based voxelization that samples only triangle centers, this method ensures coverage by checking collisions within the triangle's bounding volume, either via brute-force iteration or Breadth-First Search (BFS).

### [GPU Mesh Voxelization via Plane/Projection Tests](6744-gpu-mesh-voxelization-via-plane-projection-tests.md)
This method converts a triangle mesh into a solid voxel grid by testing each voxel within a triangle's bounding box against the triangle's plane equation and its 2D orthogonal projections. It differs from its sibling [Sparse Voxel DAG Voxelization via Bounding Box Grid and Plane Projections](1680-sparse-voxel-dag-voxelization-via-bounding-box-grid-and-plan.md) by using a dense bit-packed voxel table and explicit 3D plane tests rather than a hierarchical DAG structure, and from [Surface Voxelization](1683-surface-voxelization.md) by filling the interior of the mesh volume rather than marking only surface-adjacent cells.

## Operating characteristics
*   **Time Complexity:**
    *   Variant A (SAT): $O(V \cdot T)$ where $V$ is the number of voxels and $T$ is the number of triangles [S2, S3, unofficial].
    *   Variant B (Plane/Projection): $O(T \cdot V_{bbox})$ where $V_{bbox}$ is the number of voxels in the triangle's bounding box [S1, S5, unofficial].
*   **Memory Complexity:** $O(G_x G_y G_z)$ for the voxel table [S1, S5, unofficial].
*   **Performance:** Supports resolutions up to $256^3$ with real-time performance (10 fps or higher) for models with up to 2 million triangles [S6, unofficial].
*   **Strengths:** Exact geometric results; no rasterization artifacts; suitable for both surface and solid voxelization [S3, S4, S6, unofficial].
*   **Limitations:** Computationally expensive for high triangle counts due to brute-force intersection testing; requires explicit triangle-voxel logic [S2, S3, S4, unofficial].
*   **When to Use:** When exact geometric intersection is required, or when rasterization-based methods are unsuitable (e.g., non-polygonal inputs, specific occupancy criteria) [S3, S6, unofficial].
*   **When Not to Use:** For very high-poly meshes where $O(V \cdot T)$ becomes prohibitive; consider spatial acceleration structures [S2, S3, unofficial].

## Background
GPU Mesh Voxelization was introduced by Schwarz and Seidel in 2010 as a real-time voxelization technique for complex models [S6, unofficial]. It leverages compute shaders (Direct3D 11, Shader Model 5.0) to perform triangle-voxel intersection tests on the GPU, avoiding the bandwidth demands of frame buffer access [S4, S6, unofficial]. The method has been implemented in various open-source projects and Unity assets, often referred to as "GPU Voxelizer Part 3" or "Compute Shader Voxelization" [S2, S4, S5, unofficial]. It is used in applications such as volume-based modeling, transparent rendering, and collision detection [S6, unofficial].

## Sources
[S1] yuyujunjun/mesh-voxelize, https://github.com/yuyujunjun/mesh-voxelize, fetched 2026-10-26 (unofficial)
[S2] bzgeb/UnityGPUMeshVoxelizerPart3, https://github.com/bzgeb/UnityGPUMeshVoxelizerPart3, fetched 2026-10-26 (unofficial)
[S3] NotABottle/Voxel-Shader, https://github.com/NotABottle/Voxel-Shader, fetched 2026-10-26 (unofficial)
[S4] michschw/GpuPro3Vox, https://github.com/michschw/GpuPro3Vox, fetched 2026-10-26 (unofficial)
[S5] ivandrodri/gpu-voxelizer, https://github.com/ivandrodri/gpu-voxelizer, fetched 2026-10-26 (unofficial)
[S6] PDF Real-time Voxelization for Complex Models - GitHub Pages, https://wo1fsea.github.io/files/About_NavMesh/P2_Real-time_Voxelization_for_Complex_Models.pdf, fetched 2026-10-29 (unofficial)
