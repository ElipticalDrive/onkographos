# Marching Squares

Also called: Marching Cubes 2D, 2D Isosurface Extraction, 2D Isocontour Extraction

## Overview
Marching Squares is an algorithm for extracting a polygonal contour (isolines) from a two-dimensional scalar field by processing each grid cell independently [S2, S11]. It determines the topology of the isosurface intersection with a square cell based on the scalar values at the cell's four vertices relative to an isolevel [S2]. Unlike its 3D counterpart, [Marching Cubes](774-marching-cubes.md), which can produce ambiguous saddle points requiring additional heuristics, the 2D case on a square grid shares the same lookup-table and interpolation framework but requires explicit resolution for saddle points (cases 5 and 10) using the cell center value [S2, S11].

## How it works
**Inputs and Outputs:**
*   **Input:** A 2D scalar field $F$ defined on a uniform grid of size $N_x \times N_y$ (where $N_x$ is columns, $N_y$ is rows) [S2][S8, unofficial]. Each cell has side length $CELL\_SIZE$ [S4, unofficial]. An isovalue $L$ [S2][S5, unofficial].
*   **Output:** A set of line segments representing the contour $F(x,y) = L$ [S1, S4, unofficial].

**Data Structures:**
*   **Lookup Table:** An array of 16 entries (indices 0–15) [S2]. Each entry specifies which edges of the square cell are intersected by the isosurface [S6, unofficial].
*   **Vertex Buffer:** A list of unique 2D points to avoid duplicate vertices for shared edges [S2][S10, unofficial].

**Numbered Steps:**

1.  **Grid Setup:** Define the contouring grid. The contouring grid consists of $(N_x - 1) \times (N_y - 1)$ cells [S2, S11]. Iterate over each cell $(i, j)$ where $i \in [0, N_x - 2]$ and $j \in [0, N_y - 2]$ [S8, unofficial]. Loop order may vary (e.g., row-major or column-major) [S2][S5, unofficial].
2.  **Vertex Sampling:** For cell $(i, j)$, retrieve the scalar values at the four corners [S4, unofficial]. Let the grid coordinates be $(x, y) = (i \cdot CELL\_SIZE, j \cdot CELL\_SIZE)$ [G]. The corners are:
    *   $v_0$: Bottom-Left $(x, y)$, value $s_0 = F[i][j]$
    *   $v_1$: Bottom-Right $(x + CELL\_SIZE, y)$, value $s_1 = F[i+1][j]$
    *   $v_2$: Top-Right $(x + CELL\_SIZE, y + CELL\_SIZE)$, value $s_2 = F[i+1][j+1]$
    *   $v_3$: Top-Left $(x, y + CELL\_SIZE)$, value $s_3 = F[i][j+1]$
    [S4, S7, S10, unofficial]
3.  **Case Index Calculation:** Determine the binary state of each vertex relative to the isovalue $L$ [S8, unofficial]. A vertex is "inside" if $s_k \ge L$ (or $s_k > L$, provided consistency is maintained) [S2][S8, unofficial]. Compute the 4-bit case index $C$:
    $$ C = b_0 + 2b_1 + 4b_2 + 8b_3 $$
    where $b_k = 1$ if $s_k \ge L$, else $0$ [S4, unofficial][S11]. Note: Bit ordering must be consistent with the lookup table definition. A common convention is MSB at Top-Left ($v_3$) or Bottom-Left ($v_0$) [S2]. The source [S2] specifies walking clockwise from Top-Left as MSB. Source [S4, unofficial] uses a different bit assignment. The implementer must ensure the bit order matches the lookup table [S2][S4, unofficial].
4.  **Lookup:** Use $C$ to access the lookup table [S2]. If $C=0$ (all outside) or $C=15$ (all inside), no geometry is generated for this cell [S2][S7, unofficial]. Otherwise, the table lists the edges needed to represent the cell [S2].
5.  **Intersection Calculation:** For each intersected edge, calculate the exact intersection point using linear interpolation [S2, S11]. For an edge between vertices with values $s_{start}$ and $s_{end}$ at positions $p_{start}$ and $p_{end}$:
    $$ t = \frac{L - s_{start}}{s_{end} - s_{start}} $$
    $$ p_{intersect} = p_{start} + t \cdot (p_{end} - p_{start}) $$
    [S5, unofficial]
6.  **Vertex Insertion:** Insert the calculated intersection points into a vertex buffer [S10, unofficial]. To optimize, check if a point already exists (within a tolerance) before adding it, or simply add all points and use an index buffer for rendering [S2][S10, unofficial].
7.  **Edge Generation:** Create line segments connecting the intersection points on the intersected edges [S5, unofficial]. For cases with two intersection points, create one segment [S5, unofficial]. For cases with four intersection points (ambiguous saddle cases), create two segments [S5, unofficial][S11].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def marching_squares(grid, isovalue, cell_size):
    """
    grid: 2D list of floats, shape (Ny, Nx)
    isovalue: float
    cell_size: float
    Returns: list of line segments, each segment is ((x1, y1), (x2, y2))
    """
    Ny = len(grid)
    Nx = len(grid[0])
    segments = []
    
    # Lookup table: case_index -> list of edge pairs to connect
    # Edges: 0=Bottom(v0-v1), 1=Right(v1-v2), 2=Top(v2-v3), 3=Left(v3-v0)
    # Bit order: v0=LSB(1), v1(2), v2(4), v3=MSB(8)
    lookup = {
        0: [],
        1: [(3, 0)], 2: [(0, 1)], 3: [(3, 1)], 4: [(1, 2)],
        5: [(3, 2), (0, 1)], 6: [(0, 2)], 7: [(2, 3)],
        8: [(2, 3)], 9: [(0, 1)], 10: [(3, 0), (1, 2)],
        11: [(1, 2)], 12: [(0, 1)], 13: [(0, 2)], 14: [(3, 0)],
        15: []
    }
    
    def interpolate(v_start, v_end, p_start, p_end, iso):
        if abs(v_end - v_start) < 1e-9:
            return ((p_start[0] + p_end[0]) / 2.0, (p_start[1] + p_end[1]) / 2.0)
        t = (iso - v_start) / (v_end - v_start)
        return (p_start[0] + t * (p_end[0] - p_start[0]),
                p_start[1] + t * (p_end[1] - p_start[1]))

    for j in range(Ny - 1):
        for i in range(Nx - 1):
            s0, s1, s2, s3 = grid[j][i], grid[j][i+1], grid[j+1][i+1], grid[j+1][i]
            p0, p1, p2, p3 = (i*cell_size, j*cell_size), ((i+1)*cell_size, j*cell_size), \
                             ((i+1)*cell_size, (j+1)*cell_size), (i*cell_size, (j+1)*cell_size)
            
            case = 0
            if s0 >= isovalue: case |= 1
            if s1 >= isovalue: case |= 2
            if s2 >= isovalue: case |= 4
            if s3 >= isovalue: case |= 8
            
            if case == 0 or case == 15:
                continue
                
            edge_pairs = lookup[case]
            for e1, e2 in edge_pairs:
                pts = []
                for e in (e1, e2):
                    if e == 0: pt = interpolate(s0, s1, p0, p1, isovalue)
                    elif e == 1: pt = interpolate(s1, s2, p1, p2, isovalue)
                    elif e == 2: pt = interpolate(s2, s3, p2, p3, isovalue)
                    elif e == 3: pt = interpolate(s3, s0, p3, p0, isovalue)
                    pts.append(pt)
                segments.append((pts[0], pts[1]))
    return segments
```

**Edge Cases and Pitfalls:**
*   **Saddle Points:** Cases 5 and 10 (and their inverses) create ambiguous connections if the grid is not square or if the data is not planar [S2]. In 2D squares, these are topologically distinct but require careful handling to ensure the correct pair of edges is connected [S2, S11].
*   **Vertex on Isovalue:** If a vertex value equals the isovalue exactly, it must be treated consistently (e.g., as "inside" or "outside") to avoid ambiguity [S2].
*   **Aliasing:** Directly checking $f(x,y) == 0$ causes aliasing; using a threshold $L$ with interpolation is required [S1, unofficial].
*   **Non-Planar Quads:** If the 2D grid is embedded in 3D space and the quads are not planar, the interpolation scheme becomes ambiguous [S2].

## Subtopics

### [Marching Lines Algorithm](942-marching-lines-algorithm.md)
The 1D analog to Marching Squares, processing grid cells in a linear, row-by-row manner. It is not template-based and requires further processing to resolve apparent cell ambiguities.

### [Dual Marching Squares](1418-dual-marching-squares.md)
Generates a polygonal mesh by placing a single vertex per grid cell at the cell's center (dual grid position) and connecting these vertices into quads. It produces smoother results and avoids topological inconsistencies common in standard Marching Squares, though it may lose sharp features.

### [Marching Squares (Isoband)](14185-marching-squares-isoband.md)
Generates filled contour bands between an upper and lower threshold. It classifies cells into three states (below, within, above) rather than two, allowing for continuous filled regions.

### 2D Banded Contouring
Finds the 2D region where an interval $[L, H]$ of values exists [S3, unofficial]. Each square corner is assigned a ternary value: 0 if value $< L$, 1 if $L \le value \le H$, 2 if value $> H$ [S3, unofficial]. This results in $3^4 = 81$ initial cases, reduced to 13 canonical cases after accounting for symmetries (rotational, mirror, high/low) [S3, unofficial]. The case number is determined by the base-3 number formed by the corner states [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N_x N_y)$, as each cell is processed independently [S2].
*   **Memory Complexity:** $O(N_x N_y)$ for the input grid and output geometry [S2].
*   **Parallelism:** The algorithm is embarrassingly parallel because all cells are processed independently [S2, S11].
*   **Performance:** A naive implementation performs linear interpolation twice per edge for isolines, resulting in duplicate vertices [S2, S11]. Caching interpolation results for one row of the input grid can reduce computational overhead [S2]. Using indexed geometric primitives reduces output size [S2].
*   **Strengths:** Simple to implement, robust for 2D data, fast [S2].
*   **Limitations:** Produces jagged contours at low resolutions; aliasing if not interpolated correctly; ambiguous saddle points in 2D squares or non-planar 2D embeddings [S1, unofficial][S2].

## Background
Marching Squares is the 2D analog of the Marching Cubes algorithm [S2, S11]. It was developed as a method to generate contours for two-dimensional scalar fields, such as topographic maps or weather isobars [S2]. The algorithm processes each cell in the grid independently, calculating a cell index using comparisons of the contour level with the data values at the cell corners, and using a pre-built lookup table to describe the output geometry [S2]. It is used in scientific data visualization, such as the 'Hurricane Isabel Scientific Data Visualization' project [S5, unofficial].

## Sources
[S1] Marching Cubes, Explained Using Marching Squares, https://blog.42yeah.is/algorithm/2023/07/15/marching-cubes.html, fetched 2026-10-26 (unofficial)
[S2] Marching squares, https://en.wikipedia.org/wiki/Marching_squares, fetched 2026-10-26
[S3] Smokeview (Version 6) A Tool for Visualizing Fire Dynamics Simulation Data Volume II : Technical Reference Guide, https://doi.org/10.6028/nist.sp.1017-2, fetched 2026-10-26 (unofficial)
[S4] BorisTheBrave/mc-dc, https://github.com/BorisTheBrave/mc-dc, fetched 2026-10-26 (unofficial)
[S5] killerv1per/vtk-data-visualization, https://github.com/killerv1per/vtk-data-visualization, fetched 2026-10-26 (unofficial)
[S6] thebusytypist/cudc, https://github.com/thebusytypist/cudc, fetched 2026-10-26 (unofficial)
[S7] alphaparkinc/genpark-marching-cubes-isosurface-extractor-skill, https://github.com/alphaparkinc/genpark-marching-cubes-isosurface-extractor-skill, fetched 2026-10-27 (unofficial)
[S8] Alpha-Park/genpark-marching-cubes-isosurface-extractor-skill, https://github.com/Alpha-Park/genpark-marching-cubes-isosurface-extractor-skill, fetched 2026-10-27 (unofficial)
[S9] , https://www.scitepress.org/PublishedPapers/2007/20798/pdf/index.html, fetched 2026-10-27 (unofficial)
[S10] 2453852330/VoxelHelper, https://github.com/2453852330/VoxelHelper, fetched 2026-10-27 (unofficial)
[S11] Marching squares, https://smunix.github.io/en.wikipedia.org/wiki/Marching_squares.html, fetched 2026-10-30
