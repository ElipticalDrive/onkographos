# 2D Reduction for 3D DIF Contouring

Also called: Our interpolant

## Overview
A contouring technique for Discrete Indicator Functions (DIFs) that determines vertex positions on 3D grid edges using a 2D intersection formula, providing a more accurate approximation of 3D linear functions than linear interpolation while remaining efficient. It sets itself apart from [Marching Cubes 2](1393-marching-cubes-2.md) by replacing standard linear interpolation with a reduction of the 3D problem to a 2D case, which exactly preserves 2D line areas and significantly reduces normal error and volume deviation for planar surfaces.

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D grid of Discrete Indicator Functions (DIFs). Each cell $C$ has an occupancy value $a \in [0, 1]$. The method assumes uniform cell sizes and handles two-material interfaces only.
*   **Output:** A triangular mesh representing the isosurface at occupancy threshold $0.5$.

**Data Structures:**
*   **Grid:** A 3D array of floats representing occupancy values.
*   **Edge Configuration:** For each edge shared by two adjacent cubes, the occupancies of the two cubes are retrieved.

**Steps:**
1.  **Iterate** over all edges of the 3D dual grid. An edge connects two adjacent cubes (cells).
2.  **Retrieve Occupancies:** Let $a_1$ be the occupancy of the first cube and $a_2$ be the occupancy of the second cube connected by the edge.
3.  **Check Intersection:** Determine if the surface intersects the edge. The surface intersects if:
    $$ a_1 < 0.5 \le a_2 \quad \text{or} \quad a_2 < 0.5 \le a_1 $$
    If no intersection, continue to the next edge.
4.  **Determine Vertex Position:** If an intersection exists, calculate the parameter $t$ (position along the edge) using the 2D reduction method. The method chooses between four cases based on the values of $a_1$ and $a_2$ (as described in Section 3.5 of the source [S1]).
5.  **Calculate Parameter $t$:** Use the specific equations from Section 3 of the source [S1] corresponding to the identified case. The source references Equations 2, 4, 6, and 8 for different cases. The exact formulas for these equations are not provided in the summary notes; they depend on the specific 2D reduction logic for the case defined by $a_1$ and $a_2$.
6.  **Place Vertex:** Place the vertex on the edge at parameter $t$.
7.  **Connect Vertices:** Connect the placed vertices into triangles according to the standard Marching Cubes topology logic for the specific edge configuration.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def contour_dif_2d_reduction(grid, threshold=0.5):
    """
    Contours a 3D DIF grid using 2D reduction for vertex placement.
    Assumes uniform grid spacing.
    """
    vertices = []
    triangles = []
    
    # Iterate over all edges in the grid
    # An edge is defined by two adjacent cells (c1, c2) and the edge index within the cell
    for edge in grid.edges():
        c1, c2 = edge.cells
        a1 = grid.get_occupancy(c1)
        a2 = grid.get_occupancy(c2)
        
        # Check for intersection with the threshold
        if (a1 < threshold <= a2) or (a2 < threshold <= a1):
            # Determine the case based on a1 and a2
            # The source specifies four cases in Section 3.5.
            # Since the exact mapping of cases to equations is not fully detailed in the notes,
            # we assume a function that selects the correct 2D reduction formula.
            case = determine_case(a1, a2)
            
            # Calculate parameter t using the specific equation for the case
            # Equations 2, 4, 6, 8 from [S1] are referenced.
            # Implementation requires the specific formulas from the source paper.
            t = calculate_t_2d_reduction(a1, a2, case)
            
            # Place vertex on the edge
            vertex = edge.get_vertex_at(t)
            vertices.append(vertex)
            
            # Store edge index for triangle construction
            edge.vertex_index = len(vertices) - 1

    # Construct triangles based on the standard Marching Cubes lookup table
    # using the vertex indices placed on the edges.
    triangles = construct_triangles(grid, vertices)
    
    return triangles
```

**Parameters:**
*   **Threshold:** $0.5$ (standard for DIFs).
*   **Cell Size:** Assumed uniform. Not given in the sources for specific values, but the method is restrictive to uniform cells.

**Edge Cases and Pitfalls:**
*   **Non-uniform Cells:** The method is restrictive to uniform cell sizes. Applying it to adaptive grids (e.g., octrees) is not supported without modification, as a plane passing through a large cell may intersect smaller bordering cells in complex ways [S1].
*   **Multiple Materials:** The method only handles two-material interfaces. It does not handle multiple materials considered in volume fraction methods [S1].
*   **Unaligned Planes:** The 2D reduction is imperfect for unaligned planes, though it provides a reasonable approximation [S1].
*   **Haar Wavelet Artifacts:** When applied to DIFs derived from Haar wavelets (e.g., from laser scans), block artifacts remain visible, though ridges are reduced compared to MC [S1].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of edges in the grid, similar to standard Marching Cubes, as it involves a single pass over edges with constant-time vertex placement logic.
*   **Memory Complexity:** Linear with respect to the number of vertices and triangles generated.
*   **Performance:** On spheres with radius $\ge 15$ cells, the maximum normal error is equal to or better than the average normal error of Marching Cubes [S1]. For spheres with radius 10 cells, it underestimates the radius by $\approx 0.03$ cells (similar to MC) but with a much smaller standard deviation [S1].
*   **Volume Preservation:** In tests on 5000 random planes, the standard deviation of volume difference was $0.07\%$ (vs $2.76\%$ for MC) and the average absolute difference was $0.46\%$ (vs $8.36\%$ for MC) [S1].
*   **Strengths:** Excellent normal accuracy for nearly planar surfaces; exactly reproduces 2D lines; significantly reduces aliasing patterns and ridges compared to MC [S1].
*   **Limitations:** Does not preserve 3D plane volumes exactly; restrictive to uniform grids and two-material interfaces; imperfect for unaligned planes [S1].

## Background
*   **Origin:** The method is described in the paper "Contouring Discrete Indicator Functions" [S1].
*   **History:** It was developed as a simpler alternative to non-linear least squares minimization for finding separating planes in DIFs. It leverages the intuition that extruding a 2D separating plane into 3D reduces the problem to 2D when the plane aligns with a Cartesian axis [S1].
*   **Usage:** Used for contouring DIFs, particularly in contexts where accurate normals and reduced aliasing are critical, such as visualizing surfaces from laser scans or wavelet-reconstructed data [S1].

## Sources
[S1] Contouring Discrete Indicator Functions, https://doi.org/10.1111/j.1467-8659.2011.01869.x, fetched 2026-10-27
