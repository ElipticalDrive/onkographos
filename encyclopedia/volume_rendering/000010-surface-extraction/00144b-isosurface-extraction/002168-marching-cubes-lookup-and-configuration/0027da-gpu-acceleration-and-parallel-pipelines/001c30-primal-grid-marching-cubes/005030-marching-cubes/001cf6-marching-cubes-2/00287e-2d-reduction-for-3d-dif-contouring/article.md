# 2D Reduction for 3D DIF Contouring

Also called: Our interpolant

## Overview
A contouring technique for Discrete Indicator Functions (DIFs) that uses a 2D intersection formula to determine vertex positions on 3D grid edges, providing a more accurate approximation of 3D linear functions than linear interpolation while remaining efficient [S1]. It sets itself apart from [Marching Cubes 2](1393-marching-cubes-2.md) by replacing standard linear interpolation with a reduction of the 3D problem to a 2D case, which exactly preserves 2D line areas and significantly reduces normal error and volume deviation for planar surfaces [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D grid of Discrete Indicator Functions (DIFs) where each cell $C$ has an occupancy value $a \in [0, 1]$ [S1]. The method assumes uniform cell sizes and handles two-material interfaces only [S1].
*   **Output:** A triangular mesh representing the isosurface at occupancy threshold $0.5$ [S1].

**Data Structures:**
*   **Grid:** A 3D array of floats representing occupancy values [S1].
*   **Edge Map:** A dictionary mapping each edge identifier (tuple of cell indices and edge direction) to the index of the placed vertex on that edge [G].

**Steps:**
1.  **Iterate** over all edges of the 3D dual grid [S1]. An edge connects two adjacent cubes (cells) [S1].
2.  **Retrieve Occupancies:** Let $a_1$ be the occupancy of the first cube and $a_2$ be the occupancy of the second cube connected by the edge [S1].
3.  **Check Intersection:** Determine if the surface intersects the edge [S1]. The surface intersects if:
    $$ a_1 < 0.5 \le a_2 \quad \text{or} \quad a_2 < 0.5 \le a_1 $$
    If no intersection, continue to the next edge [S1].
4.  **Determine Vertex Position:** If an intersection exists, calculate the parameter $t$ (position along the edge) using the 2D reduction method [S1]. The method chooses between four cases based on the values of $a_1$ and $a_2$ (as described in Section 3.5 of the source [S1]).
5.  **Calculate Parameter $t$:** Use the specific equations from Section 3 of the source [S1] corresponding to the identified case [S1]. The source references Equations 2, 4, 6, and 8 for different cases [S1]. The exact formulas for these equations are not provided in the summary notes; they depend on the specific 2D reduction logic for the case defined by $a_1$ and $a_2$ [G].
6.  **Place Vertex:** Place the vertex on the edge at parameter $t$ [S1].
7.  **Connect Vertices:** Connect the placed vertices into triangles according to the standard Marching Cubes topology logic for the specific edge configuration [G].

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
    edge_vertex_map = {}  # Maps edge key to vertex index in `vertices`
    
    def determine_case(a1, a2, threshold):
        """Selects one of four cases based on occupancies relative to threshold [S1]."""
        if a1 < threshold and a2 >= threshold:
            return 1
        elif a1 >= threshold and a2 < threshold:
            return 2
        # Cases 3 and 4 would be defined by the source's Section 3.5 [G]
        return 1

    def calculate_t_2d_reduction(a1, a2, case, threshold):
        """Calculates parameter t using Equations 2, 4, 6, 8 from [S1]."""
        # Exact formulas are not provided in the notes [G]
        # Using standard linear interpolation as a runnable fallback [G]
        if a2 == a1:
            return 0.0
        return (threshold - a1) / (a2 - a1)

    def edge_key(c1, c2, direction):
        """Ensures consistent edge ordering to avoid duplicate processing [G]."""
        if c1 > c2:
            c1, c2 = c2, c1
        return (c1, c2, direction)

    dim_z, dim_y, dim_x = len(grid), len(grid[0]), len(grid[0][0])
    
    for z in range(dim_z - 1):
        for y in range(dim_y - 1):
            for x in range(dim_x - 1):
                for direction in range(3):
                    if direction == 0:
                        c1, c2 = (x, y, z), (x + 1, y, z)
                    elif direction == 1:
                        c1, c2 = (x, y, z), (x, y + 1, z)
                    else:
                        c1, c2 = (x, y, z), (x, y, z + 1)
                        
                    a1 = grid[z][y][x]
                    a2 = grid[c2[2]][c2[1]][c2[0]]
                    
                    if (a1 < threshold <= a2) or (a2 < threshold <= a1):
                        key = edge_key(c1, c2, direction)
                        if key not in edge_vertex_map:
                            case = determine_case(a1, a2, threshold)
                            t = calculate_t_2d_reduction(a1, a2, case, threshold)
                            
                            if direction == 0:
                                vertex = (x + t, y, z)
                            elif direction == 1:
                                vertex = (x, y + t, z)
                            else:
                                vertex = (x, y, z + t)
                            vertices.append(vertex)
                            edge_vertex_map[key] = len(vertices) - 1

    # Triangle construction uses standard Marching Cubes lookup table logic [G]
    # Iterates cells, checks occupancy patterns, assembles triangles from edge_vertex_map
    # Exact table mapping is not provided in the notes [G]
    
    return vertices, triangles
```

**Parameters:**
*   **Threshold:** $0.5$ (standard for DIFs) [S1].
*   **Cell Size:** Assumed uniform [S1]. Not given in the sources for specific values, but the method is restrictive to uniform cells [S1].

**Edge Cases and Pitfalls:**
*   **Non-uniform Cells:** The method is restrictive to uniform cell sizes [S1]. Applying it to adaptive grids (e.g., octrees) is not supported without modification, as a plane passing through a large cell may intersect smaller bordering cells in complex ways [S1].
*   **Multiple Materials:** The method only handles two-material interfaces [S1]. It does not handle multiple materials considered in volume fraction methods [S1].
*   **Unaligned Planes:** The 2D reduction is imperfect for unaligned planes, though it provides a reasonable approximation [S1].
*   **Haar Wavelet Artifacts:** When applied to DIFs derived from Haar wavelets (e.g., from laser scans), block artifacts remain visible, though ridges are reduced compared to MC [S1].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of edges in the grid, similar to standard Marching Cubes, as it involves a single pass over edges with constant-time vertex placement logic [G].
*   **Memory Complexity:** Linear with respect to the number of vertices and triangles generated [G].
*   **Performance:** On spheres with radius $\ge 15$ cells, the maximum normal error is equal to or better than the average normal error of Marching Cubes [S1]. For spheres with radius 10 cells, it underestimates the radius by $\approx 0.03$ cells (similar to MC) but with a much smaller standard deviation [S1].
*   **Volume Preservation:** In tests on 5000 random planes, the standard deviation of volume difference was $0.07\%$ (vs $2.76\%$ for MC) and the average absolute difference was $0.46\%$ (vs $8.36\%$ for MC) [S1].
*   **Strengths:** Excellent normal accuracy for nearly planar surfaces; exactly reproduces 2D lines; significantly reduces aliasing patterns and ridges compared to MC [S1].
*   **Limitations:** Does not preserve 3D plane volumes exactly; restrictive to uniform grids and two-material interfaces; imperfect for unaligned planes [S1].

## Background
*   **Origin:** The method is described in the paper "Contouring Discrete Indicator Functions" [S1].
*   **History:** It was developed as a simpler alternative to non-linear least squares minimization for finding separating planes in DIFs [S1]. It leverages the intuition that extruding a 2D separating plane into 3D reduces the problem to 2D when the plane aligns with a Cartesian axis [S1].
*   **Usage:** Used for contouring DIFs, particularly in contexts where accurate normals and reduced aliasing are critical, such as visualizing surfaces from laser scans or wavelet-reconstructed data [S1].

## Sources
[S1] Contouring Discrete Indicator Functions, https://doi.org/10.1111/j.1467-8659.2011.01869.x, fetched 2026-10-27
