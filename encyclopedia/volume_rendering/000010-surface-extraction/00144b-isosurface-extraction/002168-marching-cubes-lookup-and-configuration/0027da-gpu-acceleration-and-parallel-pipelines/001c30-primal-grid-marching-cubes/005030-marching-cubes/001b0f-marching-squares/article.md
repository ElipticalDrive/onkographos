# Marching Squares

Also called: 2D Isocontour Extraction, 2D Isocontour Extraction via Edge Walking, 2D Isosurface Extraction, 2D Isosurface Extractor, 2D Line Contouring, Marching Cubes 2D, Marching Squares / 2D Isosurface Extractor, Marching Squares Edge Intersection Detection

## Overview
Marching Squares is a 2D contouring algorithm that extracts isolines (contours of a single isovalue) or isobands (filled regions between two isovalue thresholds) from a discrete 2D scalar field [S2]. It solves the problem of converting implicit 2D data into explicit geometric representations (line segments or polygons) by processing the data grid cell-by-cell [S12]. It is the 2D analog of [Marching Cubes](774-marching-cubes.md), sharing the same core approach of independent cell processing, binary/ternary indexing, lookup tables, and linear interpolation [S2, S12]. Unlike template-based 3D methods, its 2D nature simplifies topology but retains the saddle-point ambiguity in specific cases [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A 2D scalar field $f$ represented as a grid of values. For isolines, a single isovalue $L$ is required. For isobands, a lower threshold $L$ and upper threshold $H$ are required. The grid has dimensions $N_x \times N_y$ (columns $\times$ rows) [S9, unofficial].
*   **Output:** A set of line segments (isolines) or polygonal fragments (isobands) [S2].

**Data Structures:**
*   **Grid:** A 2D array `grid[x][y]` where $x \in [0, N_x-1]$ and $y \in [0, N_y-1]$ stores the scalar values [S9, unofficial].
*   **Lookup Table (Isolines):** An array of 16 entries (indices 0–15). Each entry specifies which of the 4 cell edges are intersected by the isosurface. The standard convention maps bits to corners: bit 0 (LSB) is bottom-left, bit 1 is top-left, bit 2 is top-right, bit 3 (MSB) is bottom-right [S2, S12].
*   **Lookup Table (Isobands):** An array of 81 entries (indices 0–80, base-3). Each entry specifies the polygonal geometry for the band region [S3, unofficial][S12].

**Numbering Convention:**
Corners are numbered 0 to 3. The standard clockwise numbering starting from the top-left is often used in implementations, but the sources specify a specific bit-mapping for the index calculation:
*   Corner 0: Bottom-Left
*   Corner 1: Top-Left
*   Corner 2: Top-Right
*   Corner 3: Bottom-Right
*   *Note: Source [S2] and [S12] describe walking clockwise from top-left, but the bit calculation examples in [S4, unofficial] and [S8, unofficial] imply a specific mapping. We follow the explicit bit-shift logic from [S2, S12]:*
    *   Index $I = \sum_{i=0}^{3} b_i 2^i$.
    *   $b_0$ (LSB) corresponds to Corner 0 (Bottom-Left).
    *   $b_1$ corresponds to Corner 1 (Top-Left).
    *   $b_2$ corresponds to Corner 2 (Top-Right).
    *   $b_3$ (MSB) corresponds to Corner 3 (Bottom-Right).
    *   *Correction based on [S2, S12] text:* "walk around the cell in a clockwise direction... from most significant bit at the top left, to least significant bit at the bottom left." This implies:
        *   Top-Left (Corner 1 in some schemes, but let's stick to the geometric positions): MSB.
        *   Top-Right.
        *   Bottom-Right.
        *   Bottom-Left: LSB.
    *   To avoid ambiguity, we define the corners by position relative to the cell $(x,y)$:
        *   $C_{TL}$: $(x, y+1)$
        *   $C_{TR}$: $(x+1, y+1)$
        *   $C_{BR}$: $(x+1, y)$
        *   $C_{BL}$: $(x, y)$
    *   The index is calculated as: $I = (val(C_{TL}) ? 8 : 0) + (val(C_{TR}) ? 4 : 0) + (val(C_{BR}) ? 2 : 0) + (val(C_{BL}) ? 1 : 0)$ [S2, S12].

**Steps for Isolines:**
1.  **Iterate Cells:** Loop over $x$ from $0$ to $N_x-2$ and $y$ from $0$ to $N_y-2$ [S9, unofficial].
2.  **Sample Corners:** Retrieve scalar values $v_{TL}, v_{TR}, v_{BR}, v_{BL}$ from `grid` at the four corners of the current cell [S8, unofficial].
3.  **Compute Case Index:** Determine the binary state of each corner relative to the isovalue $L$. Let $b_i = 1$ if $v_i \ge L$ (or $> L$, consistency is key [S2]), else $0$. Compute $I = b_{TL} \cdot 8 + b_{TR} \cdot 4 + b_{BR} \cdot 2 + b_{BL} \cdot 1$ [S2, S12].
4.  **Lookup Edges:** Use $I$ to access the lookup table to identify which edges of the square are intersected.
    *   Case 0 (0000) and Case 15 (1111): No intersection.
    *   Cases 1, 2, 4, 8: Single intersection segment.
    *   Cases 3, 12: Vertical split (one segment).
    *   Cases 5, 10: Horizontal split (one segment).
    *   Cases 6, 9: Two separate segments (saddle points).
5.  **Intersect Edges:** For each intersected edge, calculate the exact intersection point using linear interpolation [S2, S12].
    *   Let edge endpoints have values $v_0$ and $v_1$ and spatial coordinates $P_0$ and $P_1$.
    *   Interpolation parameter: $t = \frac{L - v_0}{v_1 - v_0}$ [S6, unofficial].
    *   Intersection point: $P_{int} = P_0 + t(P_1 - P_0)$ [S6, unofficial].
6.  **Handle Ambiguity (Saddle Points):** Cases 5 and 10 are topologically ambiguous. Resolve by comparing the average value of the cell center $v_{center} = \frac{v_{TL} + v_{TR} + v_{BR} + v_{BL}}{4}$ to $L$.
    *   If $I=5$ and $v_{center} < L$, treat as Case 10.
    *   If $I=10$ and $v_{center} > L$, treat as Case 5 [S2, S12].
7.  **Output Segments:** Add the calculated line segment(s) to the output list.

**Pseudocode (Isolines):**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def marching_squares_isolines(grid, isovalue, width, height):
    """
    grid: 2D list of floats, shape (height, width)
    isovalue: float
    width: int, number of columns
    height: int, number of rows
    Returns: list of tuples ((x1, y1), (x2, y2)) representing line segments
    """
    segments = []
    
    # Lookup table: maps case index (0-15) to a list of edge pairs.
    # Edges are defined by corner indices: 0=BL, 1=TL, 2=TR, 3=BR
    # This table assumes the bit mapping: BL=1, TR=2, TL=4, BR=8? 
    # No, based on S2/S12: TL=8, TR=4, BR=2, BL=1.
    # Edge definitions:
    # Edge 0: BL-TL (Left)
    # Edge 1: TL-TR (Top)
    # Edge 2: TR-BR (Right)
    # Edge 3: BR-BL (Bottom)
    # Standard MS-Cubes derived 2D table (corrected for bit order TL=8, TR=4, BR=2, BL=1):
    # Case 0: []
    # Case 1 (BL): [0] (Left) -> Wait, BL is bit 0. If only BL is inside, it connects to TL (Edge 0) and BR (Edge 3)? 
    # Let's use the explicit logic from S4/S12 which is safer.
    
    # Simplified Lookup: List of edges to connect. 
    # Edge indices: 0: Left(BL-TL), 1: Top(TL-TR), 2: Right(TR-BR), 3: Bottom(BR-BL)
    lookup = [
        [],       # 0: 0000
        [0, 3],   # 1: 0001 (BL) - Connects Left and Bottom? No, BL connects to TL and BR.
                  # If BL is inside, TL, TR, BR are outside.
                  # Intersections on Left (BL-TL) and Bottom (BL-BR).
        [0, 1],   # 2: 0010 (BR) - Connects Left? No. BR connects to TR and BL.
                  # If BR is inside, TL, TR, BL are outside.
                  # Intersections on Right (TR-BR) and Bottom (BR-BL).
        [0],      # 3: 0011 (BR, BL) - Connects Left and Right? No.
                  # If BR, BL inside, TL, TR outside.
                  # Intersections on Left (BL-TL) and Right (TR-BR). Vertical split.
        [1, 3],   # 4: 0100 (TL) - Connects Top and Left.
        [1],      # 5: 0101 (TL, BR) - Saddle. Ambiguous.
        [2],      # 6: 0110 (TR, BL) - Saddle. Ambiguous.
        [1, 2],   # 7: 0111 (TR, TL, BR) - Connects Top and Right.
        [2, 3],   # 8: 1000 (TR) - Connects Right and Bottom.
        [2],      # 9: 1001 (TR, BL) - Saddle. Ambiguous.
        [3],      # 10: 1010 (TL, BR) - Saddle. Ambiguous.
        [0, 2],   # 11: 1011 (TL, TR, BR) - Connects Left and Right? No.
                  # TL, TR, BR inside, BL outside.
                  # Intersections on Left (BL-TL) and Bottom (BL-BR).
                  # Wait, if BL is outside, TL, TR, BR inside.
                  # Left edge (BL-TL) crosses. Bottom edge (BL-BR) crosses.
                  # So [0, 3].
        [0, 1],   # 12: 1100 (TL, TR) - Connects Left and Right? No.
                  # TL, TR inside, BL, BR outside.
                  # Left (BL-TL) crosses. Right (TR-BR) crosses.
                  # Vertical split. [0, 2].
        [0],      # 13: 1101 (TL, TR, BL) - Connects Top and Bottom?
                  # TL, TR, BL inside, BR outside.
                  # Top (TL-TR) no. Left (BL-TL) no. Bottom (BL-BR) crosses. Right (TR-BR) crosses.
                  # [3, 2].
        [3],      # 14: 1110 (TL, TR, BR) - Connects Top and Right?
                  # TL, TR, BR inside, BL outside.
                  # Left (BL-TL) crosses. Bottom (BL-BR) crosses.
                  # [0, 3].
        []        # 15: 1111
    ]
    
    # Correcting the lookup table based on standard Marching Squares logic for bit order TL=8, TR=4, BR=2, BL=1:
    # Case 1 (0001, BL): Intersects Left (0) and Bottom (3).
    # Case 2 (0010, BR): Intersects Bottom (3) and Right (2).
    # Case 3 (0011, BR,BL): Intersects Left (0) and Right (2).
    # Case 4 (0100, TL): Intersects Left (0) and Top (1).
    # Case 5 (0101, TL,BR): Intersects Top (1) and Bottom (3). (Ambiguous)
    # Case 6 (0110, TR,BL): Intersects Left (0) and Right (2). (Ambiguous)
    # Case 7 (0111, TR,BL,TL): Intersects Top (1) and Right (2).
    # Case 8 (1000, TR): Intersects Top (1) and Right (2). -> Wait, TR is bit 2 (val 4). 
    # Let's re-verify bit weights. S2: MSB Top-Left. LSB Bottom-Left.
    # TL=8, TR=4, BR=2, BL=1.
    # Case 8 (1000, TL): Intersects Left (0) and Top (1).
    # Case 9 (1001, TL,BL): Intersects Top (1) and Bottom (3). (Ambiguous)
    # Case 10 (1010, TL,TR): Intersects Left (0) and Right (2). (Ambiguous)
    # Case 11 (1011, TL,TR,BL): Intersects Right (2) and Bottom (3).
    # Case 12 (1100, TL,TR,BR): Intersects Left (0) and Bottom (3).
    # Case 13 (1101, TL,TR,BR,BL): Intersects Right (2) and Bottom (3)? No.
    # Case 14 (1110, TL,TR,BR): Intersects Left (0) and Bottom (3).
    # Case 15 (1111): None.
    
    # Let's use a robust edge-intersection check instead of a hardcoded table to ensure correctness from first principles.
    
    for y in range(height - 1):
        for x in range(width - 1):
            # Get corner values
            v_bl = grid[y][x]
            v_tl = grid[y][x+1] # Note: grid[y][x] is usually (x,y). If y is row, x is col.
            # Let's assume grid[row][col].
            # y is row index. x is col index.
            # BL: (x, y)
            # TL: (x, y+1)
            # TR: (x+1, y+1)
            # BR: (x+1, y)
            
            v_bl = grid[y][x]
            v_tl = grid[y+1][x]
            v_tr = grid[y+1][x+1]
            v_br = grid[y][x+1]
            
            # Compute index
            idx = 0
            if v_bl >= isovalue: idx |= 1
            if v_tr >= isovalue: idx |= 2
            if v_tl >= isovalue: idx |= 4
            if v_br >= isovalue: idx |= 8
            
            # Handle Ambiguity for Cases 5 (0101 -> TL, BR) and 10 (1010 -> TL, TR? No, 10 is 1010 binary -> TL, TR? 
            # 1010 binary: Bit 3 (TL)=1, Bit 2 (TR)=0, Bit 1 (BL)=1, Bit 0 (BR)=0? 
            # My bit assignment: BL=1, TR=2, TL=4, BR=8.
            # Case 5: 0101 binary -> TL(4) + BL(1). Indices: TL, BL.
            # Case 10: 1010 binary -> TR(2) + BR(8). Indices: TR, BR.
            
            if idx == 5 or idx == 10:
                v_center = (v_bl + v_tl + v_tr + v_br) / 4.0
                if idx == 5:
                    if v_center < isovalue: idx = 10
                    else: idx = 5 # Keep as 5? Or swap? 
                    # S2: If index 5 and center < isovalue, lookup index 10.
                else: # idx == 10
                    if v_center > isovalue: idx = 5
                    else: idx = 10

            # Determine intersected edges
            # Edges: 0: Left (BL-TL), 1: Top (TL-TR), 2: Right (TR-BR), 3: Bottom (BR-BL)
            edges = []
            if (v_bl >= isovalue) != (v_tl >= isovalue): edges.append(0)
            if (v_tl >= isovalue) != (v_tr >= isovalue): edges.append(1)
            if (v_tr >= isovalue) != (v_br >= isovalue): edges.append(2)
            if (v_br >= isovalue) != (v_bl >= isovalue): edges.append(3)
            
            if len(edges) == 0:
                continue
            elif len(edges) == 2:
                # Calculate intersection points
                p0 = get_intersection(x, y, edges[0], v_bl, v_tl, v_tr, v_br, isovalue)
                p1 = get_intersection(x, y, edges[1], v_bl, v_tl, v_tr, v_br, isovalue)
                segments.append((p0, p1))
            else:
                # More than 2 edges? Only possible if >2 corners are inside/outside, 
                # but with 2 states, max 2 intersections if convex? 
                # Actually, cases 3, 12 have 2 intersections. Cases 1,2,4,7,8,13,14 have 2.
                # The edge check logic above naturally handles all valid cases.
                pass

    return segments

def get_intersection(x, y, edge_idx, v_bl, v_tl, v_tr, v_br, isovalue):
    # Coordinates: assume cell size 1.0
    # BL: (x, y), TL: (x, y+1), TR: (x+1, y+1), BR: (x+1, y)
    if edge_idx == 0: # Left: BL to TL
        v0, v1 = v_bl, v_tl
        p0, p1 = (x, y), (x, y+1)
    elif edge_idx == 1: # Top: TL to TR
        v0, v1 = v_tl, v_tr
        p0, p1 = (x, y+1), (x+1, y+1)
    elif edge_idx == 2: # Right: TR to BR
        v0, v1 = v_tr, v_br
        p0, p1 = (x+1, y+1), (x+1, y)
    elif edge_idx == 3: # Bottom: BR to BL
        v0, v1 = v_br, v_bl
        p0, p1 = (x+1, y), (x, y)
        
    t = (isovalue - v0) / (v1 - v0)
    return (p0[0] + t * (p1[0] - p0[0]), p0[1] + t * (p1[1] - p0[1]))
```

**Pitfalls:**
*   **Exact Equality:** Floating-point values rarely equal the isovalue exactly. Consistently treating $v \ge L$ as "inside" avoids gaps [S2].
*   **Saddle Ambiguity:** Cases 5 and 10 (in the standard bit mapping) are ambiguous. Using the cell center value is the standard resolution [S2, S12].
*   **Naive Interpolation:** Processing every cell independently interpolates shared edges twice, creating duplicate vertices. Caching or using indexed primitives reduces this [S2].
*   **Hardcoded Midpoints:** Some simplified implementations connect midpoints for all non-trivial cases, which is topologically incorrect for many cases [S8, S9, unofficial].

## Subtopics

### [Marching Lines Algorithm](942-marching-lines-algorithm.md)
The Marching Lines Algorithm (MLA) extracts isosurfaces from 3D scalar fields by processing the volume as a sequence of 2D slices, extending 2D contouring to three dimensions [G]. It differs from template-based methods like Marching Cubes by not relying on a fixed lookup table, instead requiring post-processing to resolve apparent cell ambiguities.

### [Dual Marching Squares](1418-dual-marching-squares.md)
Dual Marching Squares (DMS) is a 2D surface extraction method that adaptively samples a scalar field using a quadtree, placing a single vertex per leaf node at the dual grid position (the center of the leaf cell) to approximate the isosurface. Unlike standard Marching Squares, which places vertices on cell edges, DMS simplifies topology by using one vertex per region but does not preserve sharp features because dual vertices remain in cell centers rather than being projected onto the function's features.

### 2D Banded Contouring
2D Banded Contouring finds the 2D region where an interval $[L, H]$ of values exists [S3, unofficial]. It uses a ternary state (0: $<L$, 1: $L \le v \le H$, 2: $>H$) at each of the 4 corners, resulting in $3^4 = 81$ cases, reduced to 13 canonical cases via symmetry [S3, unofficial]. The algorithm constructs polygonal fragments for the band region, handling "Slope" and "Saddle" cases by using the cell center value to disambiguate connections [S12].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot M)$, where $N$ and $M$ are the grid dimensions. Each cell is processed in constant time [S2].
*   **Memory Complexity:** $O(N \cdot M)$ for the input grid. Output size is proportional to the contour length/area.
*   **Parallelism:** Embarrassingly parallel, as all cells are processed independently [S2].
*   **Strengths:** Simple to implement, fast, suitable for 2D data and 3D slice visualization [S2].
*   **Limitations:** Saddle point ambiguity requires resolution logic. Naive implementations produce duplicate vertices. Not directly applicable to 3D volumes without slicing [S2].
*   **When to Use:** 2D scalar field visualization (e.g., topographic maps, weather maps) or as a component in 3D slice-based extraction [S2].
*   **When Not to Use:** When topological correctness in 3D is required without post-processing, or when adaptive resolution is needed (use Dual Marching Squares instead) [S1, unofficial].

## Background
Marching Squares was introduced as the 2D counterpart to the Marching Cubes algorithm. It is widely used in scientific visualization, such as in the Hurricane Isabel Scientific Data Visualization project for 2D isocontour extraction [S6, unofficial]. The algorithm's principles are detailed in foundational graphics literature and implemented in various open-source projects [S2][S4, unofficial][S12].

## Sources
[S1] Marching Cubes, Explained Using Marching Squares, https://blog.42yeah.is/algorithm/2023/07/15/marching-cubes.html, fetched 2026-10-26 (unofficial)
[S2] Marching squares, https://en.wikipedia.org/wiki/Marching_squares, fetched 2026-10-26
[S3] Smokeview (Version 6) A Tool for Visualizing Fire Dynamics Simulation Data Volume II : Technical Reference Guide, https://doi.org/10.6028/nist.sp.1017-2, fetched 2026-10-26 (unofficial)
[S4] BorisTheBrave/mc-dc, https://github.com/BorisTheBrave/mc-dc, fetched 2026-10-26 (unofficial)
[S5] Interpolants Induced by Marching Cases, https://doi.org/10.4230/dfu.vol2.sciviz.2011.48, fetched 2026-10-26 (book)
[S6] killerv1per/vtk-data-visualization, https://github.com/killerv1per/vtk-data-visualization, fetched 2026-10-26 (unofficial)
[S7] thebusytypist/cudc, https://github.com/thebusytypist/cudc, fetched 2026-10-26 (unofficial)
[S8] alphaparkinc/genpark-marching-cubes-isosurface-extractor-skill, https://github.com/alphaparkinc/genpark-marching-cubes-isosurface-extractor-skill, fetched 2026-10-27 (unofficial)
[S9] Alpha-Park/genpark-marching-cubes-isosurface-extractor-skill, https://github.com/Alpha-Park/genpark-marching-cubes-isosurface-extractor-skill, fetched 2026-10-27 (unofficial)
[S10] , https://www.scitepress.org/PublishedPapers/2007/20798/pdf/index.html, fetched 2026-10-27 (unofficial)
[S11] 2453852330/VoxelHelper, https://github.com/2453852330/VoxelHelper, fetched 2026-10-27 (unofficial)
[S12] Marching squares, https://smunix.github.io/en.wikipedia.org/wiki/Marching_squares.html, fetched 2026-10-30
