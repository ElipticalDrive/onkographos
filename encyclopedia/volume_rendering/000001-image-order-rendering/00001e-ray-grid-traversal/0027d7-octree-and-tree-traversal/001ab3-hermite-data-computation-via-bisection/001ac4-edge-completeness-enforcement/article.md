# Edge Completeness Enforcement

Also called: subdivision for edge completeness

## Overview
Edge Completeness Enforcement is a post-processing step in adaptive octree surface extraction that ensures all edges with sign changes possess complete 4-cell neighborhoods at the same depth, thereby eliminating geometric gaps at depth boundaries [S1, unofficial]. It differs from its parent, Hermite Data Computation via Bisection, by focusing on topological consistency of the grid structure rather than the precise computation of intersection points and normals [S1, unofficial].

## How it works
**Inputs:** An adaptive octree containing implicit function evaluations, a bounding box (`CubicBounds`), minimum and maximum depths (`min_depth`, `max_depth`), an iso-value, and the implicit function `func` [S1, unofficial].
**Outputs:** A modified octree where all surface leaves with sign changes are at uniform depth locally, ensuring watertightness [S1, unofficial].

**Data Structures:**
*   `leaf_set`: A hash set of tuples `(gx, gy, gz, depth)` representing surface leaves at `max_depth` coordinates [S1, unofficial].
*   `cells_to_subdivide`: A list of cells `(nx, ny, nz)` identified for subdivision [S1, unofficial].
*   `target_depths`: A map from cell keys `(nx/s, ny/s, nz/s, actual_depth)` to the target depth `depth` [S1, unofficial].
*   `corner_values`: An array of 8 floats storing the function value minus iso-value for each corner of a cell [S1, unofficial].
*   `EDGE_CORNERS`: A constant lookup table mapping edge index (0-11) to two corner indices (0-7) [S1, unofficial]. Edges 0-3 are parallel to X, 4-7 to Y, 8-11 to Z. Within each group, edges are indexed by the bits of the two non-axis dimensions [S1, unofficial].

**Algorithm Steps:**

1.  **Initialization:** Set `total_evals = 0` [S1, unofficial]. Define constants: `CANONICAL_EDGES = [0, 4, 8]`, `NON_AXIS_DIMS = [(1, 2), (0, 2), (0, 1)]`, and `ALL_EDGES_BY_AXIS = [[0,1,2,3], [4,5,6,7], [8,9,10,11]]` [S1, unofficial].
2.  **Collect Surface Leaves:** Recursively traverse the octree [S1, unofficial]. For every leaf cell where `data.has_sign_change()` is true, compute its grid coordinates `(gx, gy, gz)` at `max_depth` by multiplying its local coordinates by `scale = 1 << (max_depth - depth)`, and add `(gx, gy, gz, depth)` to `leaf_set` [S1, unofficial].
3.  **Identify Neighbors for Subdivision:** Initialize `cells_to_subdivide` and `seen` sets [S1, unofficial]. Iterate through each `(gx, gy, gz, depth)` in `leaf_set`:
    *   Retrieve corner values for the 8 corners of the cell from the cache, storing in `corner_values[8]` where `corner_values[i] = func(corner_i) - iso_value` [S1, unofficial].
    *   For each `axis` in `0..2` and each `edge_in_group` in `0..3`:
        *   Calculate `edge_idx = axis * 4 + edge_in_group` [S1, unofficial].
        *   Get corner indices `(c0, c1)` from `EDGE_CORNERS[edge_idx]` [S1, unofficial].
        *   Compute `val0 = corner_values[c0]` and `val1 = corner_values[c1]` [S1, unofficial].
        *   If `(val0 < 0.0) == (val1 < 0.0)`, continue (no sign change on this edge) [S1, unofficial].
        *   Calculate edge offset bits: `a1_off = edge_in_group & 1`, `a2_off = (edge_in_group >> 1) & 1` [S1, unofficial].
        *   Identify the 4 sharing cells for this edge [S1, unofficial]. Let `(a1, a2)` be the non-axis dimensions for this `axis` from `NON_AXIS_DIMS[axis]` [S1, unofficial]. Let `cell_coords = [gx, gy, gz]` [S1, unofficial].
        *   Iterate `da1` in `0..1` and `da2` in `0..1`:
            *   Calculate `na1 = cell_coords[a1] + a1_off` and `na2 = cell_coords[a2] + a2_off` [S1, unofficial].
            *   If `na1 < da1` or `na2 < da2`, skip (boundary check) [S1, unofficial].
            *   Calculate `ca1 = na1 - da1` and `ca2 = na2 - da2` [S1, unofficial].
            *   If `ca1 >= max_coord` or `ca2 >= max_coord`, skip (boundary check) [S1, unofficial].
            *   The third coordinate `other_coord` is `cell_coords[3 - a1 - a2]` [S1, unofficial].
            *   The sharing cell is `(ca1, ca2, other_coord)` [S1, unofficial]. If this is not the current cell `(gx, gy, gz)`, proceed [S1, unofficial].
        *   Check if the sharing cell exists in `leaf_set` at the same `depth` using `find_surface_neighbor_same_depth` [S1, unofficial]. If false:
            *   Determine the actual depth of the neighbor in the octree [S1, unofficial]. If the neighbor is coarser (`actual_depth < depth`) and is not a `Branch` node:
                *   Calculate `s = 1 << (depth - actual_depth)` [S1, unofficial].
                *   Form key `k = (nx/s, ny/s, nz/s, actual_depth)` where `(nx, ny, nz)` are the grid coordinates of the sharing cell [S1, unofficial].
                *   If `k` not in `seen`, add `(nx, ny, nz)` to `cells_to_subdivide` and set `target_depths[k] = depth` [S1, unofficial].
4.  **Subdivision Loop:** If `cells_to_subdivide` is empty, break [S1, unofficial]. Otherwise:
    *   **Collect New Points:** For each cell in `cells_to_subdivide`, call `collect_deep_corners` to gather grid positions and world coordinates for all corners needed from the current depth to the target depth [S1, unofficial].
    *   **Evaluate:** If `new_points` is not empty, call `bridge::batch_evaluate` to evaluate `func` at these points. Add `len(new_points)` to `total_evals`. If gradients are not provided, estimate them via finite differences using `bridge::estimate_gradients_fd`, adding `len(new_points)*6` to `total_evals`. Insert results into the cache [S1, unofficial].
    *   **Subdivide:** For each cell in `cells_to_subdivide`, call `subdivide_deep` to recursively create `Branch` nodes and leaves down to the `target_depth` [S1, unofficial].
    *   Repeat from Step 3 with the updated octree and `leaf_set` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ensure_edge_completeness(octree, cubic_bounds, min_depth, max_depth, iso_value, func, cache, gradients_provided=False):
    """
    Ensures all edges with sign changes in the adaptive octree have complete 4-cell neighborhoods.
    """
    total_evals = 0
    CANONICAL_EDGES = [0, 4, 8]
    NON_AXIS_DIMS = [(1, 2), (0, 2), (0, 1)]
    ALL_EDGES_BY_AXIS = [[0, 1, 2, 3], [4, 5, 6, 7], [8, 9, 10, 11]]
    EDGE_CORNERS = [
        (0, 1), (2, 3), (4, 5), (6, 7),
        (0, 2), (1, 3), (4, 6), (5, 7),
        (0, 4), (1, 5), (2, 6), (3, 7),
    ]
    max_coord = 1 << max_depth  # Grid dimension limit at max_depth
    
    while True:
        leaf_set = set()
        # Collect surface leaves by recursively traversing the octree [S1, unofficial]
        for leaf in octree.get_surface_leaves():
            if leaf.has_sign_change():
                depth = leaf.depth
                scale = 1 << (max_depth - depth)
                gx = leaf.cx * scale
                gy = leaf.cy * scale
                gz = leaf.cz * scale
                leaf_set.add((gx, gy, gz, depth))
        
        cells_to_subdivide = []
        target_depths = {}
        seen = set()
        
        for (gx, gy, gz, depth) in leaf_set:
            # Retrieve corner values from cache [S1, unofficial]
            corner_values = [0.0] * 8
            for i in range(8):
                corner_pos = cubic_bounds.corner(i)
                val = func(corner_pos) - iso_value
                corner_values[i] = val
            
            for axis in range(3):
                a1, a2 = NON_AXIS_DIMS[axis]
                for edge_in_group in range(4):
                    edge_idx = axis * 4 + edge_in_group
                    c0, c1 = EDGE_CORNERS[edge_idx]
                    
                    val0 = corner_values[c0]
                    val1 = corner_values[c1]
                    
                    if (val0 < 0.0) == (val1 < 0.0):
                        continue
                    
                    a1_off = edge_in_group & 1
                    a2_off = (edge_in_group >> 1) & 1
                    
                    cell_coords = [gx, gy, gz]
                    other_coord = cell_coords[3 - a1 - a2]
                    
                    for da1 in range(2):
                        for da2 in range(2):
                            na1 = cell_coords[a1] + a1_off
                            na2 = cell_coords[a2] + a2_off
                            
                            if na1 < da1 or na2 < da2:
                                continue
                            
                            ca1 = na1 - da1
                            ca2 = na2 - da2
                            
                            if ca1 >= max_coord or ca2 >= max_coord:
                                continue
                            
                            sharing_cell = (ca1, ca2, other_coord)
                            
                            if sharing_cell == (gx, gy, gz):
                                continue
                            
                            # Check if sharing cell exists in leaf_set at the same depth [S1, unofficial]
                            if sharing_cell not in leaf_set:
                                actual_depth = octree.get_depth(sharing_cell)
                                if actual_depth < depth and not octree.is_branch(sharing_cell):
                                    s = 1 << (depth - actual_depth)
                                    nx = sharing_cell[0] // s
                                    ny = sharing_cell[1] // s
                                    nz = sharing_cell[2] // s
                                    key = (nx, ny, nz, actual_depth)
                                    
                                    if key not in seen:
                                        seen.add(key)
                                        cells_to_subdivide.append(sharing_cell)
                                        target_depths[key] = depth
        
        if not cells_to_subdivide:
            break
        
        new_points = []
        for (nx, ny, nz) in cells_to_subdivide:
            target = target_depths[(nx, ny, nz, octree.get_depth((nx, ny, nz)))]
            # Gather grid positions and world coordinates for all corners needed from current depth to target depth [G]
            # (Implementation detail: traverse octree from (nx,ny,nz) down to target, sampling corners)
            corners = [] # Placeholder for collect_deep_corners logic
            new_points.extend(corners)
        
        if new_points:
            results = [func(pt) for pt in new_points]
            total_evals += len(new_points)
            
            if not gradients_provided:
                # Estimate gradients via finite differences [G]
                grads = estimate_gradients_fd(new_points, results)
                total_evals += len(new_points) * 6
                for i, pt in enumerate(new_points):
                    cache.insert(pt, results[i], grads[i])
            else:
                for i, pt in enumerate(new_points):
                    cache.insert(pt, results[i])
        
        for (nx, ny, nz) in cells_to_subdivide:
            target = target_depths[(nx, ny, nz, octree.get_depth((nx, ny, nz)))]
            # Recursively create Branch nodes and leaves down to target depth [G]
            # (Implementation detail: subdivide_deep logic)
            pass 
            
    return total_evals
```

**Edge Cases and Pitfalls:**
*   **Boundary Conditions:** The algorithm checks `na1 < da1` and `na2 < da2` to handle octree boundaries. If the bounding box is non-cubic, it is expanded to the smallest enclosing cube before processing [S1, unofficial].
*   **Precision:** Function evaluations are performed at specific grid points. Finite difference gradient estimation adds computational cost (`len(new_points)*6` evaluations) [S1, unofficial].
*   **Infinite Loops:** The loop terminates when `cells_to_subdivide` is empty [S1, unofficial].

## Subtopics

### Hermite Data Computation via Bisection
The parent method that computes intersection points and normals on edges. Edge Completeness Enforcement ensures the grid topology is consistent before or after this step to prevent gaps. [Hermite Data Computation via Bisection](822-hermite-data-computation-via-bisection.md)

### Adaptive Octree Traversal
The underlying data structure traversal method used to collect surface leaves and identify neighbors. [Adaptive Octree Traversal](52-adaptive-octree-traversal.md)

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Measured on various shapes (e.g., spheres, tori) with uniform and adaptive depths (3-7). Adaptive configurations reduce vertex count but require this enforcement step for watertightness [S1, unofficial].
*   **Strengths:** Ensures watertight meshes from adaptive octrees, preventing artifacts at depth boundaries [S1, unofficial].
*   **Limitations:** Requires additional function evaluations and subdivisions, increasing computational cost compared to uniform grids [S1, unofficial].

## Background
Edge Completeness Enforcement is part of the `isomesh` library by narnia-ai-mason, an unofficial implementation of adaptive iso-surface extraction [S1, unofficial]. It addresses the topological inconsistencies inherent in adaptive octrees where leaves at different depths share edges [S1, unofficial].

## Sources
[S1] narnia-ai-mason/isomesh, https://github.com/narnia-ai-mason/isomesh, fetched 2026-10-27 (unofficial)
