# Grid-based Traversal

Also called: Hierarchical grid traversal for irregular volumes, Parker's brute-force approach, Parker's method

## Overview
Grid-based Traversal is a method for rendering irregular volume data (tetrahedral meshes) at interactive rates by partitioning the volume into a hierarchical grid structure to quickly determine candidate tetrahedra [S1][S2, unofficial]. It sets itself apart from its siblings by avoiding explicit connectivity information between tetrahedra, instead relying on spatial hashing into a grid to manage the large number of primitives [S1]. Unlike [Projected Tetrahedra](877-projected-tetrahedra.md), which projects primitives to screen space, this method operates in object space using ray-tetrahedron intersection tests within grid cells [S1].

## How it works
**Inputs:** A tetrahedral mesh (irregular volume data), a hierarchical rectilinear grid structure, and a ray origin $O$ and direction $D$.
**Outputs:** Color and opacity values for the ray, computed via ray tracing.

**Data Structures:**
1.  **Hierarchical Grid:** A multi-level grid structure (e.g., three layers as used in the source implementation) covering the volume [S1][S2, unofficial].
2.  **Grid Cell Payload:** Each leaf grid cell stores a list of pointers/references to all tetrahedra that intersect the cell's spatial bounds [S1][S2, unofficial].
3.  **Cell Bounds Statistics:** Each grid cell stores the minimum ($min$) and maximum ($max$) scalar values of all tetrahedra attached to it [S1][S2, unofficial].

**Steps:**
1.  **Grid Construction:**
    *   Initialize the hierarchical grid.
    *   For each tetrahedron in the mesh, determine which grid cells it intersects.
    *   Attach the tetrahedron to the list of all intersecting grid cells [S1][S2, unofficial].
    *   Compute and store the scalar $min$ and $max$ values for the tetrahedra in each cell [S1][S2, unofficial].

2.  **Ray Traversal (Culling):**
    *   Traverse the hierarchical grid to find leaf cells intersected by the ray [S1][S2, unofficial].
    *   For each visited leaf cell, check if the ray's current depth range overlaps with the cell's scalar range $[min, max]$ relative to the target iso-value or transfer function range [S1][S2, unofficial].
    *   If the ray's relevant scalar values fall outside $[min, max]$, cull the cell and its attached tetrahedra [S1][S2, unofficial].

3.  **Intersection Testing:**
    *   For each tetrahedron attached to the current leaf cell, perform a ray-tetrahedron intersection test [S1].
    *   Since no connectivity is used, every attached tetrahedron must be tested sequentially [S1].

4.  **Iso-surface Computation:**
    *   If a ray intersects a tetrahedron face, determine the barycentric coordinates of the intersection point on that face (triangle) [S1].
    *   Use these barycentric coordinates to interpolate the scalar value $s$ at the intersection point from the vertex scalar values [S1].
    *   Compute the precise intersection with the iso-surface (or volume density) using linear interpolation of the scalar values along the ray segment within the tetrahedron [S1].

5.  **Compositing:**
    *   Collect all valid intersections from the ray traversal.
    *   Sort intersections back-to-front based on depth [S1][S2, unofficial] (consistent with the parent's visibility principles).
    *   Composite using the Porter-Duff over operation [S1][S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def grid_based_traversal(ray_origin, ray_direction, grid, iso_value):
    """
    ray_origin: vec3
    ray_direction: vec3
    grid: HierarchicalGrid object
    iso_value: float
    """
    # 1. Traverse grid to find candidate leaf cells
    candidate_cells = grid.traverse(ray_origin, ray_direction)
    
    intersections = []
    
    for cell in candidate_cells:
        # 2. Culling based on scalar bounds
        if not scalar_range_overlaps(cell.min_scalar, cell.max_scalar, iso_value):
            continue
            
        # 3. Test all attached tetrahedra
        for tet in cell.tetrahedra_list:
            hit = ray_tetrahedron_intersection(ray_origin, ray_direction, tet)
            if hit is not None:
                # 4. Compute precise iso-surface intersection
                # Determine barycentric coords on the intersected face
                barycentric = compute_barycentric_coords(hit.face, hit.intersection_point)
                
                # Interpolate scalar value at intersection
                scalar_at_hit = interpolate_scalar(tet.vertices, barycentric)
                
                # Compute exact iso-surface crossing if needed for shading/opacity
                # Linear interpolation along ray segment within tet
                iso_t = compute_iso_intersection(ray_origin, ray_direction, tet, scalar_at_hit, iso_value)
                
                intersections.append({
                    't': iso_t,
                    'scalar': scalar_at_hit,
                    'tet': tet
                })
    
    # 5. Sort and composite (back-to-front)
    intersections.sort(key=lambda x: x['t'], reverse=True)
    
    color = vec3(0.0)
    alpha = 0.0
    
    for hit in intersections:
        # Apply transfer function to scalar
        c, a = transfer_function(hit['scalar'])
        
        # Porter-Duff over
        color = c * a + color * (1.0 - a)
        alpha = alpha + a * (1.0 - alpha)
        
        if alpha >= 1.0:
            break
            
    return color, alpha

def scalar_range_overlaps(cell_min, cell_max, iso_value):
    """
    Returns True if the iso_value (or relevant range) falls within [cell_min, cell_max].
    """
    return cell_min <= iso_value <= cell_max
```

**Edge Cases and Pitfalls:**
*   **Performance:** Because no connectivity is used, all tetrahedra attached to a grid cell must be checked, leading to potential redundancy if cells are large or tetrahedra are small [S1].
*   **Curvilinear Grids:** The method as described does not natively support curvilinear volumes, though they can be handled by decomposing hexahedra into five tetrahedra [S1].
*   **Precision:** Linear interpolation assumes scalar fields vary linearly across cells [S1].

## Operating characteristics
*   **Time Complexity:** Depends on the number of tetrahedra per grid cell. In the worst case, it approaches testing all tetrahedra, but the hierarchical grid reduces the search space [S1][S2, unofficial].
*   **Memory Complexity:** Requires storing the grid structure and lists of tetrahedra pointers for each cell. The size of these lists depends on the grid resolution and tetrahedron density [S1][S2, unofficial].
*   **Measured Performance:** Rendered a bioelectric field with over one million tetrahedra at interactive rates on a 512x512 viewport using 16 or more processors [S1][S2, unofficial].
*   **Strengths:** Enables interactive rendering of large irregular volumes on supercomputers; iso-surface is computed on the fly, allowing dynamic changes during rendering [S1].
*   **Limitations:** Does not support curvilinear volumes natively; performance scales with the number of processors but relies on brute-force checking within grid cells [S1].

## Background
Grid-based Traversal was introduced by Parker et al. [PPL* 99] as one of the first methods capable of rendering irregular volume data at interactive rates on a supercomputer [S1][S2, unofficial]. It utilizes a hierarchical data structure for rectilinear grids to accelerate the determination of tetrahedra possibly hit by traversed grid cells [S1][S2, unofficial]. The method was demonstrated on a bioelectric field simulation, highlighting its ability to handle large datasets (over 1 million tetrahedra) [S1][S2, unofficial].

## Sources
[S1] Interactive Volume Rendering with Ray Tracing, https://doi.org/10.2312/egst.20061055, fetched 2026-10-24
[S2] Interactive Volume Rendering Techniques, https://www.scribd.com/document/479714183/Interactive-Volume-Rendering-with-Ray-Tracing, fetched 2026-10-25 (unofficial)
