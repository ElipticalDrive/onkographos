# Mesh to Voxel Generation via AABB-Triangle Intersection

Also called: MeshToVoxel, Triangle Mesh Voxelization, Voxelizer

## Overview
This method converts a 3D triangle mesh into a solid, non-hollow voxel grid by first identifying surface voxels via Axis-Aligned Bounding Box (AABB) vs. Triangle intersection tests, and then filling the interior using a scanline fill algorithm [S1, S2, unofficial]. It differs from its siblings by explicitly separating the surface shell generation (using SAT-based intersection) from the volume filling (using parity-based scanlines), ensuring watertight interiors suitable for deep physical simulations like heat diffusion or fracture [S1, S2, unofficial]. Unlike [Fake Voxelization via Triangle-Cube Intersection](6951-fake-voxelization-via-triangle-cube-intersection.md) which outputs only surface voxels, this method produces a solid volume [S3, unofficial].

## How it works
**Inputs:** A triangle mesh (list of vertices and indices) and a voxel size parameter `_unitSize` [S1, S3, unofficial].
**Outputs:** A 3D integer array `_voxels[y, z, x]` representing the voxel grid, where values indicate occupancy [S3, unofficial].

### Data Structures
*   **Triangle:** Contains three `Vector3` fields: `p1`, `p2`, `p3` representing the vertices in world space [S3, unofficial].
*   **_voxels:** A 3D integer array `int[,,]` with dimensions `[y, z, x]`. Values are `0` (empty), `1` (intersected surface, temporary), or `2` (occupied/filled, final) [S3, unofficial].
*   **Bounds:** A bounding box defined by `min` and `max` vectors in world space [S3, unofficial].

### Steps

1.  **Transform and Compute Bounds:**
    Retrieve mesh vertices and transform them to world space using the model's transformation matrix [S3, unofficial]. Compute the world-space bounding box by finding the minimum and maximum coordinates of all transformed vertices [S3, unofficial].
    *   Round the bounding box `min` down to the nearest multiple of `_unitSize` [S3, unofficial].
    *   Round the bounding box `max` up to the nearest multiple of `_unitSize` [S3, unofficial].
    *   **Edge Case:** If any dimension of `max` equals `min` after rounding, add `_unitSize` to `max` to ensure non-zero grid dimensions [S3, unofficial].
    *   Calculate grid dimensions: $N_x = \lceil (max_x - min_x) / \text{unitSize} \rceil$, $N_y = \lceil (max_y - min_y) / \text{unitSize} \rceil$, $N_z = \lceil (max_z - min_z) / \text{unitSize} \rceil$ [S1, S3, unofficial]. Note: `Mathf.RoundToInt` may introduce slight inaccuracies in grid coverage but ensures integer array allocation [S3, unofficial].

2.  **Initialize Grid:**
    Allocate `_voxels` with dimensions `[N_y, N_z, N_x]` initialized to `0` [S3, unofficial].

3.  **Surface Shell Generation (AABB-Triangle Intersection):**
    Iterate through every voxel in the grid (typically in Y, Z, X order) [S3, unofficial].
    For each voxel at indices $(i_x, i_y, i_z)$:
    *   Compute the voxel center in world space:
        $$ \text{center} = (i_x \cdot \text{unitSize} + \frac{\text{unitSize}}{2}, i_y \cdot \text{unitSize} + \frac{\text{unitSize}}{2}, i_z \cdot \text{unitSize} + \frac{\text{unitSize}}{2}) + \text{bounds.min} $$
        [S3, unofficial]
    *   For each triangle in the mesh, perform an AABB-Triangle intersection test using the Separating Axis Theorem (SAT) [S3, unofficial].
    *   If an intersection is found, set `_voxels[i_y, i_z, i_x] = 1` and break the triangle loop (optimization: only one intersection is needed to mark the surface) [S3, unofficial].

    **AABB-Triangle Intersection Test (SAT):**
    Given voxel center `C` and half-size `h = unitSize / 2`, and triangle vertices `v1, v2, v3`:
    1.  Transform triangle vertices relative to box center: $v'_i = v_i - C$ [S3, unofficial].
    2.  Compute edge vectors: $e_1 = v_2 - v_1$, $e_2 = v_3 - v_2$, $e_3 = v_1 - v_3$ [S3, unofficial].
    3.  Test separation axes:
        *   **Axis-Aligned Axes (X, Y, Z):** Project triangle vertices and box onto the axis. Separation if intervals do not overlap.
        *   **Edge Cross Axes:** Test axes $e_i \times \hat{x}$, $e_i \times \hat{y}$, $e_i \times \hat{z}$ for $i \in \{1,2,3\}$ [S3, unofficial].
            *   Example for X-axis test with edge $e_1$:
                $$ \text{min} = \min(a \cdot v_{1y} - b \cdot v_{1z}, a \cdot v_{3y} - b \cdot v_{3z}) $$
                $$ \text{max} = \max(a \cdot v_{1y} - b \cdot v_{1z}, a \cdot v_{3y} - b \cdot v_{3z}) $$
                $$ \text{rad} = |e_{1z}| \cdot h_y + |e_{1y}| \cdot h_z $$
                Separation if $\text{min} > \text{rad}$ or $\text{max} < -\text{rad}$ [S3, unofficial].
        *   **Triangle Normal:** Compute normal $n = e_1 \times e_2$.
            *   Plane-Box overlap test: $d = -\text{Dot}(n, v_1)$. Compute $v_{\text{min}}$ and $v_{\text{max}}$ of the box along the normal direction (extremal points). Separation if $\text{Dot}(n, v_{\text{min}}) + d > 0$. Intersection if $\text{Dot}(n, v_{\text{max}}) + d \ge 0$ [S3, unofficial].

4.  **Scanline Fill (Interior Filling):**
    Process the voxel grid slice by slice (e.g., by Z), and each slice row by row (e.g., by X) [S1, S2, unofficial].
    *   Initialize a `pen` state to `Up` (not filling) at the start of each row [S1, S2, unofficial].
    *   Iterate through voxels in the row (increasing X).
    *   If the current voxel is empty (`0`):
        *   If `pen` is `Down`, mark it as solid (`2`) [S1, S2, unofficial].
        *   If `pen` is `Up`, leave it empty (`0`) [S1, S2, unofficial].
    *   If the current voxel is a surface voxel (`1`):
        *   Cast a horizontal ray from left to right through the center of the voxel to find all intersecting triangles [S1, S2, unofficial].
        *   Identify the 'last' triangle $T_{\text{last}}$ intersected by the ray (the one with the largest intersection parameter or furthest in the scan direction) [S1, S2, unofficial].
        *   **Pen State Update:** Let $n_x$ be the x-component of the normal of $T_{\text{last}}$ [S2, unofficial].
            *   If $n_x < 0$ (normal faces left, outward from interior relative to scan), set `pen = Down` [S1, S2, unofficial].
            *   If $n_x > 0$ (normal faces right, inward or exiting), set `pen = Up` [S1, S2, unofficial].
            *   *Note:* Some implementations re-evaluate the facing direction at each intersection to toggle the pen state [S1, unofficial].
    *   Repeat for every row in the slice, and every slice in the volume [S1, S2, unofficial].

5.  **Finalization:**
    Convert temporary surface markers (`1`) to final occupied voxels (`2`) if not already done during fill, or simply treat `1` and `2` as solid depending on the rendering pipeline [S3, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mesh_to_voxel(mesh, unit_size):
    # 1. Transform and Compute Bounds
    vertices_world = [transform_point(v) for v in mesh.vertices]
    min_coord = Vector3(float('inf'), float('inf'), float('inf'))
    max_coord = Vector3(float('-inf'), float('-inf'), float('-inf'))
    
    for v in vertices_world:
        min_coord = Vector3(min(min_coord.x, v.x), min(min_coord.y, v.y), min(min_coord.z, v.z))
        max_coord = Vector3(max(max_coord.x, v.x), max(max_coord.y, v.y), max(max_coord.z, v.z))
        
    min_coord = round_down(min_coord, unit_size)
    max_coord = round_up(max_coord, unit_size)
    
    # Handle zero-size dimensions
    if max_coord.x == min_coord.x: max_coord.x += unit_size
    if max_coord.y == min_coord.y: max_coord.y += unit_size
    if max_coord.z == min_coord.z: max_coord.z += unit_size
    
    bounds_min = min_coord
    bounds_max = max_coord
    
    # 2. Initialize Grid
    dim_x = round_int((bounds_max.x - bounds_min.x) / unit_size)
    dim_y = round_int((bounds_max.y - bounds_min.y) / unit_size)
    dim_z = round_int((bounds_max.z - bounds_min.z) / unit_size)
    
    voxels = Array3D(dim_y, dim_z, dim_x, init=0)
    
    # Precompute triangles in world space
    triangles = []
    for i in range(0, len(mesh.indices), 3):
        p1 = vertices_world[mesh.indices[i]]
        p2 = vertices_world[mesh.indices[i+1]]
        p3 = vertices_world[mesh.indices[i+2]]
        triangles.append(Triangle(p1, p2, p3))
        
    unit_size_half = unit_size * 0.5
    
    # 3. Surface Shell Generation
    for y in range(dim_y):
        for z in range(dim_z):
            for x in range(dim_x):
                center = Vector3(
                    x * unit_size + unit_size_half,
                    y * unit_size + unit_size_half,
                    z * unit_size + unit_size_half
                ) + bounds_min
                
                for tri in triangles:
                    if aabb_triangle_intersect(center, unit_size, tri):
                        voxels[y, z, x] = 1
                        break
                        
    # 4. Scanline Fill
    for z in range(dim_z):
        for y in range(dim_y):
            pen = 'Up'
            for x in range(dim_x):
                if voxels[y, z, x] == 0:
                    if pen == 'Down':
                        voxels[y, z, x] = 2
                elif voxels[y, z, x] == 1:
                    # Ray cast through center to find last intersecting triangle
                    last_tri = ray_intersect_last(center, triangles)
                    if last_tri is not None:
                        nx = last_tri.normal.x
                        if nx < 0:
                            pen = 'Down'
                        else:
                            pen = 'Up'
                    else:
                        # Fallback if ray misses all triangles (should not happen for closed mesh)
                        pass
                        
    return voxels

def aabb_triangle_intersect(center, unit_size, tri):
    # Simplified SAT implementation
    half = unit_size * 0.5
    v1 = tri.p1 - center
    v2 = tri.p2 - center
    v3 = tri.p3 - center
    
    e1 = v2 - v1
    e2 = v3 - v2
    e3 = v1 - v3
    
    # Test axes: X, Y, Z, e1xX, e1xY, e1xZ, e2xX, e2xY, e2xZ, e3xX, e3xY, e3xZ, tri_normal
    # ... (full SAT logic as described in sources) ...
    return True # Placeholder for actual separation check
```

## Subtopics

### [Level 1 Triangle-Voxel Classification](4872-level-1-triangle-voxel-classification.md)
Classifies input triangles into a coarse 3D voxel grid by mapping vertices to voxel indices and testing overlap with all candidate voxels the triangle spans. It identifies candidate voxels via vertex mapping rather than testing every voxel against every triangle, enabling parallel processing where each thread handles one triangle.

### [Tessellation-based Voxelization](5430-tessellation-based-voxelization.md)
A GPU-accelerated method for generating 3D voxel grids from triangle meshes by leveraging the hardware tessellation pipeline to subdivide geometry before rasterization. It differs from fixed-function or CPU-based voxelizers by enabling adaptive triangle density and displacement mapping during the generation phase, allowing for higher fidelity in complex regions.

### [Fake Voxelization via Triangle-Cube Intersection](6951-fake-voxelization-via-triangle-cube-intersection.md)
A surface-only voxelization technique that digitizes a triangulated mesh by independently testing each triangle against a grid of axis-aligned cubes (voxels). Unlike solid voxelization methods that fill the interior volume using Constructive Solid Geometry (CSG), this approach outputs only the voxels intersected by the mesh surface, making it computationally simpler and suitable for generating surface representations. It is distinguished from its siblings by its independence from tessellation depth and its reliance on a template-based intersection test to control topological separability (6- or 26-connectivity).

### [Robust Mesh Voxelization via Exact Predicates](8770-robust-mesh-voxelization-via-exact-predicates.md)
A mesh-native voxelizer that produces an occupancy grid or a signed-distance field (SDF) over a Piecewise Linear Complex (PLC)'s bounding box, using exact-predicate classification to avoid epsilon-ray speckle. Unlike [Level 1 Triangle-Voxel Classification](4872-level-1-triangle-voxel-classification.md) which may rely on approximate ray casting, this method uses Shewchuk's exact predicates for geometric calculations to ensure robust classification of watertight meshes in single precision. It differs from [Tessellation-based Voxelization](5430-tessellation-based-voxelization.md) by operating directly on the input triangle mesh without prior subdivision, and from [Fake Voxelization via Triangle-Cube Intersection](6951-fake-voxelization-via-triangle-cube-intersection.md) by providing true inside/outside parity checks rather than mere intersection counts.

### [Triangle Subdivision for Voxelization](11480-triangle-subdivision-for-voxelization.md)
Triangle Subdivision for Voxelization is a pre-processing method that recursively splits mesh triangles to ensure they are small enough to be voxelized efficiently and predictably. It solves the problem of variable voxel counts per triangle, which complicates memory allocation and parallel processing in out-of-core or batch-based voxelization pipelines. Unlike general-purpose mesh smoothing or tessellation, this method is geometry-driven, targeting specific edge-length or area thresholds relative to the voxel resolution to guarantee homogeneous voxel generation and compact Morton index ranges. It differs from [Level 1 Triangle-Voxel Classification](4872-level-1-triangle-voxel-classification.md) by actively modifying the mesh geometry rather than just classifying existing triangles, and from [Tessellation-based Voxelization](5430-tessellation-based-voxelization.md) by using a deterministic, CPU/GPU-friendly iterative queue rather than hardware tessellation pipelines.

### Mesh-based VCP Grid Generation
Computes the Vector-to-Closest-Point (VCP) from a grid point to the mesh triangles using barycentric coordinates of the perpendicular projection [S4, unofficial]. If the projection falls inside the triangle, the VCP is the projection point; otherwise, it is the closest point on the triangle's edges or vertices [S4, unofficial]. Triangles are inserted into an empty grid by updating grid points within the triangle's bounding box if they are empty or have a greater VCP distance [S4, unofficial]. The VCP data is expanded using a region growing algorithm that iterates over grid points, propagating triangle references and minimizing VCP distances [S4, unofficial].

### Exact Voxelization Method
An alternative voxelization method implemented in binvox, written by Eric Haines, using triangle overlap code by Tomas Akenine-Möller [S5]. It serves as a reference for robust intersection testing but has known pitfalls: version 1.17 had a bug where triangles completely at max x/y/z were not voxelized correctly, and version 1.02 had a bug where the `-e` option caused triangles to be voxelized too thick [S5].

## Operating characteristics
*   **Time Complexity:** $O(N_v \cdot N_t)$ in the naive implementation, where $N_v$ is the number of voxels and $N_t$ is the number of triangles, as each voxel is tested against all triangles [S3, unofficial]. Spatial acceleration structures (like BVH or grid hashing) are not described in the core method but are implied for performance in larger scenes [G]. The scanline fill is $O(N_v)$.
*   **Memory Complexity:** $O(N_x \cdot N_y \cdot N_z)$ for the voxel grid [S3, unofficial].
*   **Performance:** The method is computationally intensive for high-resolution grids due to the brute-force voxel-triangle intersection loop [S3, unofficial]. Parallelization (e.g., using tasks/threads for each voxel) can improve performance on multi-core CPUs [S3, unofficial].
*   **Strengths:** Produces watertight, solid voxel models suitable for physics simulations [S1, unofficial]. Simple to implement [S1, unofficial].
*   **Limitations:** Slow for large meshes and high resolutions without acceleration structures. Precision issues with `RoundDown`/`RoundUp` and floating-point comparisons may lead to slight inaccuracies in grid coverage [S3, unofficial]. The scanline fill assumes a manifold, watertight mesh; non-manifold geometry may result in incorrect filling [G].

## Background
The method was popularized by David Rosen in a 2009 blog post describing voxelization for automatic skinning and physical simulations in games [S1, S2, unofficial]. It has been implemented in various forms, including Unity components [S3, unofficial] and the `binvox` tool [S5]. The intersection test relies on the Separating Axis Theorem (SAT) [S3, unofficial].

## Sources
[S1] Triangle mesh voxelization, http://blog.wolfire.com/2009/11/Triangle-mesh-voxelization, fetched 2026-10-26 (unofficial)
[S2] Triangle Mesh Voxelization - Game Developer, https://www.gamedeveloper.com/programming/triangle-mesh-voxelization, fetched 2026-10-26 (unofficial)
[S3] MeshからVoxel生成, https://qiita.com/fukaken5050/items/cba76aa593372dc135b7, fetched 2026-10-27 (unofficial)
[S4] Vector-to-Closest-Point Octree for Surface Ray-Casting, https://doi.org/10.2312/vmv.20151259, fetched 2026-10-26 (unofficial)
[S5] binvox 3D mesh voxelizer, keywords: voxelization, voxelisation, 3D model, https://www.patrickmin.com/binvox/, fetched 2026-10-29
