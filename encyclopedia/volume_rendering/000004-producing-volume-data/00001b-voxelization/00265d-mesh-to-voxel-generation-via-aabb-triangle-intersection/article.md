# Mesh to Voxel Generation via AABB-Triangle Intersection

Also called: MeshToVoxel, Triangle Mesh Voxelization, Voxelizer

## Overview
This method converts a 3D triangle mesh into a solid voxel model by first identifying surface voxels through Axis-Aligned Bounding Box (AABB) vs. Triangle intersection tests, and then filling the interior volume using a scanline fill algorithm [S1, S2, unofficial]. It differs from rasterization-based voxelization by operating directly in world space with geometric intersection tests rather than projecting geometry onto a screen, and from point cloud voxelization by explicitly handling surface topology to ensure watertight interiors [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A triangle mesh defined by vertices and indices, and a voxel size parameter `_unitSize` (default 0.05f) [S2, unofficial].
*   **Output:** A 3D integer array `_voxels` of dimensions $[Y, Z, X]$ where values are `0` (empty), `1` (surface intersected), or `2` (solid interior) [S2, unofficial].

**Data Structures:**
*   `Triangle`: Contains three `Vector3` fields `p1`, `p2`, `p3` representing world-space vertices [S2, unofficial].
*   `_voxels`: A 3D array `int[,,]` indexed as `[y, z, x]` [S2, unofficial].
*   `_bounds`: A bounding box defined by `min` and `max` vectors in world space [S2, unofficial].

**Algorithm Steps:**

1.  **Coordinate Mapping and Bounds Calculation:**
    *   Transform all mesh vertices to world space using the mesh's transform matrix [S2, unofficial].
    *   Compute the world-space bounding box by finding the minimum and maximum coordinates of all transformed vertices [S2, unofficial].
    *   Round the bounding box `min` down to the nearest multiple of `_unitSize` and `max` up to the nearest multiple of `_unitSize` [S2, unofficial].
    *   If any dimension of `max` equals `min` after rounding, add `_unitSize` to `max` to ensure non-zero grid dimensions [S2, unofficial].
    *   Calculate grid dimensions $N_x, N_y, N_z$ as $\lfloor \frac{max - min}{\_unitSize} \rfloor$ [S1, S2, unofficial].

2.  **Shell Creation (Surface Voxelization):**
    *   Iterate through every voxel $(i, j, k)$ in the grid [S2, unofficial].
    *   Compute the voxel center in world space:
        $$ center = (i \cdot \_unitSize + \frac{\_unitSize}{2}, j \cdot \_unitSize + \frac{\_unitSize}{2}, k \cdot \_unitSize + \frac{\_unitSize}{2}) + min $$
        [S2, unofficial]
    *   For each voxel, iterate through all triangles in the mesh.
    *   Perform an AABB-Triangle intersection test between the voxel (defined by `center` and `_unitSize`) and the triangle [S2, unofficial].
    *   If an intersection is detected, set `_voxels[j, k, i] = 1` and break the triangle loop [S2, unofficial].

3.  **Scanline Fill (Interior Filling):**
    *   Process the voxel grid slice by slice (e.g., along the Z-axis), and each slice row by row (e.g., along the X-axis) [S1, unofficial].
    *   Maintain a `Pen` state (Up or Down), initialized to `Up` at the start of each row [S1, unofficial].
    *   For each voxel in the row:
        *   If the voxel is marked as surface (`1`):
            *   Cast a horizontal ray from left to right through the voxel center to find all intersecting triangles [S1, unofficial].
            *   Identify the "last" triangle intersected by the ray (the one with the largest X-coordinate intersection point) [S1, unofficial].
            *   Let $n_x$ be the X-component of the normal of this last triangle.
            *   Update `Pen` state: If $n_x < 0$ (facing left), set `Pen = Down`. If $n_x > 0$ (facing right), set `Pen = Up` [S1, unofficial].
            *   Mark the voxel as solid (`2`) if `Pen` is `Down`, otherwise leave it as surface (`1`) or empty depending on implementation specifics (typically surface voxels remain `1` or are converted to `2` if inside) [S1, S2, unofficial]. *Note: Source [S2, unofficial] converts `1` to `2` during instantiation, but [S1, unofficial] implies filling empty voxels when Pen is Down.*
        *   If the voxel is empty (`0`):
            *   If `Pen` is `Down`, mark the voxel as solid (`2`) [S1, unofficial].
            *   If `Pen` is `Up`, leave the voxel empty (`0`) [S1, unofficial].

**AABB-Triangle Intersection Test (SAT):**
The intersection test `AABBTriangleOverlaped(center, unitSize, triangle)` uses the Separating Axis Theorem [S2, unofficial]:
1.  Transform triangle vertices $v_1, v_2, v_3$ relative to the box center: $v_i' = v_i - center$ [S2, unofficial].
2.  Compute edge vectors: $e_1 = v_2 - v_1$, $e_2 = v_3 - v_2$, $e_3 = v_1 - v_3$ [S2, unofficial].
3.  Test separation axes:
    *   **Axis-Aligned Axes (X, Y, Z):** Project triangle vertices and box onto the axis. Separation if intervals do not overlap [S2, unofficial].
    *   **Edge Cross Axes:** Compute cross products of edges with X, Y, Z axes ($e_1 \times X$, etc.). Project triangle vertices and box onto these axes. Separation if intervals do not overlap [S2, unofficial].
    *   **Triangle Normal:** Compute normal $n = e_1 \times e_2$. Project triangle vertices and box onto $n$. Separation if $Dot(n, vmin) + d > 0$ where $d = -Dot(n, v_1)$ [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mesh_to_voxel(mesh, unit_size=0.05):
    # 1. Bounds Calculation
    vertices = [transform_point(v) for v in mesh.vertices]
    min_coord = min(vertices, key=lambda v: v.x) # Simplified min/max logic
    max_coord = max(vertices, key=lambda v: v.x)
    
    # Round bounds
    min_coord = round_down(min_coord, unit_size)
    max_coord = round_up(max_coord, unit_size)
    
    # Ensure non-zero size
    for axis in ['x', 'y', 'z']:
        if max_coord[axis] == min_coord[axis]:
            max_coord[axis] += unit_size
            
    bounds_min = min_coord
    bounds_max = max_coord
    bounds_size = bounds_max - bounds_min
    
    # Grid dimensions
    dims = [
        int(round(bounds_size.x / unit_size)),
        int(round(bounds_size.y / unit_size)),
        int(round(bounds_size.z / unit_size))
    ]
    
    voxels = [[[0 for _ in range(dims[2])] for _ in range(dims[1])] for _ in range(dims[0])]
    
    # Precompute triangles
    triangles = []
    for i in range(0, len(mesh.indices), 3):
        p1 = vertices[mesh.indices[i]]
        p2 = vertices[mesh.indices[i+1]]
        p3 = vertices[mesh.indices[i+2]]
        triangles.append(Triangle(p1, p2, p3))
        
    unit_size_half = unit_size * 0.5
    
    # 2. Shell Creation
    for y in range(dims[0]):
        for z in range(dims[1]):
            for x in range(dims[2]):
                center = Vector3(
                    x * unit_size + unit_size_half + bounds_min.x,
                    y * unit_size + unit_size_half + bounds_min.y,
                    z * unit_size + unit_size_half + bounds_min.z
                )
                
                for tri in triangles:
                    if aabb_triangle_intersect(center, unit_size, tri):
                        voxels[y][z][x] = 1
                        break
                        
    # 3. Scanline Fill
    for z in range(dims[1]):
        for y in range(dims[0]):
            pen = 'Up'
            for x in range(dims[2]):
                if voxels[y][z][x] == 1:
                    # Ray intersection to find last triangle
                    last_tri = find_last_triangle_intersecting_ray(center, triangles)
                    if last_tri:
                        nx = last_tri.normal.x
                        if nx < 0:
                            pen = 'Down'
                        else:
                            pen = 'Up'
                    voxels[y][z][x] = 2 # Mark as solid surface
                elif pen == 'Down':
                    voxels[y][z][x] = 2 # Fill interior
                    
    return voxels

def aabb_triangle_intersect(center, unit_size, tri):
    half_size = unit_size * 0.5
    box_center = center
    v1 = tri.p1 - box_center
    v2 = tri.p2 - box_center
    v3 = tri.p3 - box_center
    
    e1 = v2 - v1
    e2 = v3 - v2
    e3 = v1 - v3
    
    # Test X, Y, Z axes
    if test_axis(v1, v2, v3, half_size, Vector3(1,0,0)): return False
    if test_axis(v1, v2, v3, half_size, Vector3(0,1,0)): return False
    if test_axis(v1, v2, v3, half_size, Vector3(0,0,1)): return False
    
    # Test Edge Cross Axes
    axes = [
        cross(e1, Vector3(1,0,0)), cross(e1, Vector3(0,1,0)), cross(e1, Vector3(0,0,1)),
        cross(e2, Vector3(1,0,0)), cross(e2, Vector3(0,1,0)), cross(e2, Vector3(0,0,1)),
        cross(e3, Vector3(1,0,0)), cross(e3, Vector3(0,1,0)), cross(e3, Vector3(0,0,1))
    ]
    for axis in axes:
        if test_axis(v1, v2, v3, half_size, axis): return False
        
    # Test Triangle Normal
    normal = cross(e1, e2)
    if test_axis(v1, v2, v3, half_size, normal): return False
    
    return True

def test_axis(v1, v2, v3, half_size, axis):
    # Project vertices
    p1 = dot(v1, axis)
    p2 = dot(v2, axis)
    p3 = dot(v3, axis)
    min_p = min(p1, p2, p3)
    max_p = max(p1, p2, p3)
    
    # Project box (extremal points along axis)
    # Box is centered at 0 in local space
    r = half_size.x * abs(axis.x) + half_size.y * abs(axis.y) + half_size.z * abs(axis.z)
    
    if min_p > r or max_p < -r:
        return True # Separated
    return False
```

## Subtopics

### [Level 1 Triangle-Voxel Classification](4872-level-1-triangle-voxel-classification.md)
Coarse-grained hierarchical method using bounding volumes to reduce search space.

### [Tessellation-based Voxelization](5430-tessellation-based-voxelization.md)
GPU-accelerated method using hardware tessellation for adaptive density.

### [Fake Voxelization via Triangle-Cube Intersection](6951-fake-voxelization-via-triangle-cube-intersection.md)
Surface-only technique using template-based intersection tests for topological separability.

## Operating characteristics
*   **Time Complexity:** $O(N_v \cdot N_t)$ where $N_v$ is the number of voxels and $N_t$ is the number of triangles, as each voxel potentially tests all triangles [S2, unofficial]. This is inefficient for high-resolution grids or complex meshes.
*   **Memory Complexity:** $O(N_v)$ for the voxel grid [S2, unofficial].
*   **Performance:** The brute-force nature makes it slow for large scenes. Parallelization (e.g., using tasks/threads) can mitigate this, as voxel tests are independent [S2, unofficial].
*   **Strengths:** Simple to implement; produces watertight solid models; geometrically accurate intersection tests [S1, S2, unofficial].
*   **Limitations:** Poor scalability with mesh complexity; grid resolution is fixed and uniform; precision issues may arise from floating-point rounding in bounds calculation [S2, unofficial].
*   **When to use:** Small to medium meshes where simplicity and watertightness are prioritized over performance [S1, unofficial].
*   **When not to use:** Large-scale scenes, real-time applications with strict performance constraints, or meshes with extremely high triangle counts [S1, S2, unofficial].

## Background
The method was described by David Rosen in a 2008 Game Developer blog post as a two-step process for converting triangle meshes to voxel models for bone attachment in games [S1, unofficial]. An implementation in Unity using C# and the Separating Axis Theorem was published by @fukaken5050 in 2017, demonstrating the AABB-Triangle intersection logic and parallel processing [S2, unofficial].

## Sources
[S1] Triangle Mesh Voxelization, https://www.gamedeveloper.com/programming/triangle-mesh-voxelization, fetched 2026-10-26 (unofficial)
[S2] MeshからVoxel生成, https://qiita.com/fukaken5050/items/cba76aa593372dc135b7, fetched 2026-10-27 (unofficial)
