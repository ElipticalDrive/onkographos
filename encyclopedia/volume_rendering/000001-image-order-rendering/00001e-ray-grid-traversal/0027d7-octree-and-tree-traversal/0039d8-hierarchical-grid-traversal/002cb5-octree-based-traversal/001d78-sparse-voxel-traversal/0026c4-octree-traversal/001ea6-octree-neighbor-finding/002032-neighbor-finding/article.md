# Neighbor Finding

Also called: Octree Hashing (Neighbor-Finding)

## Overview
Neighbor finding locates the octree node containing a destination coordinate (`dest`) starting from a known origin node (`start`) and its depth. It solves the problem of traversing between arbitrary nodes in a sparse voxel octree without storing explicit parent pointers, by backtracing up the hierarchy to find a common ancestor and then descending to the target. It differs from sibling methods like [Morton Key Vertex Neighbor Iteration](3318-morton-key-vertex-neighbor-iteration.md) by operating on coordinate bits directly rather than Morton keys, and by supporting arbitrary (non-adjacent) neighbor lookups through a two-phase ascent-descent process.

## How it works
**Inputs and Outputs:**
*   `start`: The 3D integer coordinates of the origin node.
*   `dest`: The 3D integer coordinates of the destination node.
*   `depth`: The depth level of the `start` node in the octree.
*   `parent_trace`: An array of node indices (or pointers) representing the path from the root to the `start` node. `parent_trace[i]` holds the index of the node at depth `i`.
*   **Output**: The index (or pointer) of the node containing `dest`.

**Data Structures:**
*   `child_bit_depth`: A precomputed array where `child_bit_depth[d]` is a bitmask isolating the bits of the coordinate at depth `d`. For a standard octree where depth 0 is the root, the mask at depth `d` typically isolates the $d$-th bit of the coordinate (e.g., if coordinates are 32-bit integers, `child_bit_depth[d] = 1 << (31 - d)` or similar, depending on bit ordering). [S1][S2, unofficial]
*   `parent_trace`: An array of size `max_depth + 1`. It is filled by the ray traversal algorithm as it descends the tree. [S1][S3, unofficial][S4]

**Algorithm Steps:**
1.  Iterate from the current `depth` down to 0. Let the loop variable be `up`.
2.  Retrieve the bitmask for the current level: `child_bit = child_bit_depth[up]`.
3.  Check if `start` and `dest` share the same parent at this level. This is done by masking both coordinates with `child_bit` and comparing the results.
    *   Condition: `(dest.x & child_bit) == (start.x & child_bit)` AND `(dest.y & child_bit) == (start.y & child_bit)` AND `(dest.z & child_bit) == (start.z & child_bit)`.
4.  If the condition is true, `start` and `dest` are in the same child of the node at level `up + 1` (or are the same node if `up` is the leaf level). The common ancestor is `parent_trace[up]`.
5.  Call `point_locate(dest, up, parent_trace[up])` to descend from the common ancestor to the leaf node containing `dest`. Return the result.
6.  If the loop completes without returning, the nodes are in different subtrees of the root. Check if they are in the same child of the root (using the mask from the last iteration, or a root-level mask).
7.  If they match at the root level, call `point_locate(dest, 0, 0)` (assuming root index is 0).
8.  If they do not match, the destination is outside the octree bounds. Return 0 (or null). [S1][S2, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def neighbor_find(start: Vec3i, dest: Vec3i, depth: int, parent_trace: List[int], child_bit_depth: List[int]) -> int:
    """
    Finds the node index for 'dest' starting from 'start' at 'depth'.
    Uses parent_trace to ascend without explicit parent pointers.
    """
    # Iterate from the depth of the start node down to 0
    for up in range(depth, -1, -1):
        # Get the bitmask for the current depth level
        child_bit = child_bit_depth[up]
        
        # Check if start and dest share the same parent at this level
        # by comparing the bits corresponding to this depth
        if ((dest.x & child_bit) == (start.x & child_bit) and
            (dest.y & child_bit) == (start.y & child_bit) and
            (dest.z & child_bit) == (start.z & child_bit)):
            
            # Found common ancestor at level 'up'
            # Descend from the common ancestor to find the leaf
            return point_locate(dest, up, parent_trace[up])
            
    # If loop completes, check root level match
    # Note: child_bit from the last iteration (up=0) is used
    if ((dest.x & child_bit) == (start.x & child_bit) and
        (dest.y & child_bit) == (start.y & child_bit) and
        (dest.z & child_bit) == (start.z & child_bit)):
        
        # Common ancestor is the root (index 0)
        return point_locate(dest, 0, 0)
        
    # Destination is outside the octree bounds
    return 0
```

**Parameters:**
*   `child_bit_depth`: Typically precomputed. For a 32-bit coordinate system and depth $d$ (where 0 is root), the bit mask is $2^{31-d}$. [S1][S2, unofficial]

**Edge Cases and Pitfalls:**
*   **Parent Pointers**: Parent pointers are omitted from the octree nodes to minimize memory footprint. The `parent_trace` array must be maintained by the caller (typically the ray traversal algorithm). [S1][S3, unofficial][S4]
*   **Non-Adjacent Neighbors**: This algorithm finds the node containing `dest`, which may not be an immediate neighbor of `start`. It is a general neighbor-finding routine. [S4]
*   **Root Handling**: The root node is typically at index 0 in the `parent_trace` array. [S1][S2, unofficial]
*   **Out of Bounds**: If `dest` is not within the octree's spatial bounds, the algorithm returns 0 (or a null indicator). [S1][S2, unofficial]

## Subtopics

### [Wyvill and Kunii's Ray Tracing / Neighbor Finding Algorithm](3876-wyvill-and-kunii-s-ray-tracing-neighbor-finding-algorithm.md)
Wyvill and Kunii's algorithm is a ray-tracing method for hierarchical spatial structures (quadtrees and octrees) that determines the next intersected cell by calculating a specific point guaranteed to lie inside that cell, rather than computing a direction vector. It differs from sibling methods like the standard RAY_TRACER by using a rational t-value approach to locate the exit point directly, bypassing explicit direction vector calculation.

## Operating characteristics
*   **Time Complexity**: The ascent phase is $O(D)$ where $D$ is the depth of the octree. The descent phase (`point_locate`) is also $O(D)$. Total time is $O(D)$.
*   **Memory Complexity**: $O(1)$ additional memory beyond the `parent_trace` array, which is $O(D)$. No parent pointers are stored in the node structure itself. [S1][S3, unofficial][S4]
*   **Strengths**: Memory efficient due to lack of parent pointers. Simple bitwise operations for ascent. Supports arbitrary neighbor lookups.
*   **Limitations**: Requires the `parent_trace` array to be maintained by the traversal algorithm. Not optimized for immediate neighbor lookups (e.g., XOR-based methods might be faster for adjacent voxels). [S4]
*   **When to Use**: When memory is constrained and parent pointers are undesirable. When the traversal algorithm already maintains a path trace. [S1][S3, unofficial][S4]
*   **When Not to Use**: When immediate neighbor lookups are frequent and a faster XOR-based method is available. [S4]

## Background
The method originates from interactive isosurface ray tracing of large octree volumes. It was developed to optimize memory usage in sparse voxel octrees by removing explicit parent pointers, relying instead on the traversal context to provide parent indices. [S1][S2, S3, unofficial][S4]

## Sources
[S1] PDF Interactive Isosurface Ray Tracing of Large Octree Volumes, https://www.sci.utah.edu/~knolla/octiso-rt06.pdf, fetched 2026-10-28
[S2] Interactive isosurface ray tracing of large octree volumes, https://www.researchgate.net/publication/310054812_Interactive_isosurface_ray_tracing_of_large_octree_volumes, fetched 2026-10-29 (unofficial)
[S3] RAY TRACING IMPLICIT SURFACES FOR INTERACTIVE ..., https://www.sci.utah.edu/~knolla/thesis.pdf, fetched 2026-10-29 (unofficial)
[S4] Interactive Isosurface Ray Tracing of Large Octree Volumes, https://www.sci.utah.edu/~wald/Publications/2006/OctIso/octiso.pdf, fetched 2026-10-27
