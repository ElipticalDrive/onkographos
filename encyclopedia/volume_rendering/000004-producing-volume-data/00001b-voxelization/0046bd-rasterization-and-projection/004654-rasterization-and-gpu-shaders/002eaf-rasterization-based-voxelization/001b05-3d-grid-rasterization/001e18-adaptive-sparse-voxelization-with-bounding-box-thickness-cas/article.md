# Adaptive Sparse Voxelization with Bounding Box Thickness Cases

Also called: Out-of-Core SVO Builder Voxelization, triangle rasterization via bounding box cases

## Overview
This method rasterizes triangle meshes into a Morton-ordered sparse voxel grid by classifying each triangle into one of nine cases based on the thickness of its axis-aligned bounding box (AABB) in grid coordinates [S1, unofficial][S2, unofficial]. It solves the problem of efficiently determining intersected voxels while minimizing redundant depth tests, setting itself apart from general 3D grid rasterization by explicitly handling "thin" (1-voxel thick) triangles via 2D projection and "thick" triangles via 3D plane and projection tests [S1, unofficial][S2, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:** A list of triangles (vertices $v_0, v_1, v_2$ and optional colors), a partition defined by Morton code range `[morton_start, morton_end)`, a voxel physical size `unitlength`, and a sparseness limit `sparseness_limit`.
- **Outputs:** A `voxels` array of size `morton_end - morton_start` and an optional `voxel_data` vector. In binary mode, `voxels` stores `EMPTY_VOXEL` or `FULL_VOXEL`. In non-binary mode, `voxels` stores indices into `voxel_data`, which contains normals and colors.

**Data Structures:**
- `VoxelData`: Struct containing `normal` (vec3) and `color` (vec3). Used only in non-binary mode.
- `voxels` array: Flat array indexed by `index - morton_start`.
  - Binary mode: `uint8_t` or `bool`. Values: `EMPTY_VOXEL` (0) or `FULL_VOXEL` (1).
  - Non-binary mode: `size_t`. Values: Index into `voxel_data` or `EMPTY_VOXEL`.
- `p_bbox_grid`: Partition bounding box in grid coordinates, derived from `morton_start` and `morton_end`.

**Parameters:**
- `unitlength`: Physical size of one voxel side. Default varies by scene; used to convert grid to world coordinates via `unit_div = 1.0f / unitlength` [S1, unofficial][S2, unofficial].
- `sparseness_limit`: Float parameter (default 0.10) used to estimate max data side-array size: `max_bytes_data = ((morton_end - morton_start) * sizeof(char)) * sparseness_limit` [S2, unofficial].

**Steps:**

1. **Initialize Partition:** Iterate `i` from `0` to `morton_end - morton_start - 1`. Set `voxels[i] = EMPTY_VOXEL` [S1, unofficial].

2. **Compute Partition Bounds:** Decode `morton_start` and `morton_end - 1` to get `p_bbox_grid.min` and `p_bbox_grid.max` [S1, unofficial].

3. **Process Each Triangle:**
   a. **Compute Bounding Box:** Calculate world-space AABB `t_bbox_world`. Convert to grid coordinates `t_bbox_grid` by multiplying min/max components by `unit_div` and casting to `int` [S1, unofficial][S2, unofficial].
   b. **Clamp Bounds:** Clamp `t_bbox_grid.min` and `t_bbox_grid.max` to `p_bbox_grid.min` and `p_bbox_grid.max` [S1, unofficial].
   c. **Determine Thickness:** Compute `one_thick` vector where `one_thick[axis] = 1` if `t_bbox_grid.min[axis] == t_bbox_grid.max[axis]`, else `0` [S1, unofficial].
   d. **Compute Geometry:** Calculate edges $e_0 = v_1 - v_0$, $e_1 = v_2 - v_1$, $e_2 = v_0 - v_2$. Compute normal $n = \text{normalize}(e_0 \times e_1)$ [S1, unofficial].

   e. **Case Classification & Rasterization:**

      - **Case 1-3 (1D Bounding Box):** If `one_thick.sum() >= 2` (thin in at least 2 axes):
        Iterate all voxels in `t_bbox_grid`. For each voxel, compute `index = mortonEncode_LUT(z, y, x)`. If `voxels[index - morton_start] == EMPTY_VOXEL`, mark as filled (`FULL_VOXEL` or store index) and increment `nfilled` [S1, unofficial]. No geometric overlap test is needed [S2, unofficial].

      - **Case 4-6 (2D Bounding Box):** If `one_thick.sum() == 1`:
        Identify the thin axis. Perform a 2D projection test on the plane perpendicular to the thin axis. Iterate voxels in the remaining 2D grid bounds.
        Compute edge normals for the projection plane. If the triangle's normal component in the thin axis is negative, negate all edge normal vectors [S1, unofficial].
        Compute edge distance $d_e = (-1.0f \cdot (n_e \cdot v_{proj})) + \max(0.0f, \text{unitlength} \cdot n_e[0]) + \max(0.0f, \text{unitlength} \cdot n_e[1])$ [S2, unofficial].
        For each voxel center $p_{proj}$, check if $(n_e \cdot p_{proj}) + d_e < 0.0f$ for all 3 edges. If true for all edges, mark voxel as filled [S2, unofficial].

      - **Case 7-9 (3D Bounding Box):** If `one_thick.sum() == 0`:
        Compute critical point $c$ where $c[\text{axis}] = \text{unitlength}$ if $n[\text{axis}] > 0$, else $0$ [S1, unofficial].
        Compute $d_1 = n \cdot (c - v_0)$ and $d_2 = n \cdot ((\delta_p - c) - v_0)$ where $\delta_p = (1, 1, 1) \cdot \text{unitlength}$ [S1, unofficial].
        Perform 3D plane test: if $(n \cdot p + d_1) \cdot (n \cdot p + d_2) > 0.0f$, the voxel is entirely on one side of the triangle plane; skip it [S2, unofficial].
        Perform 2D projection tests on all three planes (XY, YZ, XZ). For each plane, compute edge normals and distances, negating normals if the corresponding normal component is negative. If the voxel center passes all 3 edge tests for a specific plane, it is inside the triangle's projection. All three planes must pass their respective tests [S2, unofficial].
        If a voxel passes the plane test and all three projection tests, mark it as filled. In non-binary mode, compute barycentric coordinates and interpolate vertex colors to store in `voxel_data` [S2, unofficial].

**Pitfalls:**
- **Duplicate Voxel Marking:** Always check if `voxels[index - morton_start]` is already `EMPTY_VOXEL` before processing to prevent double-counting and redundant writes [S1, unofficial][S2, unofficial].
- **Normal Orientation:** Edge normals in projection planes must be flipped if the triangle's main normal component in the perpendicular axis is negative to ensure correct culling [S1, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxelize_partition(triangles, morton_start, morton_end, unitlength, sparseness_limit, binary_mode=True):
    # 1. Initialize voxel array
    num_voxels = morton_end - morton_start
    voxels = [EMPTY_VOXEL] * num_voxels
    voxel_data = [] if not binary_mode else None
    
    # 2. Compute partition bounds
    p_bbox_grid_min = morton_decode(morton_start)
    p_bbox_grid_max = morton_decode(morton_end - 1)
    
    unit_div = 1.0 / unitlength
    
    for tri in triangles:
        # 3. Compute triangle AABB in grid coords
        t_bbox_world_min = min(tri.v0, tri.v1, tri.v2)
        t_bbox_world_max = max(tri.v0, tri.v1, tri.v2)
        
        t_bbox_grid_min = int(t_bbox_world_min * unit_div)
        t_bbox_grid_max = int(t_bbox_world_max * unit_div)
        
        # Clamp to partition bounds
        t_bbox_grid_min = clamp(t_bbox_grid_min, p_bbox_grid_min)
        t_bbox_grid_max = clamp(t_bbox_grid_max, p_bbox_grid_max)
        
        # 4. Determine thickness
        one_thick = [0, 0, 0]
        for axis in range(3):
            if t_bbox_grid_min[axis] == t_bbox_grid_max[axis]:
                one_thick[axis] = 1
        
        # 5. Compute geometry
        e0 = tri.v1 - tri.v0
        e1 = tri.v2 - tri.v1
        e2 = tri.v0 - tri.v2
        n = normalize(cross(e0, e1))
        
        # 6. Case Classification
        thickness_sum = sum(one_thick)
        
        if thickness_sum >= 2:
            # Case 1-3: 1D Bounding Box
            for z in range(t_bbox_grid_min[2], t_bbox_grid_max[2] + 1):
                for y in range(t_bbox_grid_min[1], t_bbox_grid_max[1] + 1):
                    for x in range(t_bbox_grid_min[0], t_bbox_grid_max[0] + 1):
                        index = morton_encode_LUT(z, y, x)
                        offset = index - morton_start
                        if voxels[offset] == EMPTY_VOXEL:
                            voxels[offset] = FULL_VOXEL if binary_mode else len(voxel_data)
                            if not binary_mode:
                                voxel_data.append(VoxelData(normal=n, color=tri.average_color()))
                                
        elif thickness_sum == 1:
            # Case 4-6: 2D Bounding Box
            thin_axis = -1
            for axis in range(3):
                if one_thick[axis] == 1:
                    thin_axis = axis
                    break
            
            # Determine projection plane axes
            proj_axes = [a for a in range(3) if a != thin_axis]
            
            # Compute edge normals for projection plane
            # Map 3D edge vectors to 2D projection plane
            # Note: Logic simplified for clarity; actual implementation requires careful axis mapping
            edge_normals_2d = []
            for edge in [e0, e1, e2]:
                # Project edge to plane perpendicular to thin_axis
                # Compute 2D normal in the projection plane
                # If n[thin_axis] < 0, negate normals
                n_2d = compute_2d_edge_normal(edge, thin_axis)
                if n[thin_axis] < 0.0:
                    n_2d = -n_2d
                edge_normals_2d.append(n_2d)
            
            # Compute edge distances
            edge_distances = []
            for i, edge in enumerate([e0, e1, e2]):
                v_proj = project_to_plane(tri.v0, thin_axis)
                n_e = edge_normals_2d[i]
                d_e = (-1.0 * dot(n_e, v_proj)) + max(0.0, unitlength * n_e[0]) + max(0.0, unitlength * n_e[1])
                edge_distances.append(d_e)
            
            # Iterate voxels in 2D bounds
            for z in range(t_bbox_grid_min[2], t_bbox_grid_max[2] + 1):
                for y in range(t_bbox_grid_min[1], t_bbox_grid_max[1] + 1):
                    for x in range(t_bbox_grid_min[0], t_bbox_grid_max[0] + 1):
                        # Skip if not in 2D bounds (already handled by loop ranges if correctly set)
                        # Check projection
                        p_proj = project_to_plane(vec3(x, y, z) * unitlength, thin_axis)
                        inside = True
                        for i in range(3):
                            if (dot(edge_normals_2d[i], p_proj) + edge_distances[i]) >= 0.0:
                                inside = False
                                break
                        if inside:
                            index = morton_encode_LUT(z, y, x)
                            offset = index - morton_start
                            if voxels[offset] == EMPTY_VOXEL:
                                voxels[offset] = FULL_VOXEL if binary_mode else len(voxel_data)
                                if not binary_mode:
                                    voxel_data.append(VoxelData(normal=n, color=tri.average_color()))
                                    
        else:
            # Case 7-9: 3D Bounding Box
            # Compute critical point
            c = vec3(0.0)
            for axis in range(3):
                if n[axis] > 0.0:
                    c[axis] = unitlength
            
            delta_p = vec3(unitlength, unitlength, unitlength)
            d1 = dot(n, c - tri.v0)
            d2 = dot(n, (delta_p - c) - tri.v0)
            
            # Precompute projection tests for all 3 planes
            proj_tests = []
            for plane in ['XY', 'YZ', 'ZX']:
                thin_axis_proj = plane[0] # Simplified; actual logic depends on plane definition
                # Compute edge normals and distances for this plane
                # Similar to 2D case but for the specific plane
                # Negate normals if corresponding normal component is negative
                edge_normals_2d = []
                edge_distances = []
                for edge in [e0, e1, e2]:
                    n_2d = compute_2d_edge_normal_for_plane(edge, plane)
                    if n[plane[0]] < 0.0: # Check normal component perpendicular to plane
                        n_2d = -n_2d
                    edge_normals_2d.append(n_2d)
                    
                    v_proj = project_to_plane(tri.v0, plane)
                    n_e = n_2d
                    d_e = (-1.0 * dot(n_e, v_proj)) + max(0.0, unitlength * n_e[0]) + max(0.0, unitlength * n_e[1])
                    edge_distances.append(d_e)
                proj_tests.append((edge_normals_2d, edge_distances))
            
            for z in range(t_bbox_grid_min[2], t_bbox_grid_max[2] + 1):
                for y in range(t_bbox_grid_min[1], t_bbox_grid_max[1] + 1):
                    for x in range(t_bbox_grid_min[0], t_bbox_grid_max[0] + 1):
                        p = vec3(x, y, z) * unitlength
                        nDOTp = dot(n, p)
                        
                        # Plane test
                        if (nDOTp + d1) * (nDOTp + d2) > 0.0:
                            continue
                        
                        # Projection tests
                        inside = True
                        for plane_idx, (edge_normals_2d, edge_distances) in enumerate(proj_tests):
                            p_proj = project_to_plane(p, ['XY', 'YZ', 'ZX'][plane_idx])
                            for i in range(3):
                                if (dot(edge_normals_2d[i], p_proj) + edge_distances[i]) >= 0.0:
                                    inside = False
                                    break
                            if not inside:
                                break
                        
                        if inside:
                            index = morton_encode_LUT(z, y, x)
                            offset = index - morton_start
                            if voxels[offset] == EMPTY_VOXEL:
                                voxels[offset] = FULL_VOXEL if binary_mode else len(voxel_data)
                                if not binary_mode:
                                    voxel_data.append(VoxelData(normal=n, color=tri.average_color()))
```

## Subtopics

### [NGLE Triangle Voxelization](1687-ngle-triangle-voxelization.md)
NGLE Triangle Voxelization is a rasterization-based algorithm for converting triangle meshes into a sparse voxel data structure (specifically a Sparse Voxel DAG or Octree) by classifying triangles into nine cases based on the thickness of their axis-aligned bounding box (AABB) in grid coordinates. It solves the problem of efficiently determining which voxels are intersected by a triangle while minimizing redundant depth tests, setting itself apart from general 3D grid rasterization by explicitly handling "thin" (1-voxel thick) triangles via 2D projection and "thick" triangles via 3D plane and projection tests.

## Operating characteristics
- **Time Complexity:** $O(N \cdot V)$, where $N$ is the number of triangles and $V$ is the number of voxels in the triangle's bounding box. The case-based optimization reduces $V$ for thin triangles.
- **Memory Complexity:** $O(M)$, where $M$ is the number of voxels in the partition (`morton_end - morton_start`). Binary mode uses 1 byte per voxel; non-binary mode uses `sizeof(size_t)` per voxel plus `VoxelData` storage.
- **Performance:** Optimized for out-of-core processing by partitioning the grid into Morton-ordered chunks [S1, unofficial].
- **Strengths:** Efficient handling of thin triangles; sparse storage reduces memory usage for large scenes.
- **Limitations:** Performance degrades for triangles with large bounding boxes covering many voxels. Precision issues may arise with floating-point to integer conversions for grid coordinates.
- **When to Use:** Suitable for building Sparse Voxel Octrees (SVOs) or DAGs from triangle meshes, especially when memory is constrained or the scene has many thin features.
- **When Not to Use:** Not ideal for real-time rendering where precomputed dense grids are preferred, or for scenes with extremely large, thick triangles covering most of the grid.

## Background
The method originates from the "Out-Of-Core SVO Builder" by Jeroen Baert (Forceflow) [S1, unofficial] and is also referenced in the "High Resolution Sparse Voxel DAGs" project [S2, unofficial]. It is designed for building large-scale sparse voxel structures from triangle meshes, leveraging Morton ordering for efficient spatial indexing and out-of-core processing capabilities.

## Sources
[S1] Forceflow/ooc_svo_builder, https://github.com/Forceflow/ooc_svo_builder, fetched 2026-10-26 (unofficial)
[S2] leonzheng2/INF584-High_Resolution_Sparse_Voxel_DAGs, https://github.com/leonzheng2/INF584-High_Resolution_Sparse_Voxel_DAGs, fetched 2026-10-26 (unofficial)
