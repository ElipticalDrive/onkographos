# Macrocell Hierarchical CGT

Also called: Hierarchical CGT, Recursive CGT with Array Stack

## Overview
Macrocell Hierarchical CGT extends Coherent Grid Traversal [2987-coherent-grid-traversal.md] to handle large volumes by introducing a recursive hierarchy of macrocells, where each parent block contains $2^3$ children [S1, unofficial]. It solves the performance degradation of flat CGT on large datasets by amortizing traversal costs across multiple levels of granularity, using an array stack to manage the hierarchy iteratively rather than recursively [S1, unofficial]. It differs from [Sphere-Center Method](5200-sphere-center-method.md) by maintaining slice-based frustum traversal and from [Bundle Parallelisation](6293-bundle-parallelisation.md) by focusing on spatial hierarchy management rather than ray packet parallelism [G].

## How it works
**Inputs:** A ray packet with bounding frustum defined by axes $U, V, K$; a volume with a hierarchical macrocell structure; a target isovalue [G].
**Outputs:** A list of visited cells or traversal state updates [G].

**Data Structures:**
The core structure is an **Array Stack** that replaces recursive function calls [S1, unofficial]. For a hierarchy of depth $L$, the stack stores:
1. `u_slice`, `v_slice`: Current slice coordinates for each level $l \in [0, L-1]$ [S1, unofficial].
2. `u_inc`, `v_inc`: Increment values for $U$ and $V$ axes for each level [S1, unofficial].
3. `current_slice`: The current slice index within the active macrocell level [S1, unofficial].
4. `return_slice`: The slice index at which to resume parent macrocell traversal [S1, unofficial].

**Algorithm Steps:**
1. **Initialization:** Initialize the stack for the root macrocell level. Set initial $U, V$ slices and increments based on the frustum intersection with the root volume. Set `current_slice` to the starting $K$-slice. Let $l$ be the current macrocell level (initially 0) [G].
2. **Loop:** While the stack is not empty and rays have not exited the volume:
   a. **Culling Test:** Check if the current macrocell at `u_slice[l]`, `v_slice[l]` contains the target isovalue by comparing it against the macrocell's precomputed min/max range [S1, unofficial].
   b. **Traversal Decision:**
      i. If the macrocell is empty (isovalue not in min/max range), skip to the next slice or parent level [S1, unofficial].
      ii. If the macrocell is non-empty and $l < L-1$ (not leaf), push state for child level $l+1$ onto the stack. Child coordinates are derived from parent coordinates as $M$-multiples [S1, unofficial].
      iii. If the macrocell is non-empty and $l = L-1$ (leaf level), traverse the full slice of volume cells within this macrocell using standard CGT logic [S1, unofficial].
   c. **Advance:** Increment `current_slice` for level $l$. If `current_slice` exceeds the macrocell boundary, pop the stack to return to the parent level $l-1$, setting `current_slice` to `return_slice[l-1]` [S1, unofficial].
3. **Termination:** Stop when all rays have intersected or exited the root macrocell [S1, unofficial].

**Formulas:**
* Child coordinate mapping: $C_{child} = M \times C_{parent}$ [S1, unofficial].
* Next macrocell slice (for binary subdivision, $M=2$): $Slice_{next} = Slice_{current} + 2$ [S1, unofficial].

**Parameters:**
* $M$: Macrocell width multiple. Typically $M=2$ for octree alignment [S1, unofficial].
* $L$: Number of macrocell layers. A 3-deep hierarchy is effective for volumes $> 256^3$ [S1, unofficial].

**Edge Cases:**
* **Implicit Macrocells:** When using an octree, macrocells are implicit. Min/max pairs are retrieved from octree interior nodes, but to avoid costly lookups, the path from the octree root to the current node along the $U,V$-minimal ray of the frustum is stored [S1, unofficial].
* **Leaf Node Mapping:** A single coarse scalar leaf node in the octree may cover multiple finer-level implicit macrocells. The algorithm must find the deepest octree child mapping to the implicit macrocell coordinate to perform the isovalue cull [S1, unofficial].

## Subtopics

### Implicit Macrocell Grid Traversal of Octree Volumes
A variant where the macrocell hierarchy is implicit in the octree structure, retrieving min/max pairs from octree interior nodes instead of explicit macrocell blocks [S1, unofficial]. It uses a common coordinate space $[0, 2^{d_{max}}]$ for all octree node depths, where $d_{max}$ is the maximum tree depth [S1, unofficial]. This allows optimization for binary subdivision: multiplying the $K$-slice by macrocell width $M=2$ is replaced by a bitwise left-shift, and computing the next macrocell slice requires a simple $+2$ addition [S1, unofficial]. It is chosen when the volume is naturally stored as an octree, avoiding the overhead of explicit macrocell data structures [G].

## Operating characteristics
* **Time Complexity:** $O(N)$ where $N$ is the number of visited cells, but with a significantly reduced constant factor for large volumes due to early culling at macrocell levels [G].
* **Memory Complexity:** $O(L)$ for the stack, where $L$ is the hierarchy depth. Additional memory for octree storage if implicit [G].
* **Performance:** A recursive 3-level macrocell hierarchy yields best performance for volumes larger than $256^3$ [S1, unofficial].
* **Strengths:** Handles large volumes efficiently; avoids recursion overhead; compatible with octree structures [G].
* **Limitations:** Requires precomputed min/max values for macrocells; complex implementation due to stack management and implicit node mapping [G].
* **When to use:** Large volumetric datasets ($> 256^3$) where flat CGT is too slow [G].
* **When not to use:** Small volumes where overhead outweighs benefits; non-octree data without explicit macrocell construction [G].

## Background
Developed by Knoll et al. as an extension of Coherent Grid Traversal [2987-coherent-grid-traversal.md] to handle larger volumes, building on Parker et al.'s work [S1, unofficial]. It was implemented to address the limitations of two-level hierarchies for volumes as small as $30^3$, finding that deeper hierarchies (equivalent to full octrees) were necessary for robustness [S1, unofficial]. It is used in interactive ray tracing of implicit surfaces and volumetric visualization [G].

## Sources
[S1] RAY TRACING IMPLICIT SURFACES FOR INTERACTIVE ..., https://www.sci.utah.edu/~knolla/thesis.pdf, fetched 2026-10-29 (unofficial)
