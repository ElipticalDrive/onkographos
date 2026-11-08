# Standard Interval Tree for Isosurface Extraction

Also called: Cignoni et al. Interval Tree, Interval Tree, Interval-Tree Based Isosurface Extraction, Normal Interval Tree

## Overview
The Standard Interval Tree is a spatial indexing structure used to efficiently identify active cells (cells intersected by an isosurface) in volumetric data, serving as a pre-processing step for surface extraction algorithms like Marching Cubes or Marching Tetrahedra [S1, unofficial][S3, book]. It solves the problem of $O(N)$ brute-force traversal by achieving optimal output-sensitive query complexity, reporting only the cells relevant to a specific isovalue [S1, unofficial][S3, book]. It sets itself apart from its parent [Octree-based Isosurfacing](1247-octree-based-isosurfacing.md) by indexing cells based on scalar value intervals rather than spatial subdivision, and from siblings like [Wilhelms and Van Gelder Octree](2107-wilhelms-and-van-gelder-octree.md) by using a binary tree of value intervals rather than a spatial hierarchy, thereby decoupling spatial locality from value-based filtering [S4, unofficial].

## How it works
**Inputs:** A structured 3D volume grid of dimensions $\Delta x \times \Delta y \times \Delta z$ [S1, unofficial]. Scalar field values $V$ for each cell [S4, unofficial]. A target isovalue $t$ [S4, unofficial].

**Outputs:** A list of active cells whose scalar range $[v_{min}, v_{max}]$ contains $t$ [S4, unofficial].

**Data Structures:** The interval tree is a balanced binary tree where each node stores a subset of cells [S3, book]. Each node contains two sorted lists of pointers to cells: a `min-list` sorted by increasing $v_{min}$, and a `max-list` sorted by decreasing $v_{max}$ [S4, unofficial]. Each cell reference stores $v_{min}$ (2 bytes), $v_{max}$ (2 bytes), and a pointer to the cell data (4 bytes), totaling 12 bytes per cell [S4, unofficial].

**Tree Construction:**
1. Calculate $v_{min}$ and $v_{max}$ for every cell in the volume [S4, unofficial].
2. Choose a threshold $t_0$ [S4, unofficial].
3. Partition cells into three sets: $S_{contain}$ (cells where $v_{min} \le t_0 \le v_{max}$), $S_{below}$ (cells where $v_{max} < t_0$), and $S_{above}$ (cells where $v_{min} > t_0$) [S4, unofficial].
4. Store $S_{contain}$ in the current node's `min-list` and `max-list` [S4, unofficial].
5. Recursively build the left child from $S_{below}$ and the right child from $S_{above}$ [S4, unofficial].

**Query Algorithm:**
1. Start at the root node with threshold $t_0$ [S4, unofficial].
2. If $t < t_0$: iterate through the `min-list`, reporting all cells where $v_{min} \le t$, then stop when encountering a cell with $v_{min} > t$, and recurse into the left child [S4, unofficial].
3. If $t > t_0$: iterate through the `max-list`, reporting all cells where $v_{max} \ge t$, then stop when encountering a cell with $v_{max} < t$, and recurse into the right child [S4, unofficial].
4. If $t = t_0$: report all cells in the current node's lists and terminate the search [S4, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Cell:
    def __init__(self, vmin, vmax, data_ptr):
        self.vmin = vmin
        self.vmax = vmax
        self.data_ptr = data_ptr

class Node:
    def __init__(self, threshold):
        self.threshold = threshold
        self.min_list = []  # sorted ascending by vmin
        self.max_list = []  # sorted descending by vmax
        self.left_child = None
        self.right_child = None

def query_interval_tree(node, t):
    """
    Recursively query the interval tree for cells containing isovalue t.
    """
    active_cells = []
    
    if t < node.threshold:
        for cell in node.min_list:
            if cell.vmin <= t:
                active_cells.append(cell)
            else:
                break
        if node.left_child is not None:
            active_cells.extend(query_interval_tree(node.left_child, t))
            
    elif t > node.threshold:
        for cell in node.max_list:
            if cell.vmax >= t:
                active_cells.append(cell)
            else:
                break
        if node.right_child is not None:
            active_cells.extend(query_interval_tree(node.right_child, t))
            
    else:  # t == node.threshold
        active_cells.extend(node.min_list)
        
    return active_cells
```

**Parameters:**
- **Pointer Size:** 4 bytes [S1, unofficial].
- **Value Type:** Typically 2 bytes (16-bit) for $v_{min}, v_{max}$ [S4, unofficial].
- **Threshold Selection:** Not explicitly defined in sources, but optimal balance requires median selection [G].

**Edge Cases and Pitfalls:**
- **Precision:** Floating-point comparisons must handle the equality case $t = t_0$ carefully to avoid missing cells or double-counting if lists overlap [S4, unofficial].
- **Memory Overhead:** The structure is space-inefficient. For 8-bit volumes, the total space usage is approximately 9 times the size of the raw volume data (1x volume + 8x overhead for pointers and lists) [S1, unofficial].
- **Degenerate Input:** If all cells have the same $v_{min}$ and $v_{max}$, the tree depth becomes 1, and all cells are stored in the root. Query time becomes $O(k)$ but construction is trivial [S4, unofficial].

## Subtopics

### [Fixed-Sized Buckets for Isosurface Extraction](1252-fixed-sized-buckets-for-isosurface-extraction.md)
Fixed-Sized Buckets is an output-optimal algorithm for accelerated isosurface extraction from volumetric data sets. It identifies active cells (cells intersected by the isosurface) with work proportional to the output size rather than the input size, using a logical 2D array data structure to sort cells by intensity ranges. It sets itself apart from its parent [Standard Interval Tree for Isosurface Extraction](1230-standard-interval-tree-for-isosurface-extraction.md) by using a simpler array-based structure that avoids tree traversal overhead, and from siblings like [Interval Table for Isosurface Extraction](4357-interval-table-for-isosurface-extraction.md) by relying on two-pass sorting rather than interval merging.

### [Interval Tree with Cell Grouping](1342-interval-tree-with-cell-grouping.md)
The Interval Tree with Cell Grouping is a memory-optimized variant of the Standard Interval Tree for Isosurface Extraction. It reduces the memory overhead of the auxiliary lists by having each pointer reference a group of eight adjacent cells rather than a single cell, resulting in a tree size approximately 1/8th that of a regular interval tree. This reduction comes at the cost of increased query time, as intersecting groups require checking all eight constituent cells individually.

### [Span Space Isosurface Extraction](1750-span-space-isosurface-extraction.md)
Span Space Isosurface Extraction is a method for rapidly locating cells intersecting an isosurface in large scalar datasets by mapping cell extrema to a 2D "span space" and indexing them with a kd-tree. It solves the problem of efficient interval search by reducing the dimensionality of the query from interval containment to point location relative to the diagonal $y=x$. Unlike the Standard Interval Tree for Isosurface Extraction, which achieves $O(\log n + k)$ query complexity, this method achieves a worst-case search complexity of $O(\sqrt{n} + k)$. It is chosen when a simpler, pointerless kd-tree implementation is preferred over complex interval tree structures, accepting a slightly higher worst-case asymptotic bound for potentially better constant factors and memory locality[6891-span-space-decomposition.md].

### [Out-of-core Interval Tree Extraction](2146-out-of-core-interval-tree-extraction.md)
Out-of-core Interval Tree Extraction is a method for isosurface extraction that stores the interval tree and associated min/max lists on disk in blocks, loading only the necessary blocks based on isovalue intervals. It builds on [Standard Interval Tree for Isosurface Extraction](1230-standard-interval-tree-for-isosurface-extraction.md) by partitioning the tree structure to fit within fixed-size disk blocks, enabling the processing of datasets larger than available main memory. Unlike its siblings, it explicitly models the I/O cost of reading tree nodes and min/max lists as a function of isovalue probability distributions.

### [Interval Table for Isosurface Extraction](4357-interval-table-for-isosurface-extraction.md)
The Interval Table is a pre-processing data structure that maps scalar value intervals to tetrahedron IDs, allowing the extraction algorithm to skip tetrahedra whose function span does not contain the isovalue. It differs from the Standard Interval Tree by using a fixed-size array of buckets (intervals) rather than a dynamic tree structure, offering a simpler implementation with comparable performance for uniform scalar distributions.

### [Search Data Structure Based Isosurface Extraction](6738-search-data-structure-based-isosurface-extraction.md)
Search Data Structure Based Isosurface Extraction is a family of methods that utilize external search structures (indices) to accelerate isosurface extraction for multiple queries on the same dataset. By creating an index that allows the algorithm to disregard regions containing only non-active cells, these methods achieve significant performance gains over brute-force traversal when repeated queries are required. This approach is distinct from single-pass methods because the overhead of index construction must be amortized over multiple queries; it is not feasible if the dataset is examined only once or if the index transfer overhead is prohibitive.

### Matryoshka Layer Insertion
Matryoshka Layer Insertion is a data structure construction where cells are inserted into layers defined by six planes (X=x0, X=x1, Y=y0, Y=y1, Z=z0, Z=z1) based on coordinate minima, allowing $O(1)$ insertion and $O(k)$ total construction time for $k$ cells [S1, unofficial]. The algorithm calculates `min*1` as the minimum of the cell's coordinates $(cellx, celly, cellz)$ and `min*2` as the minimum of the distances to the opposite boundaries $((\Delta x-1)-cellx, (\Delta y-1)-celly, (\Delta z-1)-cellz)$ [S1, unofficial]. The layer index is the minimum of `min*1` and `min*2` [S1, unofficial]. Inside the layer, the cell is inserted into the appropriate plane list based on which coordinate matches the boundary plane [S1, unofficial]. To avoid a second processing round, this insertion is merged into the query process of the interval tree [S1, unofficial]. Construction complexity is $O(k)$ where $k$ is the number of active cells, with memory overhead directly proportional to $k$ [S1, unofficial].

### Edge-Covering Space Reduction
Edge-Covering Space Reduction is a space reduction technique for interval/kd-trees by restricting the data structure to a subset of cells that covers all edges, reducing storage by a factor of 4 [S4, unofficial]. The method selects a subset of cells such that every edge in the volume grid is covered by at least one selected cell [S4, unofficial]. Since an edge is shared by four cells, only one out of four neighboring cells needs to be considered [S4, unofficial]. After extraction from the data structure, the isosurface is propagated from the reported active cells onto neighboring cells that were not covered in the data structure [S4, unofficial]. This increases extraction time but significantly reduces memory requirements [S4, unofficial].

## Operating characteristics
- **Time Complexity:** Query time is $O(\log n + k)$, where $n$ is the number of cells and $k$ is the number of active cells reported [S1, unofficial]. In practice, for scalar volumes, $k$ dominates $\log n$ as active cells often number in the hundreds of thousands or millions [S1, unofficial]. Search complexity is also described as $O(k + \log h)$, where $h$ is the number of nodes in the tree [S3, book].
- **Memory Complexity:** $O(n)$. The space overhead is significant. For a volume of size $(\Delta x - 1)(\Delta y - 1)(\Delta z - 1)$, the overhead is $8(\Delta x - 1)(\Delta y - 1)(\Delta z - 1)$ bytes for pointers alone (2 pointers per cell, 4 bytes each), plus the volume data itself, totaling 9 times the volume size [S1, unofficial]. A more precise analysis for general interval trees states memory overhead is $3h + 2m$ words, where $h$ is the number of distinct extremes and $m$ is the number of cells [S3, book].
- **Strengths:** Optimal output-sensitive performance for single-query isosurface extraction; significantly faster than brute-force traversal for sparse isosurfaces [S3, book].
- **Limitations:** High memory consumption makes it prohibitive for very large high-resolution datasets without variants like Cell Grouping or Edge-Covering [S1, unofficial].
- **When to use:** When memory is sufficient and fast single-query extraction is required [S3, book].
- **When not to use:** When memory is constrained (use [Interval Tree with Cell Grouping](1342-interval-tree-with-cell-grouping.md) or [Edge-Covering Space Reduction]) or when multiple queries are needed (consider [Search Data Structure Based Isosurface Extraction](6738-search-data-structure-based-isosurface-extraction.md)) [S1, unofficial].

## Background
The Standard Interval Tree for isosurface extraction was proposed by Cignoni, Marino, Montani, and Puppo [S3, book][S4, unofficial]. It was introduced as an alternative to kd-trees for spatial indexing in volume visualization, addressing the space inefficiency of kd-trees while maintaining optimal query complexity [S4, unofficial]. The method is detailed in the *Visualization Handbook* (2005) and various conference papers [S1, unofficial][S3, book].

## Sources
[S1] Speeding Up Isosurfacing: The Matryoshka Algorithm, https://doi.org/10.2312/localchapterevents/tpcg/tpcg07/097-105, fetched 2026-10-26 (unofficial)
[S3] marching cubes algorithm, https://www.sciencedirect.com/topics/computer-science/marching-cube-algorithm, fetched 2026-10-30 (book)
[S4] Optimal hybrid memory constrained isosurface extraction, http://nbn-resolving.de/urn:nbn:de:bsz:352-130091, fetched 2026-10-30 (unofficial)
