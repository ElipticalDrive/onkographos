# GPU Mesh Voxelization via Plane/Projection Tests

Also called: CUDA Solid Voxelizer, CUDA Triangle Voxelization, GPU Voxelization via Plane and 2D Projection Tests, Solid Voxelization via Triangle Projection

## Overview
This method converts a triangle mesh into a solid voxel grid by testing each voxel within a triangle's bounding box against the triangle's plane equation and its 2D orthogonal projections [S1, unofficial]. It differs from its sibling [Sparse Voxel DAG Voxelization via Bounding Box Grid and Plane Projections](1680-sparse-voxel-dag-voxelization-via-bounding-box-grid-and-plan.md) by using a dense bit-packed voxel table and explicit 3D plane tests rather than a hierarchical DAG structure, and from [Surface Voxelization](1683-surface-voxelization.md) by filling the interior of the mesh volume rather than marking only surface-adjacent cells.

## How it works
### Inputs and Outputs
**Inputs:**
- `triangle_data`: A flat array of `float` values containing vertex coordinates. Each triangle consists of 9 floats ($v_{0x}, v_{0y}, v_{0z}, v_{1x}, v_{1y}, v_{1z}, v_{2x}, v_{2y}, v_{2z}$) [S1, unofficial].
- `info`: A structure containing:
  - `bbox`: An `AABox<float3>` defining the global bounding box of the mesh (`min` and `max` components) [S1, unofficial].
  - `gridsize`: A `uint3` ($N_x, N_y, N_z$) defining the resolution of the voxel grid [S1, unofficial].
  - `n_triangles`: The total number of triangles in the mesh [S1, unofficial].
  - `unit`: A `float3` representing the physical size of a single voxel cell, calculated as $(bbox.max - bbox.min) / gridsize$ [S1, unofficial].
- `voxel_table`: A pointer to an array of `unsigned int` (32-bit) representing the voxel grid [S1, unofficial].

**Outputs:**
- `voxel_table`: Modified such that bits corresponding to occupied voxels are set to 1 [S1, unofficial].

### Data Structures
**Voxel Table Layout:**
The grid is stored as a linear array of 32-bit unsigned integers. Each bit represents one voxel.
- Global voxel index $i$ corresponds to word index $w = \lfloor i / 32 \rfloor$ and bit position $b = 31 - (i \pmod{32})$ [S1, unofficial].
- The bit is set using atomic OR operations [S1, unofficial].

**Morton Encoding (Optional):**
If `morton_order` is true, voxel indices are encoded using Morton codes (Z-order curve) to improve spatial locality. This requires three device-constant lookup tables (`morton256_x`, `morton256_y`, `morton256_z`), each containing 256 `uint32_t` elements, which map 8-bit coordinate chunks to interleaved bits [S1, unofficial].

### Algorithm Steps
The algorithm processes triangles in parallel using CUDA threads. Two distinct implementation variants exist in the source material; both are described below.

#### Variant 1: 3D Plane and 2D Projection Tests
This variant tests every voxel in the triangle's 3D bounding box against the triangle's plane and its projections on the XY, YZ, and ZX planes.

1.  **Thread Initialization:**
    Calculate `thread_id = threadIdx.x + blockIdx.x * blockDim.x` and `stride = blockDim.x * gridDim.x` [S1, unofficial].

2.  **Triangle Loop:**
    While `thread_id < info.n_triangles`:
    a.  **Vertex Loading:**
        Calculate offset `t = thread_id * 9`.
        Load vertices $v_0, v_1, v_2$ from `triangle_data[t...t+8]`.
        Transform vertices to local coordinates relative to `info.bbox.min`:
        $v_i = v_i - info.bbox.min$ for $i \in \{0, 1, 2\}$ [S1, unofficial].
    b.  **Geometry Calculation:**
        Compute edge vectors:
        $e_0 = v_1 - v_0$
        $e_1 = v_2 - v_1$
        $e_2 = v_0 - v_2$
        Compute the triangle normal:
        $n = \text{normalize}(\text{cross}(e_0, e_1))$ [S1, unofficial].
    c.  **Bounding Box Calculation:**
        Compute the world-space bounding box of the triangle:
        $t\_bbox\_world.min = \min(v_0, v_1, v_2)$
        $t\_bbox\_world.max = \max(v_0, v_1, v_2)$
        Convert to grid coordinates:
        $t\_bbox\_grid.min = \text{clamp}(\lfloor t\_bbox\_world.min / info.unit \rfloor, \mathbf{0}, info.gridsize - \mathbf{1})$
        $t\_bbox\_grid.max = \text{clamp}(\lfloor t\_bbox\_world.max / info.unit \rfloor, \mathbf{0}, info.gridsize - \mathbf{1})$
        (Note: Source uses `float3_to_int3` which truncates; clamping ensures bounds safety) [S1, unofficial].
    d.  **Plane Test Preparation:**
        Compute offset vector $c$ based on the normal's sign:
        If $n.x > 0$, $c.x = info.unit.x$, else $c.x = 0$.
        Similarly for $y$ and $z$ components [S1, unofficial].
        Compute plane test distances:
        $d_1 = \text{dot}(n, c - v_0)$
        $d_2 = \text{dot}(n, (info.unit - c) - v_0)$ [S1, unofficial].
    e.  **Projection Test Preparation:**
        For each projection plane (XY, YZ, ZX), compute 2D edge normals and distances.
        
        **XY Projection:**
        2D edge normals:
        $n_{xy\_e0} = (-e_{0y}, e_{0x})$
        $n_{xy\_e1} = (-e_{1y}, e_{1x})$
        $n_{xy\_e2} = (-e_{2y}, e_{2x})$
        If $n.z < 0$, negate all $n_{xy\_e*}$ [S1, unofficial].
        Distances:
        $d_{xy\_e0} = -\text{dot}(n_{xy\_e0}, (v_{0x}, v_{0y})) + \max(0, info.unit.x \cdot n_{xy\_e0.x}) + \max(0, info.unit.y \cdot n_{xy\_e0.y})$
        (Similarly for $e_1, e_2$) [S1, unofficial].
        
        **YZ Projection:**
        2D edge normals:
        $n_{yz\_e0} = (-e_{0z}, e_{0y})$
        $n_{yz\_e1} = (-e_{1z}, e_{1y})$
        $n_{yz\_e2} = (-e_{2z}, e_{2y})$
        If $n.x < 0$, negate all $n_{yz\_e*}$ [S1, unofficial].
        Distances computed similarly [S1, unofficial].
        
        **ZX Projection:**
        2D edge normals:
        $n_{zx\_e0} = (-e_{0x}, e_{0z})$
        $n_{zx\_e1} = (-e_{1x}, e_{1z})$
        $n_{zx\_e2} = (-e_{2x}, e_{2z})$
        If $n.y < 0$, negate all $n_{zx\_e*}$ [S1, unofficial].
        Distances computed similarly [S1, unofficial].
    f.  **Voxel Loop:**
        Iterate $z$ from $t\_bbox\_grid.min.z$ to $t\_bbox\_grid.max.z$, $y$ from $min.y$ to $max.y$, $x$ from $min.x$ to $max.x$:
        1.  Compute voxel center: $p = (x \cdot info.unit.x, y \cdot info.unit.y, z \cdot info.unit.z)$ [S1, unofficial].
        2.  **Plane Test:**
            $nDOTp = \text{dot}(n, p)$
            If $(nDOTp + d_1) \cdot (nDOTp + d_2) > 0$, continue (voxel is outside the slab) [S1, unofficial].
        3.  **XY Projection Test:**
            $p_{xy} = (p.x, p.y)$
            If $\text{dot}(n_{xy\_e0}, p_{xy}) + d_{xy\_e0} < 0$ OR $\text{dot}(n_{xy\_e1}, p_{xy}) + d_{xy\_e1} < 0$ OR $\text{dot}(n_{xy\_e2}, p_{xy}) + d_{xy\_e2} < 0$, continue [S1, unofficial].
        4.  **YZ Projection Test:**
            $p_{yz} = (p.y, p.z)$
            If $\text{dot}(n_{yz\_e0}, p_{yz}) + d_{yz\_e0} < 0$ OR $\text{dot}(n_{yz\_e1}, p_{yz}) + d_{yz\_e1} < 0$ OR $\text{dot}(n_{yz\_e2}, p_{yz}) + d_{yz\_e2} < 0$, continue [S1, unofficial].
        5.  **ZX Projection Test:**
            $p_{zx} = (p.z, p.x)$
            If $\text{dot}(n_{zx\_e0}, p_{zx}) + d_{zx\_e0} < 0$ OR $\text{dot}(n_{zx\_e1}, p_{zx}) + d_{zx\_e1} < 0$ OR $\text{dot}(n_{zx\_e2}, p_{zx}) + d_{zx\_e2} < 0$, continue [S1, unofficial].
        6.  **Mark Voxel:**
            Calculate global index $i = x + y \cdot N_x + z \cdot (N_y \cdot N_x)$.
            Set bit in `voxel_table` at $i$ using atomic OR [S1, unofficial].
    g.  Increment `thread_id` by `stride` [S1, unofficial].

#### Variant 2: YZ Projection and X-Axis Ray Intersection
This variant projects the triangle onto the YZ plane, iterates over the 2D bounding box, and fills voxels along the X-axis up to the intersection with the triangle plane [S1, unofficial].

1.  **Thread Initialization:** Same as Variant 1 [S1, unofficial].
2.  **Triangle Loop:**
    While `thread_id < info.n_triangles`:
    a.  **Vertex Loading:**
        Load vertices $v_0, v_1, v_2$ from `triangle_data`.
        Transform to local coordinates: $v_i = v_i - info.bbox.min$ [S1, unofficial].
    b.  **Geometry Calculation:**
        Compute edges $e_0, e_1, e_2$ and normal $n = \text{normalize}(\text{cross}(e_0, e_1))$ [S1, unofficial].
        **Pitfall:** If $|n.x| < \epsilon$ (float error), the triangle is parallel to the YZ plane. Skip this triangle to avoid division by zero [S1, unofficial].
    c.  **YZ Projection:**
        Project vertices: $v_{i,yz} = (v_{iy}, v_{iz})$ [S1, unofficial].
        Ensure counter-clockwise winding in the YZ plane. If not CCW, swap $v_{1,yz}$ and $v_{2,yz}$ [S1, unofficial].
        Compute 2D bounding box in world coordinates:
        $bbox\_max = \max(v_{0,yz}, v_{1,yz}, v_{2,yz})$
        $bbox\_min = \min(v_{0,yz}, v_{1,yz}, v_{2,yz})$
        Map to grid indices:
        $bbox\_max\_grid.y = \lfloor bbox\_max.y / info.unit.y - 0.5 \rfloor$
        $bbox\_max\_grid.z = \lfloor bbox\_max.z / info.unit.z - 0.5 \rfloor$
        $bbox\_min\_grid.y = \lceil bbox\_min.y / info.unit.y - 0.5 \rceil$
        $bbox\_min\_grid.z = \lceil bbox\_min.z / info.unit.z - 0.5 \rceil$
        (Clamp to grid bounds) [S1, unofficial].
    d.  **Voxel Loop (Y, Z):**
        Iterate $y$ from $bbox\_min\_grid.y$ to $bbox\_max\_grid.y$, $z$ from $bbox\_min\_grid.z$ to $bbox\_max\_grid.z$:
        1.  Compute voxel center in world coordinates:
            $point = ((y + 0.5) \cdot info.unit.y, (z + 0.5) \cdot info.unit.z)$ [S1, unofficial].
        2.  **2D Point-in-Triangle Test:**
            Check if $point$ is inside the projected triangle.
            If inside (checknum $\in \{1, 2, 3\}$), verify it lies on the 'TopLeftEdge' of the corresponding edge.
            If outside (checknum $= 0$), include it (boundary handling) [S1, unofficial].
        3.  **X-Axis Intersection:**
            Calculate the maximum X grid index $x_{max}$ where the voxel is filled.
            Intersect ray from $point$ along X-axis with triangle plane:
            $x_{max} = \lfloor \text{get\_x\_coordinate}(n, v_0, point) / info.unit.x - 0.5 \rfloor$
            (Where `get_x_coordinate` solves $n \cdot (P - v_0) = 0$ for $P_x$) [S1, unofficial].
        4.  **Fill Voxels (X):**
            Iterate $x$ from $0$ to $x_{max}$:
            1.  Calculate global index $i$.
            2.  If `morton_order` is true, use `mortonEncode_LUT(x, y, z)`.
            3.  Else, use linear indexing: $i = x + y \cdot N_x + z \cdot (N_y \cdot N_x)$ [S1, unofficial].
            4.  Set bit in `voxel_table` using `setBitXor` (atomic OR) [S1, unofficial].
    e.  Increment `thread_id` by `stride` [S1, unofficial].

### Parameters
- `morton_order`: Boolean flag. If true, uses Morton codes for voxel indexing; otherwise uses linear row-major indexing [S1, unofficial].
- `voxel_table`: Pointer to the output bit-packed array [S1, unofficial].

### Edge Cases and Pitfalls
- **Parallel Triangles:** If the triangle normal's X component is near zero ($|n.x| < \epsilon$), the triangle is parallel to the YZ projection plane. Variant 2 skips these triangles to prevent division by zero in the intersection calculation [S1, unofficial].
- **Boundary Handling:** In Variant 2, voxels outside the projected triangle (checknum 0) are included, likely for robust boundary filling [S1, unofficial].
- **Precision:** Floating-point comparisons for plane tests and point-in-triangle tests should account for epsilon errors, though exact epsilon values are not specified in the source [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N_t \cdot N_{bbox})$, where $N_t$ is the number of triangles and $N_{bbox}$ is the average number of voxels in a triangle's bounding box. Each voxel requires constant-time plane and projection tests [S1, unofficial].
- **Memory Complexity:** $O(N_x N_y N_z / 32)$ for the voxel table, plus $O(N_t)$ for triangle data [S1, unofficial].
- **Performance:** The method is GPU-accelerated using CUDA compute shaders. It requires a CUDA device with compute capability 2.0 or higher for atomic operations [S1, unofficial].
- **Strengths:** Simple implementation, parallelizable, produces solid voxel grids suitable for ray marching [S1, unofficial].
- **Limitations:** Dense storage can be memory-intensive for large grids. Variant 2 may fill voxels outside the mesh if the projection logic is not strictly conservative [S1, unofficial].

## Background
This method is implemented in the open-source project "cuda_voxelizer" by Forceflow [S1, unofficial]. It is an unofficial implementation of GPU-based solid voxelization techniques, leveraging CUDA for parallel processing of triangle meshes. The specific algorithm variants described are derived from the source code repository [S1, unofficial].

## Sources
[S1] GitHub - Forceflow/cuda_voxelizer: CUDA Voxelizer to convert polygon ..., https://github.com/Forceflow/cuda_voxelizer, fetched 2026-10-27 (unofficial)
