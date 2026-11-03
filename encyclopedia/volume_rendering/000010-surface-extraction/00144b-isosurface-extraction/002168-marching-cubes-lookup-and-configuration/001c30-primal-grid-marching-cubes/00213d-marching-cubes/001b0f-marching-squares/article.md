# Marching Squares

Also called: 2D Box Marching, 2D Isosurface Extraction, 2D Isosurface Extractor, Box2D, Marching Cubes 2D, Marching Squares / 2D Isosurface Extractor

## Overview
Marching Squares is a 2D contouring algorithm that extracts isocontour line segments from a discrete 2D scalar field by processing each grid cell independently [S1, S2, unofficial]. It determines the topology of the isosurface intersection within a square cell based on the scalar values at the four corners relative to an isolevel [S1, S2, unofficial]. Unlike its 3D counterpart, it operates on a 2×2 grid of vertices per cell [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 2D scalar field grid `Grid` of size $N_x \times N_y$, and an `IsoValue` threshold [S1, S2, unofficial].
*   **Output:** A list of line segments or a polygonal mesh representing the isosurface [S1, S3, unofficial].

**Data Structures:**
*   **Grid:** A 2D array of floats where `Grid[x][y]` stores the scalar value at coordinate $(x, y)$ [S2, unofficial].
*   **Edge/Segment:** A pair of 2D points $(p_1, p_2)$ [S1, unofficial].
*   **Vertex Buffer:** An optional hash map or list that stores unique vertices and their indices to ensure connectivity across cell boundaries when generating a mesh [S3, unofficial].

**Steps:**
1.  **Iterate over cells:** Loop $x$ from $0$ to $N_x - 2$ and $y$ from $0$ to $N_y - 2$ [S2, unofficial].
2.  **Evaluate corners:** Retrieve the scalar values at the four corners of the current cell in the order: bottom-left $(x, y)$, bottom-right $(x+1, y)$, top-right $(x+1, y+1)$, and top-left $(x, y+1)$ [S1, S2, unofficial]. Let these be $v_0, v_1, v_2, v_3$.
3.  **Compute case index:** Calculate an integer case index $C \in [0, 15]$ using binary encoding based on which corners are inside the isosurface ($v_i \ge \text{IsoValue}$) [S2, unofficial]:
    $$ C = \sum_{i=0}^{3} b_i 2^i, \quad \text{where } b_i = \begin{cases} 1 & \text{if } v_i \ge \text{IsoValue} \\ 0 & \text{otherwise} \end{cases} $$
4.  **Handle trivial cases:** If $C = 0$ (all corners below isovalue) or $C = 15$ (all corners above isovalue), no intersection occurs; continue to the next cell [S2, unofficial].
5.  **Determine intersections:** For each of the four edges of the square, check if the isovalue lies between the corner values [S1, unofficial].
6.  **Interpolate intersection points:** For each intersected edge between corners $i$ and $j$ with values $v_i, v_j$ and spatial coordinates $P_i, P_j$, calculate the exact intersection point $P$ using linear interpolation [G]:
    $$ P = P_i + \frac{\text{IsoValue} - v_i}{v_j - v_i} (P_j - P_i) $$
7.  **Connect segments:** Return line segments based on $C$ [S1, unofficial]:
    *   Cases 1/14, 2/13, 4/11, 8/7: Return a single edge connecting the two adjacent intersected edges [S1, unofficial].
    *   Cases 3/12, 5/10: Return a single edge connecting the two opposite intersected edges [S1, unofficial].
    *   Case 9: Return two edges following the logic of cases 1 and 8 [S1, unofficial].
    *   Case 6: Return two edges following the logic of cases 2 and 4 [S1, unofficial].
    *   If $C$ is even, swap the direction of the returned edge(s) to ensure clockwise winding around the solid [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def marching_squares(grid, isovalue):
    """
    Extracts isocontour line segments from a 2D scalar grid.
    
    Args:
        grid: 2D list of floats, shape (Nx, Ny).
        isovalue: Float threshold for extraction.
        
    Returns:
        list of tuples: Each tuple is ((x1, y1), (x2, y2)) representing a segment.
    """
    segments = []
    nx = len(grid)
    ny = len(grid[0])
    
    # Corner coordinates relative to cell origin (x, y)
    # 0: BL, 1: BR, 2: TR, 3: TL
    corner_coords = [(0, 0), (1, 0), (1, 1), (0, 1)]
    
    # Edge definitions: (corner_index_1, corner_index_2)
    # 0: BL-BR, 1: BR-TR, 2: TR-TL, 3: TL-BL
    edge_corners = [(0, 1), (1, 2), (2, 3), (3, 0)]
    
    # Case-to-edge-pair mapping (indices into edge_corners)
    case_edges = {
        1: [(0, 3)], 14: [(0, 3)],
        2: [(1, 0)], 13: [(1, 0)],
        4: [(2, 1)], 11: [(2, 1)],
        8: [(3, 2)], 7: [(3, 2)],
        3: [(0,)], 12: [(0,)],
        5: [(1, 3)], 10: [(1, 3)],
        9: [(0, 2)], 6: [(1, 3)]
    }
    
    def get_intersection(c1, c2, v1, v2, x, y):
        t = (isovalue - v1) / (v2 - v1)
        p1 = corner_coords[c1]
        p2 = corner_coords[c2]
        return (x + p1[0] + t * (p2[0] - p1[0]), y + p1[1] + t * (p2[1] - p1[1]))
            
    for x in range(nx - 1):
        for y in range(ny - 1):
            v0 = grid[x][y]
            v1 = grid[x+1][y]
            v2 = grid[x+1][y+1]
            v3 = grid[x][y+1]
            
            case = 0
            if v0 >= isovalue: case |= 1
            if v1 >= isovalue: case |= 2
            if v2 >= isovalue: case |= 4
            if v3 >= isovalue: case |= 8
            
            if case == 0 or case == 15:
                continue
                
            edge_pairs = case_edges.get(case, [])
            for pair in edge_pairs:
                e1, e2 = pair
                c1, c2 = edge_corners[e1]
                c3, c4 = edge_corners[e2]
                
                p1 = get_intersection(c1, c2, grid[x+corner_coords[c1][0]][y+corner_coords[c1][1]], grid[x+corner_coords[c3][0]][y+corner_coords[c3][1]], x, y)
                p2 = get_intersection(c3, c4, grid[x+corner_coords[c3][0]][y+corner_coords[c3][1]], grid[x+corner_coords[c1][0]][y+corner_coords[c1][1]], x, y)
                
                if case % 2 == 0:
                    p1, p2 = p2, p1
                    
                segments.append((p1, p2))
                
    return segments
```

**Edge Cases and Pitfalls:**
*   **Precision:** Linear interpolation assumes the scalar field is linear within the cell; highly non-linear fields introduce approximation error [G].
*   **Boundary Conditions:** The algorithm iterates up to $N_x - 2$ and $N_y - 2$, assuming the grid includes the outer boundary vertices [S2, unofficial].
*   **Degenerate Input:** If a corner value exactly equals the `IsoValue`, the comparison `v >= isovalue` treats it as inside, which is consistent but requires a vertex buffer to avoid duplicate vertices across adjacent cells [S2, S3, unofficial].
*   **Incorrect Implementation:** A common pitfall is using a hardcoded segment (e.g., connecting midpoints) for all non-trivial cases, which is topologically incorrect for most cases (e.g., 1, 2, 4, 7, 8, 11, 13, 14) and ignores the specific topology required by the Marching Squares lookup table [S2, unofficial].
*   **Vertex Scaling:** Some implementations scale vertices by a factor of 100 when adding them to a draw element, which may be a unit conversion or artifact [S3, unofficial].

## Subtopics

### [Dual Marching Squares](1418-dual-marching-squares.md)
A 2D surface extraction method where dual vertices are placed in the centers of their cells instead of being moved to the features of the function. This approach does not preserve sharp features [1418-dual-marching-squares.md].

### Marching Lines Algorithm
A variant that is not template-based but requires further processing to properly resolve apparent cell ambiguities [S4].

## Operating characteristics
*   **Time Complexity:** $O(N_x N_y)$, as each cell is processed independently in constant time [S2, unofficial].
*   **Memory Complexity:** $O(N_x N_y)$ for the input grid; output size is proportional to the number of intersected cells [S2, unofficial].
*   **Strengths:** Simple to implement, processes cells independently, and extracts 2D contours efficiently [S1, unofficial].
*   **Limitations:** Approximates the isosurface with linear segments, which may not capture high-frequency details without fine grid resolution [S1, unofficial]. Requires a vertex buffer for mesh connectivity [S3, unofficial].
*   **When to use:** Extracting 2D contours from scalar fields (e.g., height maps, 2D simulations, slice extraction from 3D volumes) [G].
*   **When not to use:** For 3D volumes (use [Marching Cubes](774-marching-cubes.md)) or when high-fidelity smooth surfaces are required without post-processing (consider [Marching Tetrahedra](1733-marching-tetrahedra.md) for better normal estimation) [G].

## Background
Not given in the sources.

## Sources
[S1] BorisTheBrave/mc-dc, https://github.com/BorisTheBrave/mc-dc, fetched 2026-10-26 (unofficial)
[S2] alphaparkinc/genpark-marching-cubes-isosurface-extractor-skill, https://github.com/alphaparkinc/genpark-marching-cubes-isosurface-extractor-skill, fetched 2026-10-27 (unofficial)
[S3] 2453852330/VoxelHelper, https://github.com/2453852330/VoxelHelper, fetched 2026-10-27 (unofficial)
[S4] Volume-Enclosing Surface Extraction, https://arxiv.org/abs/1011.1787v4, fetched 2026-10-26
