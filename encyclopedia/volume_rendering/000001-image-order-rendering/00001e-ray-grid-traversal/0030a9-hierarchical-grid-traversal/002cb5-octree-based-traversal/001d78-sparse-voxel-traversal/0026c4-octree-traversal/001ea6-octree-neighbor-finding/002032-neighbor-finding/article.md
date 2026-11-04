# Neighbor Finding

Also called: Octree Hashing (Neighbor-Finding)

## Overview
Neighbor finding determines the next cell in a hierarchical spatial structure (octree) intersected by a ray by backtracing from a start coordinate to the common ancestor with a destination coordinate, then performing point location to retrieve the neighbor [S1][S2, unofficial]. It differs from sibling methods like Morton Key Vertex Neighbor Iteration by relying on explicit parent-pointer traversal and coordinate bit-masking rather than Morton key arithmetic [S1].

## How it works
The method consists of a single backtracing phase that locates the common ancestor of two coordinates, followed by a point location step to retrieve the target cell [S1][S2, unofficial].

### Inputs and Outputs
*   **Inputs:**
    *   `start`: The integer coordinates of the current cell (`Vec3i`) [S1][S2, unofficial].
    *   `dest`: The integer coordinates of the destination cell (`Vec3i`) [S1][S2, unofficial].
    *   `depth`: The octree depth of the `start` cell [S1][S2, unofficial].
    *   `parent_trace`: An array of node indices or pointers storing the path from the root to the `start` node [S1][S2, unofficial].
    *   `child_bit_depth`: A precomputed array where `child_bit_depth[up]` holds the bit mask for subdivision level `up` [S2, unofficial].
*   **Outputs:**
    *   The index or pointer to the neighbor cell containing the `dest` coordinate [S1][S2, unofficial].

### Data Structures
*   **`parent_trace`**: An array where `parent_trace[up]` holds the node index at depth `up` on the path from the root to the `start` node [S2, unofficial].
*   **`child_bit_depth`**: A precomputed array mapping each octree level to a bitmask used to isolate the child index at that level [S2, unofficial].

### Algorithm Steps
1.  Iterate `up` from `depth` down to `0` [S2, unofficial].
2.  Retrieve the bit mask for the current level: `child_bit = child_bit_depth[up]` [S2, unofficial].
3.  Check if `start` and `dest` fall into the same child node at this level by comparing their masked bits:
    $$ (dest.x \ \& \ child\_bit) == (start.x \ \& \ child\_bit) \ \&\& \ (dest.y \ \& \ child\_bit) == (start.y \ \& \ child\_bit) \ \&\& \ (dest.z \ \& \ child\_bit) == (start.z \ \& \ child\_bit) $$
4.  If the bits match, the common ancestor is found at `parent_trace[up]`. Call `point_locate(dest, up, parent_trace[up])` to find the specific neighbor and return it [S2, unofficial].
5.  If the loop completes without returning, the root node is the common ancestor. Check if `start` and `dest` match at the root level using the `child_bit` value from the final iteration. If they match, call `point_locate(dest, 0, 0)` and return the result [S2, unofficial].
6.  If no match is found, return `0` [S2, unofficial].

### Pseudocode
The following pseudocode implements the backtracing neighbor finding algorithm [S2, unofficial]. `point_locate` is implemented as a standard octree descent routine [G].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def neighbor_find(start: Vec3i, dest: Vec3i, depth: int, parent_trace: List[int], child_bit_depth: List[int]) -> int:
    """
    Finds the neighbor cell containing 'dest' starting from 'start'.
    Uses parent_trace to backtrack to the common ancestor.
    """
    # Iterate from current depth down to 0
    for up in range(depth, -1, -1):
        child_bit = child_bit_depth[up]
        
        # Check if start and dest share the same parent at this level
        if ((dest.x & child_bit) == (start.x & child_bit) and
            (dest.y & child_bit) == (start.y & child_bit) and
            (dest.z & child_bit) == (start.z & child_bit)):
            
            # Found common ancestor at parent_trace[up]
            return point_locate(dest, up, parent_trace[up])
            
    # Root node check (matches source structure; reuses child_bit from up=0 iteration)
    if ((dest.x & child_bit) == (start.x & child_bit) and
        (dest.y & child_bit) == (start.y & child_bit) and
        (dest.z & child_bit) == (start.z & child_bit)):
        return point_locate(dest, 0, 0)
        
    return 0  # Not found

def point_locate(dest: Vec3i, depth: int, start_node: int) -> int:
    """
    Standard octree descent to find the leaf/child containing dest.
    Assumes a flat array 'nodes' where nodes[i].children[child_idx] gives the next node.
    """
    current = start_node
    for d in range(depth, -1, -1):
        # Extract child index from dest coordinates at level d
        child_idx = ((dest.x >> d) & 1) << 2 | ((dest.y >> d) & 1) << 1 | ((dest.z >> d) & 1)
        current = nodes[current].children[child_idx]
    return current
```

### Edge Cases and Pitfalls
*   **Root Level Handling:** The source code explicitly checks the root level again after the loop completes, likely to ensure the root-level bit comparison is evaluated with a valid `child_bit` value [S2, unofficial].
*   **Coordinate Scaling:** The bit-masking logic assumes `start` and `dest` coordinates are scaled consistently with the `child_bit_depth` masks [S2, unofficial].
*   **Missing Nodes:** If `dest` lies outside the octree bounds or in an unallocated region, `point_locate` must handle null pointers or return a sentinel value [G].

## Subtopics

### [Wyvill and Kunii's Ray Tracing / Neighbor Finding Algorithm](3876-wyvill-and-kunii-s-ray-tracing-neighbor-finding-algorithm.md)
A ray tracing algorithm for hierarchical spatial structures that determines the next intersected cell by calculating a point guaranteed to lie inside it, rather than computing a direction vector. Choose this variant when geometric projection is preferred over hierarchical backtracing.

## Operating characteristics
*   **Time Complexity:** $O(D)$ for the backtracing phase and $O(D)$ for the `point_locate` phase, where $D$ is the octree depth [S2, unofficial].
*   **Memory Complexity:** $O(D)$ for the `parent_trace` array [S2, unofficial].
*   **Strengths:** Efficient for sparse octrees requiring deep traversal [S2, unofficial].
*   **Limitations:** Requires maintaining `parent_trace` pointers throughout traversal; performance degrades if `point_locate` encounters invalid or unallocated nodes [S2, unofficial].

## Background
The neighbor-finding algorithm was developed for ray tracing with hierarchical spatial structures [S1]. It is used in interactive isosurface ray tracing of large octree volumes [S2, unofficial]. The method relies on a parent trace array to navigate the octree hierarchy without storing full parent pointers in every node [S2, unofficial].

## Sources
[S1] PDF Interactive Isosurface Ray Tracing of Large Octree Volumes, https://www.sci.utah.edu/~knolla/octiso-rt06.pdf, fetched 2026-10-28
[S2] Interactive isosurface ray tracing of large octree volumes, https://www.researchgate.net/publication/310054812_Interactive isosurface ray tracing of large octree volumes, fetched 2026-10-29 (unofficial)
