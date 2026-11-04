# Wyvill and Kunii's Ray Tracing / Neighbor Finding Algorithm

Also called: Rational t-value Neighbor Finding, Wyvill-Kunii Neighbor Finding

## Overview
Wyvill and Kunii's algorithm is a ray-tracing method for hierarchical spatial structures (quadtrees and octrees) that determines the next intersected cell by calculating a specific point guaranteed to lie inside that cell, rather than computing a direction vector [S1]. It differs from sibling methods like the standard RAY_TRACER by using a rational t-value approach to locate the exit point directly, bypassing explicit direction vector calculation [S1].

## How it works
**Inputs:**
*   Current cell coordinates and geometric parameters: $B[I]$ (base), $M[I]$ (size), $W\_B[I]$ (width/boundary offset), $CORNER[I]$ (corner coordinate), $SIGN\_M[I]$ (sign of motion, 0 or 1) for each axis $I \in \{X, Y, Z\}$ [S1].
*   The current position in the hierarchy [S1].

**Outputs:**
*   $POINT[I]$: The coordinates of a point inside the next intersected cell [S1].
*   The identity of the next cell, determined by locating $POINT$ using a spatial search procedure (e.g., `FIND_3D_BLOCK`) [S1].

**Data Structures:**
*   `T`: A rational array indexed by $\{X, Y, Z\}$, storing numerator and denominator pairs for t-values [S1].
*   `POINT`: An array of floating-point coordinates for the exit point [S1].
*   Lookup tables (not algorithmic steps but required for context in related neighbor finding):
    *   `EDGF_DIR`: Maps normal plane and signs to edge direction $E$ [S1].
    *   `VERTEX_DIR`: Maps signs to vertex direction $V$ (e.g., LDB, RUF) [S1].
    *   `CHANGE`: Maps direction vectors to 3D components [S1].

**Steps:**
1.  **Initialize Rational T-Values:** For each coordinate $I \in \{X, Y, Z\}$, compute the numerator and denominator of the exit t-value $T[I]$ [S1]:
    $$NUM(T[I]) = CORNER[I] - SIGN\_M[I] \cdot W\_B[I] - SIGN\_M[I]$$
    $$DEN(T[I]) = M[I]$$
    [S1]

2.  **Find Minimum Axis:** Identify the axis $K$ corresponding to the minimum t-value. Initialize $K = X$. For each $I \in \{Y, Z\}$, compare $T[K]$ and $T[I]$ using cross-multiplication to avoid division:
    ```python
    if abs(NUM(T[K]) * DEN(T[I])) > abs(NUM(T[I]) * DEN(T[K])):
        K = I
    ```
    [S1]

3.  **Compute Next Cell Point:** Calculate the coordinates of the point $POINT$ using the minimum t-value $T[K]$ [S1]:
    $$POINT[I] = M[I] \cdot \frac{NUM(T[K])}{DEN(T[K])} + B[I] \quad \text{for all } I \in \{X, Y, Z\}$$
    [S1]

4.  **Locate Next Cell:** Use a spatial search procedure (analogous to `FIND_3D_BLOCK`) to identify the cell containing $POINT$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def wyvill_kunii_next_cell(B, M, W_B, CORNER, SIGN_M):
    """
    B: array [X, Y, Z] base coordinates
    M: array [X, Y, Z] cell sizes
    W_B: array [X, Y, Z] width/boundary offsets
    CORNER: array [X, Y, Z] corner coordinates
    SIGN_M: array [X, Y, Z] motion signs (0 or 1)
    Returns: POINT array [X, Y, Z]
    """
    # Step 1: Initialize rational t-values
    NUM_T = {}
    DEN_T = {}
    for I in ['X', 'Y', 'Z']:
        NUM_T[I] = CORNER[I] - SIGN_M[I] * W_B[I] - SIGN_M[I]
        DEN_T[I] = M[I]

    # Step 2: Find axis K with minimum t-value
    K = 'X'
    for I in ['Y', 'Z']:
        # Compare T[K] < T[I] => NUM(K)/DEN(K) < NUM(I)/DEN(I)
        # Cross multiply to avoid division: NUM(K)*DEN(I) < NUM(I)*DEN(K)
        # We want the minimum, so if T[I] is smaller, update K
        # Condition in source: if abs(NUM(K)*DEN(I)) > abs(NUM(I)*DEN(K)) then K=I
        # This implies we are looking for the smallest magnitude t-value
        if abs(NUM_T[K] * DEN_T[I]) > abs(NUM_T[I] * DEN_T[K]):
            K = I

    # Step 3: Compute POINT using the minimum t-value T[K]
    POINT = {}
    for I in ['X', 'Y', 'Z']:
        # Rational division
        t_val = NUM_T[K] / DEN_T[K]
        POINT[I] = M[I] * t_val + B[I]

    return POINT
```
[G] Note: The source text implies `NUM` and `DEN` are stored in a rational array `T`. The pseudocode separates them for clarity. The comparison logic uses absolute values as per the source, which handles potential negative t-values or specific coordinate system conventions described in the original paper.

**Edge Cases and Pitfalls:**
*   **Corner Passing:** If a ray passes through a cell corner (e.g., SW corner of cell 5 to cell 11), applying a 2D block finder to $(ax-1, ay)$ may report a false intersection if cell 4 is fully occupied. Using $(ax, ay-1)$ avoids this [S1].
*   **Subdivision Errors:** If a cell (e.g., cell 3) is subdivided like the SE quadrant of a quadtree, the algorithm may compute an incorrect point (e.g., $(12, 16)$ instead of $(12, 15)$), routing to the wrong cell (cell 10 instead of cell 3) [S1].
*   **Fix Limitations:** Removing the subtraction of $SIGN\_M[I]$ from the numerator does not always fix the error. Decomposing motion into separate S and W directions can still cause false intersections due to distinct entry/exit conventions [S1].
*   **Simultaneous Boundaries:** Wyvill and Kunii claim that when a ray reaches multiple boundaries simultaneously, arbitrarily picking one boundary perpendicular to motion is sufficient. This claim is shown to be incorrect for certain subdivisions [S1].

## Operating characteristics
*   **Complexity:** Not given in the sources.
*   **Performance:** Not given in the sources.
*   **Strengths:** Simpler structure than neighbor-finding algorithms that compute exact direction vectors; avoids explicit direction vector calculation [S1].
*   **Limitations:** Prone to errors in specific hierarchical subdivisions (e.g., SE quadrant subdivisions) and corner cases; the claim of robustness for simultaneous boundary crossings is incorrect [S1].

## Background
The algorithm was proposed by Wyvill and Kunii [S1]. It is discussed in the context of implementing ray tracing with octrees and neighbor finding, where it is compared against a more robust RAY_TRACER algorithm [S1].

## Sources
[S1] Implementing ray tracing with octrees and neighbor finding, https://doi.org/10.1016/0097-8493(89)90006-x, fetched 2026-10-26
