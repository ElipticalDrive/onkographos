# Neighbor Finding

Also called: Octree Hashing (Neighbor-Finding)

## Overview
Neighbor finding determines the next cell in a hierarchical spatial structure (octree) intersected by a ray, by calculating a point guaranteed to lie inside that cell rather than computing a direction vector [S1]. It solves the problem of traversing from a current node to a destination node by backtracing up the octree to their common ancestor and then performing point location [S2, S3]. It differs from sibling methods like Morton Key Vertex Neighbor Iteration by relying on explicit geometric point calculation and parent-pointer traversal rather than key arithmetic [S1].

## How it works
The method consists of two phases: determining the target cell via geometric projection and locating that cell via hierarchical descent.

### Inputs and Outputs
*   **Inputs:**
    *   `start`: The coordinates of the current cell (integer vector).
    *   `dest`: The coordinates of the destination cell (integer vector).
    *   `depth`: The octree depth of the `start` cell.
    *   `parent_trace`: An array of pointers/indices to nodes, storing the path from the root to the `start` node [S2, S3].
*   **Outputs:**
    *   The index or pointer to the neighbor cell containing the destination coordinate.

### Data Structures
*   **`parent_trace`**: An array where `parent_trace[up]` holds the node index at depth `up` on the path from the root to the start node [S2, S3].
*   **`child_bit_depth`**: A precomputed array where `child_bit_depth[up]` is the bit mask corresponding to the subdivision level `up`. For a standard octree rooted at depth 0 with unit cells, this is typically $2^{max\_depth - up}$ or similar, depending on the coordinate system scaling [S2, S3].
*   **Geometric Parameters** (used in the Wyvill/Kunii variant):
    *   `CORNER[I]`: Corner coordinate of the cell.
    *   `W_B[I]`: Width/bounding parameter.
    *   `M[I]`: Scaling factor.
    *   `B[I]`: Base offset.
    *   `SIGN_M[I]`: Sign indicator for direction [S1].

### Algorithm Steps

#### Phase 1: Backtracing to Common Ancestor
The algorithm iterates from the current depth up to the root to find the highest level where the `start` and `dest` coordinates share the same parent.

1.  Initialize loop variable `up` from `depth` down to `0`.
2.  Retrieve the bit mask for the current level: `child_bit = child_bit_depth[up]`.
3.  Check if `start` and `dest` fall into the same child node at this level by comparing their bits masked by `child_bit`:
    $$ (dest.x \ \& \ child\_bit) == (start.x \ \& \ child\_bit) \ \&\& \ (dest.y \ \& \ child\_bit) == (start.y \ \& \ child\_bit) \ \&\& \ (dest.z \ \& \ child\_bit) == (start.z \ \& \ child\_bit) $$
4.  If they match, the common ancestor is found at `parent_trace[up]`. Call `point_locate(dest, up, parent_trace[up])` to find the specific neighbor and return it [S2, S3].
5.  If the loop completes without returning, the root node is the common ancestor. Check if `start` and `dest` match at the root level (using the last `child_bit` or implicit root mask). If so, call `point_locate(dest, 0, 0)` and return the result [S2, S3].

#### Phase 2: Geometric Point Calculation (Wyvill/Kunii Variant)
*Note: The core backtracing logic above is sufficient for finding the *target* cell index if `point_locate` is implemented. However, the geometric method described below is used to determine the *next* cell when traversing by ray intersection, particularly in the context of the "Neighbor Finding" algorithm described in [S1]. This phase calculates a point `POINT` guaranteed to be inside the next cell.*

1.  Initialize a rational array $T$ for coordinates $I \in \{X, Y, Z\}$.
2.  Compute the numerator and denominator of the exit $t$-value for each coordinate:
    $$ NUM(T[I]) = CORNER[I] - SIGN\_M[I] \cdot W\_B[I] - SIGN\_M[I] $$
    $$ DEN(T[I]) = M[I] $$
3.  Find the coordinate $K$ corresponding to the minimum $t$-value (the first boundary crossed). Initialize $K = X$. For $I \in \{Y, Z\}$:
    $$ \text{if } |NUM(T[K]) \cdot DEN(T[I])| > |NUM(T[I]) \cdot DEN(T[K])| \text{ then } K = I $$
    This comparison avoids floating-point division by using cross-multiplication [S1].
4.  Compute the coordinates of the point `POINT` that lies on the boundary of the next cell:
    $$ POINT[I] = M[I] \cdot \frac{NUM(T[K])}{DEN(T[K])} + B[I] \quad \forall I \in \{X, Y, Z\} $$
5.  Locate the cell containing `POINT` using a spatial search procedure (e.g., `FIND_3D_BLOCK` or `point_locate`) [S1].

### Pseudocode
The following pseudocode implements the backtracing neighbor finding algorithm [S2, S3]. It assumes `point_locate` is a separate routine that finds the node containing a coordinate given a starting node and depth.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def neighbor_find(start: Vec3i, dest: Vec3i, depth: int, parent_trace: List[int]) -> int:
    """
    Finds the neighbor cell containing 'dest' starting from 'start'.
    Uses parent_trace to backtrack to the common ancestor.
    """
    # Iterate from current depth up to root
    for up in range(depth, -1, -1):
        # Get the bit mask for the current depth level
        # child_bit_depth is a precomputed array mapping depth to bit mask
        child_bit = child_bit_depth[up]
        
        # Check if start and dest share the same parent at this level
        if ((dest.x & child_bit) == (start.x & child_bit) and
            (dest.y & child_bit) == (start.y & child_bit) and
            (dest.z & child_bit) == (start.z & child_bit)):
            
            # Found common ancestor at parent_trace[up]
            # Locate the specific child node containing dest
            return point_locate(dest, up, parent_trace[up])
            
    # If loop completes, we are at the root level (up=0 was processed in loop)
    # However, the loop range(depth, -1, -1) includes 0.
    # If we exit the loop, it means no match was found, which implies
    # dest is not in the same tree or an error occurred, 
    # but the source code explicitly checks root again.
    
    # Root node check (redundant if loop covers 0, but kept for fidelity to source)
    # Note: In the source, child_bit might be stale from the last iteration (up=0)
    if ((dest.x & child_bit) == (start.x & child_bit) and
        (dest.y & child_bit) == (start.y & child_bit) and
        (dest.z & child_bit) == (start.z & child_bit)):
        return point_locate(dest, 0, 0)
        
    return 0 # Not found
```

### Edge Cases and Pitfalls
*   **Corner Cases:** If a ray passes exactly through a cell corner, naive point location can report false intersections. For example, moving from cell 5 to 11 through the SW corner: applying `FIND_2D_BLOCK` to $(ax-1, ay)$ may incorrectly identify cell 4 if it is occupied, whereas $(ax, ay-1)$ correctly identifies cell 12 [S1].
*   **Subdivision Errors:** In subdivided quadtrees/octrees, the Wyvill/Kunii geometric calculation can yield incorrect points (e.g., $(12, 16)$ instead of $(12, 15)$), leading to routing to the wrong cell (cell 10 instead of cell 3) [S1].
*   **Direction Convention:** The algorithm relies on the convention that a ray must enter and exit a cell at two distinct points. Deviating from this (e.g., treating corner touches as valid transitions) causes errors [S1].
*   **Fix Limitations:** Simply removing the subtraction of $SIGN\_M[I]$ from the numerator does not always fix the error, as decomposing diagonal motion into axis-aligned motions can still cause false intersections [S1].

## Operating characteristics
*   **Time Complexity:** $O(D)$ where $D$ is the octree depth, for the backtracing phase. The `point_locate` phase is also $O(D)$. Total time is linear in depth.
*   **Memory Complexity:** $O(D)$ for the `parent_trace` array.
*   **Strengths:** Efficient for sparse octrees where deep traversal is required; avoids expensive direction vector normalization.
*   **Limitations:** Requires maintaining `parent_trace` pointers; geometric variant is sensitive to precision and subdivision topology [S1].

## Background
The neighbor-finding algorithm was developed for ray tracing with hierarchical spatial structures. The geometric variant (Wyvill and Kunii) was proposed as a shorter alternative to direction-vector-based traversal [S1]. However, it was shown to be incorrect for certain subdivisions, leading to the more robust point-location-based backtracing method described in [S2, S3]. It is used in interactive isosurface ray tracing of large octree volumes [S2, S3].

## Sources
[S1] Implementing ray tracing with octrees and neighbor finding, https://doi.org/10.1016/0097-8493(89)90006-x, fetched 2026-10-26
[S2] PDF Interactive Isosurface Ray Tracing of Large Octree Volumes, https://www.sci.utah.edu/~knolla/octiso-rt06.pdf, fetched 2026-10-28
[S3] Interactive isosurface ray tracing of large octree volumes, https://www.researchgate.net/publication/310054812_Interactive_isosurface_ray_tracing_of_large_octree_volumes, fetched 2026-10-29 (unofficial)
