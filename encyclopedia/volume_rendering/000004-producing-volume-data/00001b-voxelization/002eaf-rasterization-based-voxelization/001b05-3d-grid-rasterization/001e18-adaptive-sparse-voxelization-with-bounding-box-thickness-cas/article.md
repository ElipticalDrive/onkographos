# Adaptive Sparse Voxelization with Bounding Box Thickness Cases


## Overview
This method rasterizes a triangle mesh into a sparse voxel data structure (specifically a Sparse Voxel DAG) by classifying each triangle's axis-aligned bounding box (AABB) into nine cases based on its thickness in grid coordinates [S1, unofficial]. It optimizes overlap testing by using simple iteration for 1D-thick boxes, 2D projection tests for 2D-thick boxes, and combined 3D plane and 2D projection tests for 3D-thick boxes [S1, unofficial]. This approach distinguishes itself from general 3D clipping methods by reducing computational complexity for thin primitives, which are common in high-resolution scenes [S1, unofficial].

## How it works
**Inputs:**
- Triangle mesh vertices in world space.
- `unitlength`: Physical size of one voxel side [S1, unofficial].
- `sparseness_limit`: Float parameter for estimating data side-array size [S1, unofficial].
- `p_bbox_grid`: Global bounding box of the partition in grid coordinates [S1, unofficial].

**Outputs:**
- A sparse voxel array indexed by Morton code offset.
- In binary mode: `char` values (`EMPTY_VOXEL` or `FULL_VOXEL`/1) [S1, unofficial].
- In non-binary mode: `size_t` indices into a `voxel_data` vector containing `VoxelData` (normal `vec3`, color `vec3`) [S1, unofficial].

**Data Structures:**
- `VoxelData`: Struct with fields `normal` (vec3) and `color` (vec3) [S1, unofficial].
- `voxels` array: Sparse array. Index is `morton_index - morton_start`. Stores `char` (binary) or `size_t` (non-binary) [S1, unofficial].
- `voxel_data` vector: Only in non-binary mode. Stores `VoxelData` objects [S1, unofficial].

**Steps:**

1. **Compute Bounding Box:**
   Calculate the triangle's world-space AABB (`t_bbox_world`). Convert to grid coordinates (`t_bbox_grid`) by multiplying vertex coordinates by `unit_div = 1.0f / unitlength` and casting to `int` [S1, unofficial]. Clamp `t_bbox_grid` to the partition's global bounds `p_bbox_grid` [S1, unofficial].

2. **Determine Thickness Case:**
   Compute the `one_thick` vector, where each component is `1` if the corresponding min and max grid coordinates are equal (thickness is 1 voxel), otherwise `0` [S1, unofficial].
   - If `one_thick.sum() >= 2`: Case 1-3 (1D Bounding Box) [S1, unofficial].
   - If `one_thick.sum() == 1`: Case 4-6 (2D Bounding Box) [S1, unofficial].
   - If `one_thick.sum() == 0`: Case 7-9 (3D Bounding Box) [S1, unofficial].

3. **Process 1D Bounding Boxes (Cases 1-3):**
   Iterate all voxels within `t_bbox_grid`. For each voxel:
   - Compute Morton index [S1, unofficial].
   - Check if the voxel is already marked `FULL_VOXEL` (or non-empty). If so, skip to avoid duplicate work [S1, unofficial].
   - Mark as filled. In non-binary mode, add to `voxel_data` [S1, unofficial].
   - No geometric overlap test is needed [S1, unofficial].

4. **Process 2D Bounding Boxes (Cases 4-6):**
   Identify the thin axis (where `one_thick` is 1) [S1, unofficial]. Perform a 2D projection test on the plane perpendicular to the thin axis [S1, unofficial].
   - Iterate voxels in the remaining 2D grid bounds [S1, unofficial].
   - Compute edge normals (`n_e`) and distances (`d_e`) for the projection plane [S1, unofficial].
   - For each voxel center projected onto the plane (`p_proj`), check if it passes all 3 edge tests [S1, unofficial].
   - Edge test formula: `(n_e DOT p_proj) + d_e < 0.0f`. If true, the voxel is outside and skipped [S1, unofficial].
   - Edge distance calculation: `d_e = (-1.0f * (n_e DOT vertex_proj)) + max(0.0f, unitlength*n_e[0]) + max(0.0f, unitlength*n_e[1])` [S1, unofficial].
   - If the voxel passes all 3 tests, mark as filled. In non-binary mode, compute barycentric coordinates and interpolate vertex colors [S1, unofficial].

5. **Process 3D Bounding Boxes (Cases 7-9):**
   Compute the 3D triangle normal `n` [S1, unofficial].
   - Determine the dominant axis (component with largest absolute value) [S1, unofficial].
   - Compute a critical point `c` where `c[axis] = unitlength` if `n[axis] > 0` else `0` [S1, unofficial].
   - Compute plane test distances `d1` and `d2` using `c` [S1, unofficial].
   - Perform 3D plane test: `(nDOTp + d1) * (nDOTp + d2) > 0.0f`, where `nDOTp = dot(n, voxel_center)` [S1, unofficial]. If true, the voxel is entirely on one side of the triangle plane and is skipped [S1, unofficial].
   - Perform 2D projection tests on all three planes (XY, YZ, XZ) [S1, unofficial].
   - For each plane, compute edge normals and distances. If the voxel center passes all 3 edge tests for a specific plane, it is inside the triangle's projection for that plane [S1, unofficial].
   - All three planes must pass their respective tests [S1, unofficial].
   - If the voxel passes the plane test and all three projection tests, mark as filled. In non-binary mode, compute barycentric coordinates and interpolate vertex colors [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxelize_partition3(triangles, unitlength, sparseness_limit, p_bbox_grid):
    unit_div = 1.0f / unitlength
    voxels = {} # Sparse map: morton_index -> data
    voxel_data = [] # List of VoxelData structs (normal, color)
    
    for tri in triangles:
        # 1. Compute Bounding Box
        t_bbox_world = compute_world_aabb(tri.vertices)
        t_bbox_grid = clamp_to_int(
            multiply_by(t_bbox_world, unit_div), 
            p_bbox_grid.min, p_bbox_grid.max
        )
        
        # 2. Determine Thickness Case
        one_thick = [
            1 if t_bbox_grid.min[i] == t_bbox_grid.max[i] else 0 
            for i in range(3)
        ]
        thickness_sum = sum(one_thick)
        
        if thickness_sum >= 2:
            # Case 1-3: 1D Bounding Box
            for voxel in iterate_grid(t_bbox_grid):
                morton_idx = compute_morton(voxel.center)
                if morton_idx not in voxels:
                    voxels[morton_idx] = 1 # Binary mode flag
                    # Non-binary: store index to voxel_data
        elif thickness_sum == 1:
            # Case 4-6: 2D Bounding Box
            thin_axis = one_thick.index(1)
            proj_plane = get_perpendicular_plane(thin_axis)
            
            for voxel in iterate_2d_grid(t_bbox_grid, thin_axis):
                p_proj = project(voxel.center, proj_plane)
                if passes_2d_edge_tests(p_proj, tri.vertices, unitlength, thin_axis):
                    morton_idx = compute_morton(voxel.center)
                    if morton_idx not in voxels:
                        voxels[morton_idx] = 1
                        # Non-binary: interpolate color, add to voxel_data
        else:
            # Case 7-9: 3D Bounding Box
            n = compute_triangle_normal(tri.vertices)
            dominant_axis = argmax(abs(n))
            c = compute_critical_point(n, dominant_axis, unitlength)
            d1, d2 = compute_plane_distances(n, c, tri.vertices)
            
            for voxel in iterate_grid(t_bbox_grid):
                nDOTp = dot(n, voxel.center)
                if (nDOTp + d1) * (nDOTp + d2) > 0.0f:
                    continue # Outside plane
                
                if passes_3d_projection_tests(voxel.center, tri.vertices, unitlength):
                    morton_idx = compute_morton(voxel.center)
                    if morton_idx not in voxels:
                        voxels[morton_idx] = 1
                        # Non-binary: interpolate color, add to voxel_data

    return voxels, voxel_data

def passes_2d_edge_tests(p_proj, vertices, unitlength, thin_axis):
    # Project vertices to 2D plane
    v_proj = [project(v, get_perpendicular_plane(thin_axis)) for v in vertices]
    for i in range(3):
        v1 = v_proj[i]
        v2 = v_proj[(i + 1) % 3]
        edge_vec = v2 - v1
        n_e = normalize(vec2(-edge_vec.y, edge_vec.x)) # 2D normal
        d_e = (-1.0f * dot(n_e, v1)) + max(0.0f, unitlength * n_e[0]) + max(0.0f, unitlength * n_e[1])
        if (dot(n_e, p_proj) + d_e) >= 0.0f:
            return False
    return True

def passes_3d_projection_tests(center, vertices, unitlength):
    # Check XY, YZ, XZ planes
    planes = ['xy', 'yz', 'xz']
    for plane in planes:
        if not passes_2d_edge_tests_for_plane(center, vertices, unitlength, plane):
            return False
    return True

def compute_critical_point(n, axis, unitlength):
    c = [0.0f, 0.0f, 0.0f]
    if n[axis] > 0:
        c[axis] = unitlength
    return c

def compute_plane_distances(n, c, vertices):
    # Simplified: distances based on critical point and triangle geometry
    # Exact implementation depends on specific derivation in [S1, unofficial]
    # Placeholder for specific d1, d2 calculation logic
    return 0.0f, 0.0f
```

## Subtopics

### [NGLE Triangle Voxelization](1687-ngle-triangle-voxelization.md)
NGLE Triangle Voxelization is a rasterization-based algorithm that converts a triangle mesh into a sparse voxel DAG by classifying each triangle's axis-aligned bounding box (AABB) thickness into nine cases (1D, 2D, or 3D) and applying optimized plane and projection tests to determine intersected grid cells. It distinguishes itself from general 3D clipping methods by classifying bounding box thickness into 1D, 2D, and 3D cases, using 2D projection tests for thin boxes and combined plane/projection tests for thick boxes.

## Operating characteristics
- **Time Complexity:** Depends on the number of voxels in the bounding box and the thickness case. 1D cases are O(V) where V is voxels in AABB. 2D and 3D cases involve additional geometric tests per voxel [S1, unofficial].
- **Memory Complexity:** Sparse storage using Morton codes. Memory usage depends on `sparseness_limit` and the number of filled voxels [S1, unofficial].
- **Strengths:** Optimized for thin triangles (common in meshes) by avoiding expensive 3D tests. Supports both binary and data-rich modes [S1, unofficial].
- **Limitations:** Requires careful handling of edge cases and precision. Non-binary mode increases memory and computation due to barycentric interpolation [S1, unofficial].
- **When to use:** High-resolution sparse voxelization where many triangles are thin or flat.
- **When not to use:** Dense voxelization or when memory bandwidth is severely constrained and binary mode is insufficient.

## Background
The method originates from the "High Resolution Sparse Voxel DAGs" project by leonzheng2 [S1, unofficial]. It is part of a larger system for constructing and reducing Sparse Voxel DAGs, which are used for efficient storage and rendering of high-resolution 3D scenes [S1, unofficial]. The algorithm is implemented in C++ and utilizes timing utilities (`svo_algo_timer`) to measure performance [S1, unofficial].

## Sources
[S1] leonzheng2/INF584-High_Resolution_Sparse_Voxel_DAGs, https://github.com/leonzheng2/INF584-High_Resolution_Sparse_Voxel_DAGs, fetched 2026-10-26 (unofficial)
