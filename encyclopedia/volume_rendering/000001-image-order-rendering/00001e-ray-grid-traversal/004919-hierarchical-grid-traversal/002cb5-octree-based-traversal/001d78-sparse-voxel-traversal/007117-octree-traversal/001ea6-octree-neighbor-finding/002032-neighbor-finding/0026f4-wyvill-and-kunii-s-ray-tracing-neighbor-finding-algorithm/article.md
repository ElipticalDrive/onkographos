# Wyvill and Kunii's Ray Tracing / Neighbor Finding Algorithm

Also called: Rational t-value Neighbor Finding, Wyvill-Kunii Neighbor Finding

## Overview
Wyvill and Kunii's algorithm is a ray-tracing method for hierarchical spatial structures (quadtrees and octrees) that determines the next intersected cell by calculating a point guaranteed to lie inside it, rather than computing a direction vector [S1]. It sets itself apart from its parent [Neighbor Finding](2198-neighbor-finding.md) by using a spatial search procedure (similar to `FIND_3D_BLOCK`) to locate the calculated point, and from [Octree Neighbor Finding without Parent Pointers](5593-octree-neighbor-finding-without-parent-pointers.md) by avoiding the explicit neighbor pointer traversal in favor of geometric point location.

## How it works
The algorithm computes the coordinates of a point (`POINT`) that is purportedly guaranteed to be in the next cell intersected by the ray, and then locates this point using a spatial search procedure similar to `FIND_3D_BLOCK` [S1]. It does not compute a direction vector [S1].

**Inputs:**
- Ray parameters defining the exit from the current cell.
- Geometric parameters: `CORNER[I]`, `W_B[I]`, `M[I]`, `B[I]` for each coordinate $I \in \{X, Y, Z\}$ [S1].
- Sign indicators: `SIGN_M[I]` for each coordinate $I \in \{X, Y, Z\}$ [S1].

**Outputs:**
- `POINT`: A 3D coordinate vector guaranteed to lie in the next cell [S1].

**Data Structures:**
- `T`: A rational array of size 3 (for X, Y, Z). Each element stores a numerator and a denominator.
- `POINT`: A 3D vector.
- `axis`: Variables `I` and `K` to track coordinate indices.

**Formulas:**
1.  **Rational t-value Numerator:**
    $$NUM(T[I]) = CORNER[I] - SIGN\_M[I] \cdot W\_B[I] - SIGN\_M[I]$$
2.  **Rational t-value Denominator:**
    $$DEN(T[I]) = M[I]$$
3.  **Point Coordinate:**
    $$POINT[I] = M[I] \cdot \frac{NUM(T[K])}{DEN(T[K])} + B[I]$$
    Where $K$ is the coordinate index corresponding to the minimum t-value [S1].

**Parameters:**
- `CORNER[I]`, `W_B[I]`, `M[I]`, `B[I]`: Geometric parameters used in rational t-value and point coordinate calculations. Exact definitions are not provided in the sources [S1].
- `SIGN_M[I]`: Sign indicators, typically 0 or 1, representing the direction of motion along axis $I$ [S1].

**Steps:**
1.  Initialize a rational array `T` for coordinates X, Y, Z.
2.  For each coordinate $I \in \{X, Y, Z\}$:
    a. Compute $NUM(T[I]) = CORNER[I] - SIGN\_M[I] \cdot W\_B[I] - SIGN\_M[I]$.
    b. Compute $DEN(T[I]) = M[I]$.
3.  Initialize $K = X$.
4.  For each coordinate $I \in \{Y, Z\}$:
    a. If $|NUM(T[K]) \cdot DEN(T[I])| > |NUM(T[I]) \cdot DEN(T[K])|$, then set $K = I$.
    b. This comparison identifies the coordinate $K$ corresponding to the minimum t-value without division [S1].
5.  For each coordinate $I \in \{X, Y, Z\}$:
    a. Compute $POINT[I] = M[I] \cdot \frac{NUM(T[K])}{DEN(T[K])} + B[I]$.
6.  Locate the computed `POINT` using a spatial search procedure similar to `FIND_3D_BLOCK` to identify the next cell in the hierarchy [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def wyvill_kunii_next_cell(CORNER, W_B, M, B, SIGN_M):
    """
    Computes the next cell point using Wyvill and Kunii's rational t-value method.
    
    Args:
        CORNER: Array of 3 floats [cx, cy, cz]
        W_B: Array of 3 floats [wx, wy, wz]
        M: Array of 3 floats [mx, my, mz]
        B: Array of 3 floats [bx, by, bz]
        SIGN_M: Array of 3 integers [sx, sy, sz] (0 or 1)
        
    Returns:
        POINT: Array of 3 floats [px, py, pz]
    """
    # Step 1 & 2: Initialize rational t-values
    NUM = [0.0] * 3
    DEN = [0.0] * 3
    
    for i in range(3):
        # NUM(T[I]) = CORNER[I] - SIGN_M[I] * W_B[I] - SIGN_M[I]
        NUM[i] = CORNER[i] - SIGN_M[i] * W_B[i] - SIGN_M[i]
        # DEN(T[I]) = M[I]
        DEN[i] = M[i]
        
    # Step 3 & 4: Find coordinate K with minimum t-value
    # K corresponds to the index with the smallest NUM/DEN ratio.
    # We compare cross-products to avoid division: |NUM[K]*DEN[I]| > |NUM[I]*DEN[K]|
    K = 0  # Initialize K to X (index 0)
    
    for i in [1, 2]:  # Iterate over Y (1) and Z (2)
        # abs(NUM[T[K]] * DEN[T[I]]) > abs(NUM[T[I]] * DEN[T[K]])
        lhs = abs(NUM[K] * DEN[i])
        rhs = abs(NUM[i] * DEN[K])
        if lhs > rhs:
            K = i
            
    # Step 5: Compute POINT coordinates
    POINT = [0.0] * 3
    for i in range(3):
        # POINT[I] = M[I] * (NUM(T[K]) / DEN(T[K])) + B[I]
        # Note: Division by zero in DEN[K] is a potential pitfall if M[K] is 0.
        # The sources do not specify handling for M[K]=0. [G]
        t_val = NUM[K] / DEN[K]
        POINT[i] = M[i] * t_val + B[i]
        
    return POINT
```

**Edge Cases and Pitfalls:**
- **Corner Passing:** If a ray passes through a cell corner (e.g., SW corner of cell 5 to cell 11), applying `FIND_2D_BLOCK` to $(ax-1, ay)$ may report a false object intersection when cell 4 is fully occupied, whereas $(ax, ay-1)$ avoids it [S1].
- **Subdivision Errors:** When a cell is subdivided (e.g., cell 3 subdivided like the SE quadrant of a quadtree), the algorithm may compute `POINT = (12, 16)` instead of the correct `(12, 15)`, causing `FIND_2D_BLOCK` to route to the wrong cell (e.g., cell 10 instead of cell 3) [S1].
- **Sign Subtraction Fix:** Removing the subtraction of `SIGN_M[I]` from the numerator of $t$ for coordinate $I$ does not always fix the error. Decomposing motion (e.g., SW into S and W) can still cause false intersections due to distinct entry/exit conventions [S1].
- **Simultaneous Boundaries:** Wyvill and Kunii claim that exact direction computation is unnecessary and that arbitrarily picking a boundary perpendicular to motion when multiple boundaries are reached simultaneously is sufficient. This claim is shown to be incorrect for certain subdivisions [S1].

## Subtopics

### [Neighbour List Octree Traversal](21149-neighbour-list-octree-traversal.md)
Neighbour List Octree Traversal is a ray-tracing method for sparse voxel octrees that determines the next intersected cell by consulting precomputed lists of adjacent cells, rather than performing geometric intersection tests or restarting a search from the root. It sets itself apart from the parent [Wyvill and Kunii's Ray Tracing / Neighbor Finding Algorithm](3876-wyvill-and-kunii-s-ray-tracing-neighbor-finding-algorithm.md) by replacing the calculation of a guaranteed interior point with a direct lookup of potential neighbours, and from the [Top-down Method](52-top-down-method.md) by avoiding the $O(depth)$ traversal cost for every cell crossing.

## Operating characteristics
- **Time Complexity:** Not given in the sources. The algorithm involves a constant number of arithmetic operations per step plus the cost of the spatial search procedure (`FIND_3D_BLOCK`), which depends on the octree depth and structure.
- **Memory Complexity:** Not given in the sources. The algorithm requires storage for the rational array `T` and the `POINT` vector.
- **Strengths:** The algorithm is simpler in structure than methods that compute exact direction vectors, as it relies on point location [S1].
- **Limitations:** The algorithm is prone to errors in specific subdivision cases (e.g., SE quadrant subdivision) and when rays pass through cell corners, leading to false intersections or wrong cell transitions [S1]. The claim that exact direction computation is unnecessary is incorrect for certain cases [S1].
- **When to use:** Suitable for hierarchical spatial structures where point location is efficient and the specific subdivision patterns do not trigger the known pitfalls.
- **When not to use:** When high precision is required in complex subdivision scenarios or when rays frequently pass through cell corners or edges, as the algorithm may produce incorrect results [S1].

## Background
The algorithm was proposed by Wyvill and Kunii [S1]. It is described in the context of implementing ray tracing with octrees and neighbor finding [S1]. The algorithm is noted to be shorter than the author's own `RAY_TRACER` algorithm but has a similar structure, adhering to different conventions [S1].

## Sources
[S1] Implementing ray tracing with octrees and neighbor finding, https://doi.org/10.1016/0097-8493(89)90006-x, fetched 2026-10-26
