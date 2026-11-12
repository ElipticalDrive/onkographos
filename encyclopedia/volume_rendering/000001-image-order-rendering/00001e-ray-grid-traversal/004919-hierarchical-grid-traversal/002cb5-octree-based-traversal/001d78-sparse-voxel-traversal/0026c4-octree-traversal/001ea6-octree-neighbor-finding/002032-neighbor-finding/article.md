# Neighbor Finding

Also called: Octree Hashing (Neighbor-Finding)

## Overview
Neighbor finding locates the octree node containing a destination coordinate relative to a start coordinate by backtracing up the hierarchy to a common ancestor and performing point location [S1][S2, S3, unofficial]. It differs from sibling methods like [Morton Key Vertex Neighbor Iteration](3318-morton-key-vertex-neighbor-iteration.md) by relying on explicit coordinate masking and parent trace arrays rather than arithmetic on Morton keys [S3, unofficial]. This method is essential for sparse voxel traversal where direct neighbor access is not available via hashing [S1].

## How it works
**Inputs:**
*   `start`: The 3D integer coordinates of the origin node [S1][S2, unofficial].
*   `dest`: The 3D integer coordinates of the desired neighbor [S1][S2, unofficial].
*   `depth`: The depth of the `start` node in the octree [S1][S2, unofficial].
*   `parent_trace[]`: An array storing the indices of parent nodes encountered during traversal, indexed by depth [S1][S2, S3, unofficial].

**Outputs:**
*   The index of the node containing `dest` [S1][S2, unofficial].

**Data Structures:**
*   `child_bit_depth[]`: An array where `child_bit_depth[up]` contains the bitmask for the bit position corresponding to the child index at depth `up`. For a standard octree, this is typically $2^{2 \times up}$ or similar depending on Morton ordering, but the algorithm treats it as a generic mask [S1][S2, unofficial].
*   `parent_trace[]`: Array of integers. `parent_trace[up]` holds the index of the node at depth `up` that is an ancestor of the current node [S1][S2, unofficial].

**Algorithm Steps:**
1.  Iterate from the current depth down to 0: `for up = depth down to 0` [S1][S2, unofficial].
2.  Retrieve the child bit mask for the current level: `child_bit = child_bit_depth[up]` [S1][S2, unofficial].
3.  Check if `start` and `dest` share the same bits at this depth level for all three dimensions [S1][S2, S3, unofficial]. This is done by masking both coordinates with `child_bit` and comparing for equality [S3, unofficial].
    *   Condition: `(dest.x & child_bit) == (start.x & child_bit)` AND `(dest.y & child_bit) == (start.y & child_bit)` AND `(dest.z & child_bit) == (start.z & child_bit)` [S1][S2, unofficial].
4.  If the condition is true, a common parent is found at this level. Call `point_locate(dest, up, parent_trace[up])` to find the specific leaf node at `dest` starting from the identified parent [S1][S2, unofficial]. Return the result [S1][S2, unofficial].
5.  If the loop completes without returning, check the root level match using the same masking logic (note: `child_bit` from the last iteration `up=0` is used) [S1][S2, unofficial].
6.  If the root check passes, call `point_locate(dest, 0, 0)` (assuming index 0 is the root) [S1][S2, unofficial].
7.  If no common parent is found (or `point_locate` fails), return 0 or an error indicator [S1][S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def neighbor_find(start: Vec3i, dest: Vec3i, depth: int, parent_trace: List[int], child_bit_depth: List[int]) -> int:
    """
    Finds the node containing dest starting from start by backtracing to a common ancestor.
    
    Args:
        start: Coordinates of the origin node.
        dest: Coordinates of the destination node.
        depth: Depth of the start node.
        parent_trace: Array of parent node indices indexed by depth.
        child_bit_depth: Array of bitmasks for each depth level.
        
    Returns:
        Index of the node containing dest, or 0 if not found.
    """
    # Iterate from current depth down to 0
    for up in range(depth, -1, -1):
        # Retrieve child bit mask for current level
        child_bit = child_bit_depth[up]
        
        # Check if start and dest share the same bits at this depth
        if ((dest.x & child_bit) == (start.x & child_bit) and
            (dest.y & child_bit) == (start.y & child_bit) and
            (dest.z & child_bit) == (start.z & child_bit)):
            
            # Common parent found; perform point location from this parent
            return point_locate(dest, up, parent_trace[up])
            
    # Root node check (if loop completes without returning)
    # Note: child_bit is from the last iteration (up=0)
    if ((dest.x & child_bit) == (start.x & child_bit) and
        (dest.y & child_bit) == (start.y & child_bit) and
        (dest.z & child_bit) == (start.z & child_bit)):
        
        return point_locate(dest, 0, 0)
        
    return 0
```

**Parameters:**
*   `child_bit_depth`: Implementation-specific. Typically powers of 2 or Morton bit positions. Not given in the sources. [G: In standard Morton ordering, `child_bit_depth[up]` is often $2^{2 \times up}$ if bits are packed sequentially, or specific masks depending on the interleaving scheme.]

**Edge Cases and Pitfalls:**
*   **Parent Pointers:** Explicit parent pointers are omitted from node structures to minimize memory footprint [S3, unofficial]. Parent indices are provided by the ray traversal algorithm filling the `parent_trace[]` array [S3, unofficial].
*   **Root Handling:** The algorithm explicitly handles the case where the common ancestor is the root node [S1][S2, unofficial].
*   **Optimization:** Static polymorphism via templates can be used for X, Y, Z dimensions to reduce equality tests to one or two per iteration for neighbors along designated axes [S2, unofficial].

## Subtopics

### [Wyvill and Kunii's Ray Tracing / Neighbor Finding Algorithm](3876-wyvill-and-kunii-s-ray-tracing-neighbor-finding-algorithm.md)
Uses a rational t-value approach to locate the exit point directly, bypassing explicit direction vector calculation.

### [Octree Neighbor Finding without Parent Pointers](5593-octree-neighbor-finding-without-parent-pointers.md)
Omits explicit parent pointers, using an `index trace[]` array and bitwise AND masking with integer equality checks to identify common parents.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the depth of the octree, $O(D)$, where $D$ is the maximum depth [S1][S2, unofficial].
*   **Memory Complexity:** $O(D)$ for the `parent_trace` array, which is reused during traversal [S1][S2, S3, unofficial].
*   **Strengths:** Minimizes memory footprint by avoiding explicit parent pointers in node structures [S3, unofficial]. Simple to implement using coordinate masking [S3, unofficial].
*   **Limitations:** Requires a valid `parent_trace` array populated by a prior traversal step [S3, unofficial]. Performance depends on the depth of the common ancestor [S1].
*   **When to use:** When traversing sparse octrees where neighbor access is needed but direct hashing is insufficient or expensive [S1][S3, unofficial].

## Background
Originated in the context of interactive isosurface ray tracing of large octree volumes [S1][S2, S3, unofficial]. The method was developed to optimize ray traversal by efficiently finding neighboring cells without storing redundant parent pointers [S3, unofficial]. It builds on earlier work by Frisken and Perry, who required foreknowledge of neighbor direction (left/right) on midplanes [S3, unofficial]. This implementation omits that distinction, simplifying the logic by using coordinate masking [S3, unofficial].

## Sources
[S1] PDF Interactive Isosurface Ray Tracing of Large Octree Volumes, https://www.sci.utah.edu/~knolla/octiso-rt06.pdf, fetched 2026-10-28
[S2] Interactive isosurface ray tracing of large octree volumes, https://www.researchgate.net/publication/310054812_Interactive_isosurface_ray_tracing_of_large_octree_volumes, fetched 2026-10-29 (unofficial)
[S3] RAY TRACING IMPLICIT SURFACES FOR INTERACTIVE ..., https://www.sci.utah.edu/~knolla/thesis.pdf, fetched 2026-10-29 (unofficial)
