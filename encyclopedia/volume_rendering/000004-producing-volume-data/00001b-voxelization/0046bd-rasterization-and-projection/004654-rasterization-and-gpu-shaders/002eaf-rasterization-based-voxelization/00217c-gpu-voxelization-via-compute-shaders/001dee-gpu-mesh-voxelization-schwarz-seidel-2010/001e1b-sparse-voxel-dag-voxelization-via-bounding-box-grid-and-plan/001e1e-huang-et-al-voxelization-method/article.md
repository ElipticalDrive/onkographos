# Huang et al. Voxelization Method

Also called: Huang voxelization

## Overview
A voxelization algorithm that tests grid cells for overlap with a triangle using spheres at vertices, cylinders on edges, and a plane test for the triangle interior [S1, S2, unofficial]. It is adapted for morton-code-based subgrids, serving as the core rasterization step within the Sparse Voxel DAG Voxelization via Bounding Box Grid and Plane Projections framework [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A list of triangles (vertices $v_0, v_1, v_2$), a voxel side length `unitlength`, and a Morton code range `[morton_start, morton_end)` defining the current subgrid partition.
*   **Outputs:** A `voxels` array of size `morton_end - morton_start`, where each entry indicates occupancy (and optionally payload data).

**Data Structures:**
*   `voxels`: An array of size $N = \text{morton\_end} - \text{morton\_start}$.
    *   If `BINARY_VOXELIZATION` is defined: stores `bool` or `char` (occupancy).
    *   If undefined: stores `size_t` indices into a `voxel_data` vector, or a sentinel `EMPTY_VOXEL`.
*   `voxel_data` (non-binary mode only): A vector of `VoxelData` structs containing the Morton index, triangle normal, and averaged vertex colors.

**Parameters:**
*   `unitlength`: Voxel side length in world units [S2, unofficial].
*   `morton_start`, `morton_end`: 64-bit Morton codes defining the subgrid range [S2, unofficial].
*   `BINARY_VOXELIZATION`: Compile-time flag determining storage mode [S1, S2, unofficial].

**Steps:**

1.  **Initialization:**
    *   Compute `unit_div = 1.0f / unitlength` and `radius = unitlength / 2.0f` [S1, S2, unofficial].
    *   Decode `morton_start` and `morton_end - 1` to determine the partition's bounding box in grid coordinates: `p_bbox_grid.min` and `p_bbox_grid.max` [S1, S2, unofficial].
    *   Initialize the `voxels` array: for $i$ from $0$ to $N-1$, set `voxels[i] = EMPTY_VOXEL` (or `false` in binary mode) [S1, S2, unofficial].

2.  **Triangle Processing Loop:**
    For each triangle $t$:
    *   Compute the triangle's world bounding box $t\_bbox\_world$ [S1, S2, unofficial].
    *   Convert to grid coordinates: $t\_bbox\_grid = \text{floor}(t\_bbox\_world \times \text{unit\_div})$ [S1, S2, unofficial].
    *   Clamp $t\_bbox\_grid$ to the partition bounding box $p\_bbox\_grid$ to ensure iteration stays within the current subgrid [S1, S2, unofficial].

3.  **Voxel Iteration:**
    Iterate through all grid cells $(x, y, z)$ within the clamped $t\_bbox\_grid$:
    *   Compute the Morton index: $idx = \text{morton3D\_64\_encode}(z, y, x)$ [S1, S2, unofficial].
    *   Compute the local array index: $array\_idx = idx - \text{morton\_start}$ [S1, unofficial].
    *   If `voxels[array_idx]` is already filled, skip to the next cell [S1, S2, unofficial].
    *   Compute the voxel center: $\mathbf{p} = \text{vec3}((x + 0.5f) \cdot \text{unitlength}, (y + 0.5f) \cdot \text{unitlength}, (z + 0.5f) \cdot \text{unitlength})$ [S1, S2, unofficial].

4.  **Overlap Tests (Three-Stage):**
    Construct test primitives:
    *   Three spheres centered at $v_0, v_1, v_2$ with radius `radius` [S1, S2, unofficial].
    *   Three cylinders connecting edges $(v_0-v_1), (v_1-v_2), (v_2-v_0)$ with radius `radius` [S1, S2, unofficial].
    *   Plane $S$ defined by the triangle vertices [S1, S2, unofficial].

    *   **Test 1 (Vertex Spheres):** If $\mathbf{p}$ is inside any of the three vertex spheres, mark `voxels[array_idx]` as filled. If in non-binary mode, add payload data. Increment filled counter and continue to next cell [S1, S2, unofficial].

    *   **Test 2 (Edge Cylinders):** If $\mathbf{p}$ is inside any of the three edge cylinders, mark `voxels[array_idx]` as filled. Add payload if non-binary. Increment filled counter and continue to next cell [S1, S2, unofficial].

    *   **Test 3 (Plane & Barycentric):**
        *   Compute $\text{halfunit} = \text{unitlength} / 2.0f$ [S1, S2, unofficial].
        *   Calculate $b_0, b_1, b_2$ as the absolute dot products of the half-unit vectors along the X, Y, and Z axes with the plane normal $\mathbf{n}$:
            $$b_i = | \mathbf{e}_i \cdot \mathbf{n} | \cdot \text{halfunit}$$
            where $\mathbf{e}_i$ are the unit vectors for axes X, Y, Z [S1, unofficial].
        *   Compute $\cos\beta = \frac{\max(b_0, b_1, b_2)}{\|\text{halfunit\_vector}\| \cdot \|\mathbf{n}\|}$ [S1, unofficial]. Note: $\|\text{halfunit\_vector}\|$ refers to the magnitude of the vector formed by the half-unit components used in the max calculation, effectively normalizing the projection error bound [S1, unofficial].
        *   Compute offset $t_c = \frac{\text{unitlength}}{2.0f} \cdot \cos\beta$ [S1, unofficial].
        *   Construct two parallel planes offset by $t_c$ from the triangle plane $S$: Plane($\mathbf{n}, D + t_c$) and Plane($\mathbf{n}, D - t_c$), where $D$ is the plane constant derived from $t$ [S1, unofficial].
        *   Check if $\mathbf{p}$ lies between these two parallel planes. If not, skip the cell [S1, unofficial].
        *   If the plane test passes, compute barycentric-like signs $s_1, s_2, s_3$:
            $$s_1 = (\mathbf{n} \times \mathbf{e}_1) \cdot (\mathbf{p} - \mathbf{v}_0)$$
            $$s_2 = (\mathbf{n} \times \mathbf{e}_2) \cdot (\mathbf{p} - \mathbf{v}_1)$$
            $$s_3 = (\mathbf{n} \times \mathbf{e}_3) \cdot (\mathbf{p} - \mathbf{v}_2)$$
            where $\mathbf{n}$ is the triangle normal, $\mathbf{e}_1 = v_1-v_0$, $\mathbf{e}_2 = v_2-v_1$, $\mathbf{e}_3 = v_0-v_2$ are the edge vectors, $\mathbf{p}$ is the voxel center, and $\mathbf{v}_i$ are the vertices [S1, S2, unofficial].
        *   If the signs are consistent (i.e., $(s_1 \le 0) == (s_2 \le 0)$ and $(s_2 \le 0) == (s_3 \le 0)$), mark `voxels[array_idx]` as filled. Add payload if non-binary. Increment filled counter [S1, S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxelize_huang(triangles, unitlength, morton_start, morton_end, binary_mode=True):
    """
    Voxelize triangles into a Morton-coded subgrid using Huang's method.
    
    Args:
        triangles: List of Triangle objects with v0, v1, v2 (vec3).
        unitlength: Float, side length of a voxel.
        morton_start: Int, start of Morton range.
        morton_end: Int, end of Morton range.
        binary_mode: Bool, if True store occupancy only, else store indices.
        
    Returns:
        voxels: List of values (bool/char or size_t).
        voxel_data: List of VoxelData (only if not binary_mode).
    """
    n_voxels = morton_end - morton_start
    voxels = [EMPTY_VOXEL] * n_voxels
    voxel_data = [] if not binary_mode else None
    
    unit_div = 1.0 / unitlength
    radius = unitlength / 2.0
    halfunit = unitlength / 2.0
    
    # Decode bounds to get partition grid bbox
    p_min = morton3D_64_decode(morton_start)
    p_max = morton3D_64_decode(morton_end - 1)
    
    for t in triangles:
        # Compute world bbox and convert to grid
        t_bbox_world = compute_world_bbox(t.v0, t.v1, t.v2)
        t_bbox_grid_min = floor(t_bbox_world.min * unit_div)
        t_bbox_grid_max = floor(t_bbox_world.max * unit_div)
        
        # Clamp to partition
        t_bbox_grid_min = max(t_bbox_grid_min, p_min)
        t_bbox_grid_max = min(t_bbox_grid_max, p_max)
        
        # Construct primitives
        spheres = [Sphere(t.v0, radius), Sphere(t.v1, radius), Sphere(t.v2, radius)]
        cylinders = [Cylinder(t.v0, t.v1, radius), Cylinder(t.v1, t.v2, radius), Cylinder(t.v2, t.v0, radius)]
        plane = Plane.from_points(t.v0, t.v1, t.v2)
        n = plane.normal
        D = plane.d
        
        # Precompute edge vectors and cross products for plane test
        e1 = t.v1 - t.v0
        e2 = t.v2 - t.v1
        e3 = t.v0 - t.v2
        
        # Plane test bounds
        # b0, b1, b2 are abs(dot(half_unit_axis, n)) * halfunit
        # half_unit_axis are (1,0,0), (0,1,0), (0,0,1) scaled by halfunit
        b0 = abs(n.x) * halfunit
        b1 = abs(n.y) * halfunit
        b2 = abs(n.z) * halfunit
        
        max_b = max(b0, b1, b2)
        # length(halfunit_vector) is halfunit * sqrt(1) = halfunit if we consider individual axes
        # length(n) is norm(n)
        # cosbeta = max_b / (halfunit * norm(n))
        norm_n = length(n)
        if norm_n == 0:
            continue # Degenerate triangle
            
        cos_beta = max_b / (halfunit * norm_n)
        tc = halfunit * cos_beta
        
        # Parallel planes offset by tc
        # Check if point is between D-tc and D+tc
        # dot(p, n) + D is the signed distance * norm(n) ? 
        # Plane eq: dot(p, n) + D = 0. 
        # The sources define planes as Plane(S.normal, S.D + tc). 
        # Assuming standard plane representation where D is the constant term.
        
        for z in range(t_bbox_grid_min.z, t_bbox_grid_max.z + 1):
            for y in range(t_bbox_grid_min.y, t_bbox_grid_max.y + 1):
                for x in range(t_bbox_grid_min.x, t_bbox_grid_max.x + 1):
                    idx = morton3D_64_encode(z, y, x)
                    array_idx = idx - morton_start
                    
                    if voxels[array_idx] != EMPTY_VOXEL:
                        continue
                    
                    p = vec3((x + 0.5) * unitlength, (y + 0.5) * unitlength, (z + 0.5) * unitlength)
                    
                    filled = False
                    
                    # Test 1: Spheres
                    for s in spheres:
                        if s.contains(p):
                            filled = True
                            break
                    
                    if not filled:
                        # Test 2: Cylinders
                        for c in cylinders:
                            if c.contains(p):
                                filled = True
                                break
                    
                    if not filled:
                        # Test 3: Plane
                        # Check if p is between parallel planes
                        # dist = dot(p, n) + D
                        # We need dist to be within [-tc, tc] relative to the plane?
                        # The source says: "Check if middle_point is between parallel planes S.normal, S.D + tc and S.normal, S.D - tc"
                        # Plane equation: dot(p, n) + D' = 0.
                        # Plane 1: dot(p, n) + (D + tc) = 0 => dot(p, n) = -D - tc
                        # Plane 2: dot(p, n) + (D - tc) = 0 => dot(p, n) = -D + tc
                        # So we check if dot(p, n) is between -D-tc and -D+tc
                        # Or equivalently, if |dot(p, n) + D| <= tc
                        
                        dist = dot(p, n) + D
                        if abs(dist) <= tc:
                            # Barycentric check
                            s1 = dot(cross(n, e1), p - t.v0)
                            s2 = dot(cross(n, e2), p - t.v1)
                            s3 = dot(cross(n, e3), p - t.v2)
                            
                            if (s1 <= 0) == (s2 <= 0) and (s2 <= 0) == (s3 <= 0):
                                filled = True
                    
                    if filled:
                        voxels[array_idx] = FILLED_VOXEL
                        if not binary_mode:
                            # Add payload
                            data = VoxelData(morton=idx, normal=n, color=average_colors(t))
                            voxel_data.append(data)
                            voxels[array_idx] = len(voxel_data) - 1
                            
    return voxels, voxel_data
```

## Operating characteristics
*   **Time Complexity:** Proportional to the number of voxels in the triangle's grid bounding box, multiplied by the cost of the three overlap tests. The tests are constant time but involve vector operations.
*   **Memory Complexity:** Linear in the number of voxels in the subgrid ($N = \text{morton\_end} - \text{morton\_start}$).
*   **Strengths:** Robust overlap detection that handles thin triangles and grazing angles better than simple rasterization. The three-stage test (Sphere -> Cylinder -> Plane) allows early exits for most voxels, optimizing performance.
*   **Limitations:** More computationally expensive per voxel than simple rasterization due to the geometric tests. Requires careful handling of floating-point precision in the plane tests.
*   **When to use:** When building high-quality Sparse Voxel Octrees/DAGs where accurate geometry representation is critical, especially for thin or complex meshes.

## Background
The method is described in the context of Out-Of-Core Sparse Voxel Octree (SVO) building [S1, unofficial]. It is part of the pipeline that converts triangle meshes into voxelized structures for rendering and processing. The implementation is found in the `ooc_svo_builder` repository [S1, unofficial] and referenced in work on High Resolution Sparse Voxel DAGs [S2, unofficial].

## Sources
[S1] Forceflow/ooc_svo_builder, https://github.com/Forceflow/ooc_svo_builder, fetched 2026-10-26 (unofficial)
[S2] leonzheng2/INF584-High_Resolution_Sparse_Voxel_DAGs, https://github.com/leonzheng2/INF584-High_Resolution_Sparse_Voxel_DAGs, fetched 2026-10-26 (unofficial)
