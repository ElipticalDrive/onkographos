# Marching Squares (Isoband)


## Overview
Marching Squares (Isoband) is a surface extraction algorithm that generates filled contour bands (isobands) between an upper and lower threshold value, producing polygonal fragments from a 2D scalar field [S1]. It extends the standard Marching Squares algorithm by classifying grid cells into three states (below, within, or above the band) rather than two, allowing for the creation of continuous filled regions rather than just isoline segments [S1].

## How it works
**Inputs:** A 2D scalar field $V$ defined on a regular grid; a lower threshold $T_{low}$ and an upper threshold $T_{high}$ defining the band interval $[T_{low}, T_{high}]$.
**Outputs:** A list of polygonal fragments (polygons with 3–8 sides) representing the filled isobands.

**Data Structures:**
*   **Cell Index:** A ternary integer (base-3) constructed from 4 "trit" values, one for each corner of the square cell.
*   **Trit Value:**
    *   $0$: Corner value $< T_{low}$
    *   $1$: $T_{low} \le$ Corner value $\le T_{high}$
    *   $2$: Corner value $> T_{high}$
*   **Index Construction:** The trits are appended clockwise around the cell. The most-significant trit corresponds to the top-left corner, and the least-significant trit corresponds to the bottom-left corner. There are $3^4 = 81$ possible indices.

**Algorithm Steps:**
1.  **Classify Corners:** For each cell, compare the scalar value at each of the four corners against $T_{low}$ and $T_{high}$ to determine the trit for each corner.
2.  **Compute Index:** Construct the 4-digit ternary index $I$ from the trits.
3.  **Determine Category:**
    *   **Empty:** If $I = 0$ (0000) or $I = 80$ (2222 in base-3, representing 2222$_3$). No geometry is generated.
    *   **Not Empty:** All other indices.
4.  **Generate Geometry for Not Empty Cells:**
    *   **Slope Cases:** Build a single polygon fragment with 3–7 sides. The algorithm walks around the cell, adding corner positions that are within the band and interpolating along relevant edges to connect the points. The specific connectivity is determined by the index $I$.
    *   **Saddle Cases:** These cases require disambiguation because the topology is not determined solely by the corner values.
        1.  Calculate the average cell center value $V_c = \frac{1}{4}\sum V_{corners}$.
        2.  Use $I$ and $V_c$ to determine the topology.
        3.  Build 1 or 2 polygonal fragments with a total of 6, 7, or 8 sides.
5.  **Edge Interpolation:** For edges where one endpoint is within the band and the other is not, or where endpoints are on opposite sides of the band boundaries, perform linear interpolation to find the intersection points. The intersection point $P$ on an edge between corners with values $V_1$ and $V_2$ and positions $P_1$ and $P_2$ is calculated as:
    $$ P = P_1 + \frac{T - V_1}{V_2 - V_1} (P_2 - P_1) $$
    where $T$ is the relevant threshold ($T_{low}$ or $T_{high}$) depending on which boundary is being crossed.

**Pitfalls and Edge Cases:**
*   **Missing 6-Sided Saddle:** A specific 6-sided saddle case is omitted because the central value configuration required for it cannot occur in a valid scalar field interpolation [S1].
*   **Omitted 7-Sided Saddle:** A valid 7-sided saddle case exists where the central value is dominated by a single extreme value. The resulting geometric structure is too complex to fit the simple model of two convex fragments. This case is merged with the case where the central value is within the band. Linear interpolation in this merged case produces a plausible single heptagon [S1].
*   **Planarity:** In 3D space, a quadrilateral embedded in 3D is not necessarily planar, requiring a choice of geometric interpolation scheme for banded surfaces, though the 2D algorithm assumes planarity [S1].

## Subtopics

### Marching Squares Performance Optimization
This variant focuses on reducing computational overhead and output size. It caches interpolation results to avoid redundant calculations; a single-threaded serial version only needs to cache interpolated results for one row of the input grid [S1]. It reduces output size by using indexed geometric primitives, creating an array of 2D vertices and specifying polygons with short integer offsets into the array [S1]. It is designed as an embarrassingly parallel algorithm assuming shared read-only input scalar field and shared append-only geometry output stream [S1].

## Operating characteristics
*   **Complexity:** A naive implementation performs linear interpolation four times per edge for isobands, resulting in 4 copies of vertices for polygons [S1].
*   **Parallelism:** The algorithm is embarrassingly parallel because all cells are processed independently [S1].
*   **Memory:** A naive implementation produces 4 copies of 2D vertices for polygons (isobands) due to independent cell processing [S1]. Indexed primitives can reduce this significantly.
*   **Strengths:** Simple to implement; produces filled regions; embarrassingly parallel.
*   **Limitations:** Naive implementation has high vertex duplication; saddle cases require center value calculation and complex logic; omitted cases may lead to topological inaccuracies in specific scalar field configurations [S1].

## Background
The Marching Squares algorithm was originally developed for isoline extraction. The isoband variant extends this to filled contours. The algorithm is widely used in geographic information systems (GIS) for contour mapping and in scientific visualization for scalar field analysis. The specific handling of saddle cases and the omission of certain complex topologies were detailed in discussions of substitope algorithms and geometric design [S1, S2].

## Sources
[S1] Marching squares, https://smunix.github.io/en.wikipedia.org/wiki/Marching_squares.html, fetched 2026-10-30
[S2] Banks, D. C. (2004). "Counting cases in substitope algorithms". IEEE Trans. Visual. Comp. Graphics. 10 (4): 371–384. doi:10.1109/TVCG.2004.6.
