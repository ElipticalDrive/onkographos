# Standard Interval Tree for Isosurface Extraction
Also called: Cignoni et al. Interval Tree, Interval Tree, Interval-Tree Based Isosurface Extraction, Normal Interval Tree

## Overview
The Standard Interval Tree is a spatial indexing structure used to efficiently identify active cells (cells intersected by an isosurface) in volumetric data, serving as a pre-processing step for surface extraction algorithms like Marching Cubes or Marching Tetrahedra [S1, unofficial][S3, unofficial]. It solves the problem of $O(N)$ traversal cost by providing optimal output-sensitive query complexity $O(\log n + k)$, where $n$ is the number of cells and $k$ is the number of active cells [S1, unofficial]. It sets itself apart from its parent [Octree-based Isosurfacing](1247-octree-based-isosurfacing.md) by operating on a fixed logical grid structure rather than an adaptive octree, and from siblings like [IsoOctree](1744-isooctree.md) by using a binary tree of value intervals rather than a spatial hierarchy of cell groups [S1, unofficial][S3, unofficial].

## How it works
**Inputs:** A structured volume grid of dimensions $\Delta x \times \Delta y \times \Delta z$, containing $m = (\Delta x - 1)(\Delta y - 1)(\Delta z - 1)$ cells. Each cell $i$ has a scalar value range $[v_{min}^i, v_{max}^i]$. A target isovalue $t$.
**Outputs:** A list of active cells $C_{active}$ such that $v_{min}^i \le t \le v_{max}^i$ for all $i \in C_{active}$.

**Data Structures:**
The structure is a binary tree. Each node stores:
1.  A threshold value $t_0$.
2.  Two doubly-linked lists (or arrays) of cell indices:
    *   `min-list`: Cells sorted by increasing $v_{min}$.
    *   `max-list`: Cells sorted by decreasing $v_{max}$.
3.  Pointers to left and right children.

**Memory Layout:**
For each cell stored in a node, the storage consists of:
*   $v_{min}$ (2 bytes, typically)
*   $v_{max}$ (2 bytes, typically)
*   Pointer to cell in `min-list` (4 bytes)
*   Pointer to cell in `max-list` (4 bytes)
Total per cell: 12 bytes [S3, unofficial].
The total memory overhead for the general representation is $3h + 2m$ words, where $h$ is the number of distinct extremes of intervals and $m$ is the number of cells [S2, book]. In 8-bit volumes, the total space usage including the volume data is approximately 9 times the size of the raw volume data [S1, unofficial].

**Construction Algorithm:**
1.  Collect all cell intervals $[v_{min}^i, v_{max}^i]$ for $i = 1 \dots m$.
2.  Call `BuildTree(intervals)`:
    a.  If `intervals` is empty, return `null`.
    b.  Choose a threshold $t_0$ from the set of interval endpoints. (Optimal choice minimizes tree height; a common heuristic is the median of the endpoints) [S3, unofficial].
    c.  Partition `intervals` into three sets:
        *   $S_{contain}$: Intervals where $v_{min} \le t_0 \le v_{max}$.
        *   $S_{below}$: Intervals where $v_{max} < t_0$.
        *   $S_{above}$: Intervals where $v_{min} > t_0$.
    d.  Create a new node with threshold $t_0$.
    e.  For each cell in $S_{contain}$:
        i.  Add cell to the node's `min-list` and `max-list`.
        ii. Insert into `min-list` maintaining sort order of $v_{min}$.
        iii. Insert into `max-list` maintaining sort order of $v_{max}$.
    f.  Set `node.left` = `BuildTree(S_below)`.
    g.  Set `node.right` = `BuildTree(S_above)`.
    h.  Return `node`.

**Query Algorithm:**
1.  Start at `root`.
2.  If `root` is `null`, return empty list.
3.  Let $t$ be the target isovalue.
4.  If $t < root.t_0$:
    a.  Iterate through `root.min-list` from the beginning.
    b.  For each cell $c$ in the list:
        i.  If $c.v_{min} > t$, stop iteration (list is sorted by increasing $v_{min}$, so no subsequent cells can contain $t$).
        ii. Else, add $c$ to $C_{active}$.
    c.  Recurse: $C_{active} \leftarrow C_{active} \cup \text{Query}(root.left, t)$.
5.  Else If $t > root.t_0$:
    a.  Iterate through `root.max-list` from the beginning.
    b.  For each cell $c$ in the list:
        i.  If $c.v_{max} \le t$, stop iteration (list is sorted by decreasing $v_{max}$, so no subsequent cells can contain $t$).
        ii. Else, add $c$ to $C_{active}$.
    c.  Recurse: $C_{active} \leftarrow C_{active} \cup \text{Query}(root.right, t)$.
6.  Else ($t = root.t_0$):
    a.  Add all cells in `root.min-list` (or `root.max-list`) to $C_{active}$.
    b.  Return $C_{active}$ (terminate search, as all containing intervals are stored here) [S3, unofficial].

**Parameters:**
*   Pointer size: 4 bytes [S1, unofficial].
*   Value type size: 2 bytes (commonly used for 16-bit scalars) [S3, unofficial].

**Edge Cases and Pitfalls:**
*   **Space Inefficiency:** The memory overhead is significant. For 8-bit volumes, total space is ~9x the volume size. This can be prohibitive for large datasets [S1, unofficial].
*   **Threshold Selection:** Poor choice of $t_0$ leads to unbalanced trees, degrading query performance from $O(\log n)$ to $O(n)$ [S3, unofficial].
*   **Duplicate Reporting:** The algorithm ensures each active cell is reported exactly once because intervals are partitioned into disjoint sets ($S_{contain}, S_{below}, S_{above}$) at each step [S3, unofficial].

## Subtopics

### [Fixed-Sized Buckets for Isosurface Extraction](1252-fixed-sized-buckets-for-isosurface-extraction.md)
Fixed-Sized Buckets is an output-optimal algorithm for accelerated isosurface extraction from volumetric data sets. It identifies active cells (cells intersected by the isosurface) with work proportional to the output size rather than the input size, using a logical 2D array data structure to sort cells by intensity ranges. It sets itself apart from its parent [Standard Interval Tree for Isosurface Extraction](1230-standard-interval-tree-for-isosurface-extraction.md) by using a simpler array-based structure that avoids tree traversal overhead, and from siblings like [Interval Table for Isosurface Extraction](4357-interval-table-for-isosurface-extraction.md) by relying on two-pass sorting rather than interval merging.

### [Interval Tree with Cell Grouping](1342-interval-tree-with-cell-grouping.md)
The Interval Tree with Cell Grouping is a memory-optimized variant of the Standard Interval Tree for Isosurface Extraction. It reduces the memory overhead of the auxiliary lists by having each pointer reference a group of eight adjacent cells rather than a single cell, resulting in a tree size approximately 1/8th that of a regular interval tree. This reduction comes at the cost of increased query time, as intersecting groups require checking all eight constituent cells individually.

### [Span Space Isosurface Extraction](1750-span-space-isosurface-extraction.md)
Span Space Isosurface Extraction is a method for rapidly locating cells intersecting an isosurface in large scalar datasets by mapping cell extrema to a 2D "span space" and indexing them with a kd-tree. It solves the problem of efficient interval search by reducing the dimensionality of the query from interval containment to point location relative to the diagonal $y=x$. Unlike the Standard Interval Tree for Isosurface Extraction, which achieves $O(\log n + k)$ query complexity, this method achieves a worst-case search complexity of $O(\sqrt{n} + k)$. It is chosen when a simpler, pointerless kd-tree implementation is preferred over complex interval tree structures, accepting a slightly higher worst-case asymptotic bound for potentially better constant factors and memory locality.

### [Out-of-core Interval Tree Extraction](2146-out-of-core-interval-tree-extraction.md)
Out-of-core Interval Tree Extraction is a method for isosurface extraction that stores the interval tree and associated min/max lists on disk in blocks, loading only the necessary blocks based on isovalue intervals. It builds on [Standard Interval Tree for Isosurface Extraction](1230-standard-interval-tree-for-isosurface-extraction.md) by partitioning the tree structure to fit within fixed-size disk blocks, enabling the processing of datasets larger than available main memory. Unlike its siblings, it explicitly models the I/O cost of reading tree nodes and min/max lists as a function of isovalue probability distributions.

### [Interval Table for Isosurface Extraction](4357-interval-table-for-isosurface-extraction.md)
The Interval Table is a pre-processing data structure that maps scalar value intervals to tetrahedron IDs, allowing the extraction algorithm to skip tetrahedra whose function span does not contain the isovalue. It differs from the Standard Interval Tree by using a fixed-size array of buckets (intervals) rather than a dynamic tree structure, offering a simpler implementation with comparable performance for uniform scalar distributions.

### Matryoshka Layer Insertion
Matryoshka Layer Insertion is a data structure construction technique where cells are inserted into layers defined by six planes ($X=x_0, X=x_1, Y=y_0, Y=y_1, Z=z_0, Z=z_1$) based on coordinate minima, allowing $O(1)$ insertion and $O(k)$ total construction time for $k$ cells [S1, unofficial].
**Steps:**
1.  Calculate $min^*1$ as the minimum of the values $cellx, celly, cellz$.
2.  Calculate $min^*2$ as the minimum of the values $(\Delta x - 1) - cellx, (\Delta y - 1) - celly, (\Delta z - 1) - cellz$.
3.  Determine the layer index by taking the minimum of $min^*1$ and $min^*2$.
4.  Inside the selected layer, insert the cell into the appropriate plane list by checking if $cellx$ equals $x_0$ or $x_1$; if not, check $celly$ against $y_0/y_1$, then $cellz$ against $z_0/z_1$ [S1, unofficial].
**Pitfall:** To avoid a second processing round, the insertion of a cell in the layers is merged into the query process of the interval tree [S1, unofficial].
**Complexity:** Construction complexity is $O(k)$ where $k$ is the number of cells; memory overhead is directly proportional to $k$ [S1, unofficial].

### Edge-Covering Space Reduction
Edge-Covering Space Reduction is a space reduction technique for interval/kd-trees by restricting the data structure to a subset of cells that covers all edges, reducing storage by a factor of 4 [S3, unofficial].
**Steps:**
1.  Select a subset of cells such that every edge in the volume grid is covered by at least one selected cell. Since an edge is shared by four cells, only one out of four neighboring cells needs to be considered [S3, unofficial].
2.  After extraction from the data structure, propagate the isosurface from the reported active cells onto neighboring cells that were not covered in the data structure [S3, unofficial].

## Operating characteristics
**Time Complexity:**
*   Query: $O(\log n + k)$, where $n$ is the number of cells and $k$ is the number of active cells [S1, unofficial].
*   Search Complexity: Specifically $O(k + \log h)$, where $h$ is the number of nodes in the tree [S3, unofficial].
*   For scalar data volumes, $k$ (active cells) usually dominates $\log n$, as $k$ is often in the hundreds of thousands or millions [S1, unofficial].

**Memory Complexity:**
*   General representation: $O(n)$ space. Specifically $3h + 2m$ words, where $h$ is the number of distinct extremes and $m$ is the number of cells [S2, book].
*   Pointer overhead: For 8-bit volumes, the interval tree holds pointers to cells in two auxiliary lists, with each cell referenced by two pointers. With 4-byte pointers, the space overhead is $8(\Delta x - 1)(\Delta y - 1)(\Delta z - 1)$ bytes [S1, unofficial].
*   Total space usage including volume data is approximately 9 times the size of the volume [S1, unofficial].

**Performance and Scaling:**
*   The method considerably improves the performance of the traversal phase with respect to standard marching tetrahedra and marching cubes algorithms [S2, book].
*   Memory overhead becomes a critical factor in high-resolution datasets [S2, book].
*   Strategies to reduce space occupancy include optimized representations for unstructured datasets (reducing overhead to $3h + m$, <25%) and chess-board approaches for structured datasets [S2, book].

**Strengths:**
*   Optimal output-sensitive time complexity for active cell extraction [S1, unofficial].
*   Effective speedup for isosurface extraction traversal [S2, book].

**Limitations:**
*   High memory consumption (up to 9x volume size) can be prohibitive [S1, unofficial].
*   Not adaptive to memory constraints without hybrid approaches [S3, unofficial].

**When to use:**
*   When fast extraction of active cells is required and sufficient memory is available.
*   When the number of active cells is large relative to the total cells, making $O(\log n + k)$ significantly better than $O(n)$.

**When not to use:**
*   When memory is severely constrained and $O(n)$ overhead is unacceptable.
*   When the dataset is extremely large and out-of-core processing is required (use [Out-of-core Interval Tree Extraction](2146-out-of-core-interval-tree-extraction.md)).

## Background
The Standard Interval Tree for isosurface extraction was proposed by Cignoni, Marino, Montani, and Puppo [S3, unofficial]. It is based on the general properties of interval trees [BvKOS00, Ede80] and was introduced as an alternative to kd-trees for spatial indexing in visualization [S3, unofficial]. The method was tested on various structured and unstructured datasets, demonstrating significant speedups over standard marching algorithms [S2, book]. It is cited in the Visualization Handbook [S2, book] and various research papers on optimal hybrid memory constrained isosurface extraction [S3, unofficial].

## Sources
[S1] Speeding Up Isosurfacing: The Matryoshka Algorithm, https://doi.org/10.2312/localchapterevents/tpcg/tpcg07/097-105, fetched 2026-10-26 (unofficial)
[S2] marching cubes algorithm, https://www.sciencedirect.com/topics/computer-science/marching-cube-algorithm, fetched 2026-10-30 (book)
[S3] Optimal hybrid memory constrained isosurface extraction, http://nbn-resolving.de/urn:nbn:de:bsz:352-130091, fetched 2026-10-30 (unofficial)
