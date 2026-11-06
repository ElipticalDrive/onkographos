# Macrocell Hierarchical CGT

Also called: Hierarchical CGT, Recursive CGT with Array Stack

## Overview
Macrocell Hierarchical CGT extends Coherent Grid Traversal [2987-coherent-grid-traversal.md] to handle large volumes by introducing a recursive hierarchy of macrocells, where each parent block contains $2^3$ children [S1, unofficial]. It solves the performance degradation of flat CGT on large datasets by amortizing traversal costs across multiple levels of granularity, using an array stack to manage the hierarchy iteratively rather than recursively [S1, unofficial]. It differs from [Sphere-Center Method](5200-sphere-center-method.md) by maintaining slice-based frustum traversal and from [Bundle Parallelisation](6293-bundle-parallelisation.md) by focusing on spatial hierarchy management rather than ray packet parallelism.

## How it works
**Inputs:** A ray packet with bounding frustum defined by axes $U, V, K$; a volume with a hierarchical macrocell structure; a target isovalue.
**Outputs:** A list of visited cells or traversal state updates.

**Data Structures:**
The core structure is an **Array Stack** that replaces recursive function calls. For a hierarchy of depth $L$, the stack stores:
1.  `u_slice`, `v_slice`: Current slice coordinates for each level $l \in [0, L-1]$.
2.  `u_inc`, `v_inc`: Increment values for $U$ and $V$ axes for each level.
3.  `current_slice`: The current slice index within the active macrocell level.
4.  `return_slice`: The slice index at which to resume parent macrocell traversal.

**Algorithm Steps:**
1.  **Initialization:** Initialize the stack for the root macrocell level. Set initial $U, V$ slices and increments based on the frustum intersection with the root volume. Set `current_slice` to the starting $K$-slice.
2.  **Loop:** While the stack is not empty and rays have not exited the volume:
    a.  Let $l$ be the current macrocell level (initially 0).
    b.  **Culling Test:** Check if the current macrocell at `u_slice[l]`, `v_slice[l]` contains the target isovalue. This is done by comparing the isovalue against the precomputed min/max values of the macrocell [S1, unofficial].
    c.  **Traversal Decision:**
        i.  If the macrocell is empty (isovalue not in min/max range), skip to the next slice or parent level.
        ii. If the macrocell is non-empty and $l < L-1$ (not leaf), push state for child level $l+1$ onto the stack. The child coordinates are derived from parent coordinates. For a macrocell width multiple $M$, child coordinates are $M$-multiples of parent coordinates [S1, unofficial].
        iii. If the macrocell is non-empty and $l = L-1$ (leaf level), traverse the full slice of volume cells within this macrocell using standard CGT logic [S1, unofficial].
    d.  **Advance:** Increment `current_slice` for level $l$. If `current_slice` exceeds the macrocell boundary, pop the stack to return to the parent level $l-1$, setting `current_slice` to `return_slice[l-1]` [S1, unofficial].
3.  **Termination:** Stop when all rays have intersected or exited the root macrocell.

**Formulas:**
*   Child coordinate mapping: $C_{child} = M \times C_{parent}$ [S1, unofficial].
*   Next macrocell slice (for binary subdivision, $M=2$): $Slice_{next} = Slice_{current} + 2$ [S1, unofficial].

**Parameters:**
*   $M$: Macrocell width multiple. Typically $M=2$ for octree alignment [S1, unofficial].
*   $L$: Number of macrocell layers. A 3-deep hierarchy is effective for volumes $> 256^3$ [S1, unofficial].

**Edge Cases:**
*   **Implicit Macrocells:** When using an octree, macrocells are implicit. Min/max pairs are retrieved from octree interior nodes via hashing [S1, unofficial].
*   **Leaf Node Mapping:** A single coarse scalar leaf node in the octree may cover multiple finer-level implicit macrocells. The algorithm must find the deepest octree child mapping to the implicit macrocell coordinate to perform the isovalue cull [S1, unofficial].
*   **Neighbor Finding:** Lookup costs are high; storing the path from the octree root to the current node along the $U,V$-minimal ray of the frustum is recommended [S1, unofficial].

## Subtopics

### Implicit Macrocell Grid Traversal of Octree Volumes
A variant where the macrocell hierarchy is implicit in the octree structure, retrieving min/max pairs from octree interior nodes instead of explicit macrocell blocks [S1, unofficial]. It uses a common coordinate space $[0, 2^{d_{max}}]$ for all octree node depths, where $d_{max}$ is the maximum tree depth [S1, unofficial]. This allows optimization for binary subdivision: multiplying the $K$-slice by macrocell width $M=2$ is replaced by a bitwise left-shift, and computing the next macrocell slice requires a simple $+2$ addition [S1, unofficial]. It is chosen when the volume is naturally stored as an octree, avoiding the overhead of explicit macrocell data structures.

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of visited cells, but with a significantly reduced constant factor for large volumes due to early culling at macrocell levels.
*   **Memory Complexity:** $O(L)$ for the stack, where $L$ is the hierarchy depth. Additional memory for octree storage if implicit.
*   **Performance:** A recursive 3-level macrocell hierarchy yields best performance for volumes larger than $256^3$ [S1, unofficial].
*   **Strengths:** Handles large volumes efficiently; avoids recursion overhead; compatible with octree structures.
*   **Limitations:** Requires precomputed min/max values for macrocells; complex implementation due to stack management and implicit node mapping.
*   **When to use:** Large volumetric datasets ($> 256^3$) where flat CGT is too slow.
*   **When not to use:** Small volumes where overhead outweighs benefits; non-octree data without explicit macrocell construction.

## Background
Developed by Knoll et al. as an extension of Coherent Grid Traversal [2987-coherent-grid-traversal.md] to handle larger volumes, building on Parker et al.'s work [S1, unofficial]. It was implemented to address the limitations of two-level hierarchies for volumes as small as $30^3$, finding that deeper hierarchies (equivalent to full octrees) were necessary for robustness [S1, unofficial]. It is used in interactive ray tracing of implicit surfaces and volumetric visualization.

## Sources
[S1] RAY TRACING IMPLICIT SURFACES FOR INTERACTIVE ..., https://www.sci.utah.edu/~knolla/thesis.pdf, fetched 2026-10-29 (unofficial)
