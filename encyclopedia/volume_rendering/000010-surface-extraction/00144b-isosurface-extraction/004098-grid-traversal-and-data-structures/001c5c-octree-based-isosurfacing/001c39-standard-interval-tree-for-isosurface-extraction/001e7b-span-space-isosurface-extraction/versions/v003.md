# Span Space Isosurface Extraction

Also called: Span Space Algorithm, Livat, Shen, and Johnson method

## Overview
Span Space Isosurface Extraction is a method for rapidly locating all volume cells intersecting a specific isovalue by mapping cell scalar extrema into a 2D coordinate system known as "span space" [S1, book]. It solves the problem of efficient active cell identification in large datasets by reducing the search to a 2D range query, achieving a worst-case search complexity of $O(\sqrt{n} + k)$, where $n$ is the dataset size and $k$ is the number of cells in the isosurface [S1, book]. It differs from sibling methods by explicitly leveraging the geometric properties of min/max value pairs in a 2D plane rather than relying solely on hierarchical spatial subdivision of the domain [6891-span-space-decomposition.md].

## How it works
**Inputs and Outputs**
*   **Input:** A volume dataset consisting of $n$ cells. For each cell $i$, the minimum scalar value $a_i$ and maximum scalar value $b_i$ among its vertices are precomputed. An isovalue $v$ (or $t$) defining the surface.
*   **Output:** A list of all cells $i$ such that $a_i < v < b_i$ [S1, book].

**Data Structures**
The core data structure is a kd-tree built over the span space points [S1, book].

1.  **Span Space Point:** Each cell $c_i$ is represented as a point $p_i = (a_i, b_i)$ in $\mathbb{R}^2$, where $a_i = \min_j \{v_j\}_i$ and $b_i = \max_j \{v_j\}_i$ [S1, book]. All points lie on or above the line $y=x$ because $a_i \le b_i$ [S1, book].
2.  **Kd-Tree Node:**
    *   **Standard Layout:** Each node stores one span point coordinate (alternating between min and max dimensions at each level) and two child pointers [S1, book].
    *   **Pointerless Layout:** To reduce memory overhead, the tree can be stored in a 1D array. The root is at the middle index. The left subtree occupies the first $n/2$ nodes, and the right subtree occupies the last $(n-1)/2$ nodes [S1, book].
    *   **Per-Cell Storage (Pointerless):** 2 bytes for $v_{min}$, 2 bytes for $v_{max}$, and 4 bytes for a pointer back to the original cell data, totaling 8 bytes per cell [S2, unofficial].

**Construction Phase**
1.  Compute $a_i$ and $b_i$ for all $n$ cells.
2.  Recursively build a balanced kd-tree:
    *   At each level, select a dimension to split on (alternating between min-coordinate and max-coordinate) [S1, book].
    *   Find the median value of the data points along the current dimension. The sources suggest using a Quicksort-based selection algorithm, which has an average case complexity of $O(n)$ and requires no additional memory [S1, book].
    *   Partition the data into two subsets: those with values less than the median and those greater [S1, book].
    *   Repeat recursively until leaf nodes are reached [S1, book].
    *   Total construction complexity is $O(n \log n)$ [S1, book].

**Query Phase**
Given an isovalue $v$, the goal is to find all points $(x_i, y_i)$ such that $x_i < v < y_i$ [S1, book]. Geometrically, this corresponds to finding points to the left of the vertical line $x=v$ and above the horizontal line $y=v$ (i.e., upper-left of $(v,v)$) [S2, unofficial].

The search uses two recursive routines: `SearchMinMax` and `SearchMaxMin`. The first name indicates the dimension currently being checked; the second indicates the dimension still needing search [S1, book].

1.  **SearchMinMax(isovalue, node):**
    *   Check `node.min` against `isovalue`.
    *   If `node.min < isovalue`:
        *   The minimum condition is satisfied. Check `node.max`.
        *   If `node.max > isovalue`, the cell intersects the isosurface. Construct polygon(s) from the node [S1, book].
        *   Recursively call `SearchMaxMin(isovalue, node.right)`.
    *   Recursively call `SearchMaxMin(isovalue, node.left)` [S1, book].

2.  **SearchMaxMin(isovalue, node):**
    *   Check `node.max` against `isovalue`.
    *   If `node.max > isovalue`:
        *   The maximum condition is satisfied. Check `node.min`.
        *   If `node.min < isovalue`, the cell intersects. Construct polygon(s) [S1, book].
        *   Recursively call `SearchMinMax(isovalue, node.left)`.
    *   Recursively call `SearchMinMax(isovalue, node.right)` [S1, book].

**Optimized Search (Search-Min / Search-Max)**
To improve performance, the algorithm can skip levels when one condition is already known to be satisfied [S1, book].

1.  **Search-Min(isovalue, node):**
    *   Used when the maximum condition is already satisfied (e.g., we are in a subtree where all points have $y > v$).
    *   If `isovalue > node.min`:
        *   The left subtree's minimum condition is satisfied for all its nodes.
        *   Skip odd levels for the left subtree and sequentially traverse the contiguous block of nodes representing that subtree to report active cells [S1, book].
    *   Recursively call `Search-Max(isovalue, node.right)` [S1, book].

2.  **Search-Max(isovalue, node):**
    *   Used when the minimum condition is already satisfied.
    *   If `isovalue < node.max`:
        *   The right subtree's maximum condition is satisfied.
        *   Skip odd levels for the right subtree and sequentially traverse [S1, book].
    *   Recursively call `Search-Min(isovalue, node.left)` [S1, book].

**Pruning**
*   If the region corresponding to a node in span space is entirely contained in the area to the upper left of $(t,t)$, report all cells in that node as active [S2, unofficial].
*   If the region is disjoint from that rectangle, report them as empty [S2, unofficial].

**Edge Cases and Pitfalls**
*   **Degenerate Cells:** Cells with more than one vertex sharing the minimum or maximum value may have points lying exactly on the lines $x=v$ or $y=v$ [S1, book].
*   **Intersection vs. Touch:** If the isovalue equals the extremum, the surface touches but does not strictly intersect the cell [S1, book].
    *   Ignoring face-touching cases causes holes in the isosurface [S1, book].
    *   Ignoring edge/vertex cases is safe [S1, book].
*   **Solutions:**
    1.  Perturb the isovalue by a small amount to ensure strict inequality [S1, book].
    2.  Check both sides of the kd-tree for equality (costly) [S1, book].

## Subtopics

### [Span Space Decomposition](6891-span-space-decomposition.md)
A family of methods partitioning volume cells by mapping extreme scalar values into a 2D coordinate system. It reduces the search to a 2D range query with $O(\sqrt{n} + k)$ complexity. It differs from siblings by leveraging geometric properties of min/max pairs in 2D rather than hierarchical spatial subdivision.

## Operating characteristics
*   **Time Complexity:**
    *   **Search:** Worst-case $O(\sqrt{n} + k)$ [S1, book]. This is based on Lee and Wong's analysis where visited nodes are not part of the final result [S1, book].
    *   **Average-case:** $O(k)$ for typical applications where $k \sim n^{2/3}$, making the algorithm optimal $\theta(k)$ as it is bounded below by output size [S1, book].
    *   **Construction:** $O(n \log n)$ [S1, book].
*   **Memory Complexity:** $O(n)$, where $n$ is the number of cells [S1, book]. Using a pointerless kd-tree reduces overhead to 8 bytes per cell (2 bytes min, 2 bytes max, 4 bytes pointer) [S2, unofficial].
*   **Strengths:** Near-optimal search complexity; efficient for large datasets; pointerless implementation reduces memory footprint [S1, book].
*   **Limitations:** Degenerate cells require special handling; worst-case performance depends on data distribution [S1, book].
*   **When to use:** Large-scale imaging datasets requiring near-interactive extraction rates [S1, book].
*   **When not to use:** Small datasets where simpler methods suffice; scenes with many degenerate cells if perturbation is not acceptable [S1, book].

## Background
The method was developed by Livat, Shen, and Johnson as part of the NOISE (Near Optimal Isosurface Extraction) algorithm [S1, book]. It addresses the limitations of previous interval search methods that ordered intervals by only one dimension (min or max), requiring complex merging of lists [S1, book]. By using span space and kd-trees, it simultaneously orders cells by both minimum and maximum values [S1, book]. The worst-case search complexity analysis was developed by Lee and Wong [S1, book].

## Sources
[S1] Fast Isosurface Extraction Methods for Large Image Data sets, https://publications.sci.utah.edu/publications/livnat08/Large-scale-isosurface08.pdf, fetched 2026-10-28 (book)
[S2] Optimal hybrid memory constrained isosurface extraction, http://nbn-resolving.de/urn:nbn:de:bsz:352-130091, fetched 2026-10-30 (unofficial)
