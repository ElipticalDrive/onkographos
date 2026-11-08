# Grid-based Traversal

Also called: Hierarchical grid traversal for irregular volumes, Parker's brute-force approach, Parker's method

## Overview
Grid-based Traversal is a method for rendering irregular volume data (tetrahedral meshes) at interactive rates by partitioning the volume into a hierarchical grid structure to quickly determine candidate tetrahedra [S1][S2, unofficial]. It differs from sibling methods like [Projected Tetrahedra](877-projected-tetrahedra.md) by avoiding explicit geometric projection and sorting of primitives; instead, it relies on spatial hashing via a grid to filter tetrahedra for ray intersection tests [S1][S2, unofficial].

## How it works
**Inputs:** A tetrahedral mesh defining the volume; a hierarchical rectilinear grid structure; camera parameters (ray origin and direction); an iso-value for surface extraction.
**Outputs:** A rendered image with the iso-surface computed on the fly.

**Data Structures:**
1.  **Hierarchical Grid:** A multi-level grid (e.g., three layers as used in the original implementation) covering the volume bounding box. Each grid cell stores:
    *   `min_val`: The minimum scalar value among all tetrahedra attached to this cell.
    *   `max_val`: The maximum scalar value among all tetrahedra attached to this cell.
    *   `tetrahedra`: A list of references to all tetrahedra intersecting this cell.
2.  **Tetrahedron:** Defined by four vertices with associated scalar values.

**Steps:**
1.  **Grid Construction:**
    *   Initialize the hierarchical grid.
    *   For each tetrahedron in the mesh, determine which grid cells it intersects.
    *   Attach the tetrahedron to all intersecting grid leaf cells [S1][S2, unofficial].
    *   For each grid cell, compute and store the minimum and maximum scalar values of all attached tetrahedra [S1][S2, unofficial].
2.  **Ray Traversal:**
    *   For each pixel, generate a ray $R(t) = O + D \cdot t$.
    *   Traverse the hierarchical grid to find the first leaf cell intersected by the ray [S1][S2, unofficial].
    *   **Culling:** If the ray's current $t$ range does not intersect the cell's spatial bounds, or if the iso-value is outside the cell's `[min_val, max_val]` range, skip the cell [S1][S2, unofficial].
3.  **Intersection Testing:**
    *   For each tetrahedron attached to the current grid cell, perform a ray-tetrahedron intersection test [S1].
    *   Since no connectivity information is used, all attached tetrahedra must be checked sequentially [S1].
    *   If an intersection is found with a face (triangle) of the tetrahedron:
        *   Compute the barycentric coordinates of the intersection point relative to the triangle vertices [S1].
        *   Interpolate the scalar value at the intersection point using these barycentric coordinates and the vertex scalar values [S1].
        *   Compute the precise intersection with the iso-surface using linear interpolation of the scalar values along the ray segment within the tetrahedron [S1].
4.  **Compositing:**
    *   Accumulate the color and opacity at the computed iso-surface intersection point [S1][S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def grid_based_traversal(ray, grid, iso_value):
    # 1. Traverse grid to find candidate cells
    cells = grid.traverse(ray)
    
    for cell in cells:
        # 2. Cull based on scalar range
        if not (cell.min_val <= iso_value <= cell.max_val):
            continue
            
        # 3. Test all attached tetrahedra
        for tet in cell.tetrahedra:
            # Check if ray intersects tetrahedron
            hit, t_hit, face_normal, barycentric = ray.intersect_tetrahedron(tet)
            
            if hit:
                # 4. Interpolate scalar value at intersection
                scalar_at_hit = interpolate_scalar(tet.vertices, barycentric)
                
                # 5. Compute precise iso-surface intersection
                # Linear interpolation between entry and exit scalar values
                # along the ray segment within the tetrahedron
                t_iso = compute_iso_intersection(ray, tet, scalar_at_hit, iso_value)
                
                if t_iso is not None:
                    return t_iso, tet, barycentric
                    
    return None

def interpolate_scalar(vertices, barycentric):
    return sum(b * v.scalar for b, v in zip(barycentric, vertices))

def compute_iso_intersection(ray, tet, scalar_at_hit, iso_value):
    # Simplified: assumes linear variation along ray within tet
    # Requires finding entry scalar value as well
    # [G] Implementation detail: typically involves solving for t where
    # scalar(ray(t)) == iso_value using linear interpolation between
    # vertex values projected onto the ray.
    pass
```

**Parameters:**
*   **Grid Levels:** The original implementation used a multilevel grid with three layers [S1].
*   **Iso-value:** Computed on the fly, allowing dynamic changes during rendering [S1].

**Edge Cases:**
*   **Curvilinear Volumes:** Not directly supported; must be decomposed into tetrahedra (e.g., hexahedra into five tetrahedra) [S1].
*   **Precision:** Linear interpolation is used for scalar values and iso-surface intersection [S1].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of tetrahedra attached to traversed grid cells. No connectivity information is used, requiring sequential checking of all attached tetrahedra [S1].
*   **Memory Complexity:** Proportional to the number of tetrahedra times the average number of grid cells they intersect, plus grid storage for min/max values [S1][S2, unofficial].
*   **Performance:** Rendered a bioelectric field with over one million tetrahedra at interactive rates on a 512x512 viewport using 16 or more processors [S1][S2, unofficial].
*   **Strengths:** Enables interactive rendering of large irregular volumes on supercomputers; iso-surface is computed on the fly [S1].
*   **Limitations:** Does not support curvilinear volumes natively; relies on brute-force checking of attached tetrahedra without connectivity optimization [S1].

## Background
Grid-based Traversal was introduced by Parker et al. [PPL∗ 99] as one of the first methods to render irregular volume data at interactive rates on a supercomputer [S1][S2, unofficial]. It was demonstrated on a bioelectric field simulation with over one million tetrahedra [S1][S2, unofficial]. The method uses a hierarchical rectilinear grid to quickly determine tetrahedra possibly hit by traversed grid cells [S1][S2, unofficial].

## Sources
[S1] Interactive Volume Rendering with Ray Tracing, https://doi.org/10.2312/egst.20061055, fetched 2026-10-24
[S2] Interactive Volume Rendering Techniques, https://www.scribd.com/document/479714183/Interactive-Volume-Rendering-with-Ray-Tracing, fetched 2026-10-25 (unofficial)
