# Span Space Tree
## Overview
The Span Space Tree is a spatial data structure designed to accelerate isosurface extraction (specifically Marching Cubes) by organizing volume cells based on their scalar value ranges rather than spatial coordinates [S2]. It maps 3D volume cells into a 2D "span-space" defined by the minimum and maximum scalar values within each cell, allowing efficient querying of active cells for a given isovalue [S2]. It sets itself apart from standard spatial trees by decoupling spatial locality from value-based culling, enabling rapid traversal of cells whose value ranges overlap a query isovalue [S1, unofficial][S2].

## How it works
### Inputs and Outputs
*   **Input:** A 3D volume dataset $V$ containing $N$ cells. Each cell $c_i$ has a scalar field value range $[min_i, max_i]$ and an isovalue $h$ (for Marching Cubes) or a transfer function range [S2].
*   **Output:** A tree structure containing references to cells that are "active" (i.e., their value range overlaps with the query isovalue or range), enabling rapid traversal without checking every cell [S2].

### Data Structures
The core concept is the **Span-Space Point** [S2]. For each cell $c_i$, a point $P_i$ is defined in a 2D space:
$$ P_i = (x_i, y_i) = (min_i, max_i) $$
Constraints: $min_i \le max_i$, which restricts points to the upper-left triangle of the 2D plane where $x \le y$ [S2].

The tree structure depends on the variant:
1.  **Interval Tree:** A binary search tree where each node stores a median value and a list of intervals [S2].
2.  **K-D Tree:** A binary search tree where non-leaf nodes represent splitting hyperplanes in the 2D span-space [S2].
3.  **Quadtree:** A 4-ary tree partitioning the 2D span-space into quadrants [S2].

### Steps
1.  **Span-Space Mapping:** For every cell $c_i$ in the volume:
    *   Compute $min_i = \min(\text{scalar values at vertices of } c_i)$ [S2].
    *   Compute $max_i = \max(\text{scalar values at vertices of } c_i)$ [S2].
    *   Create a span-space point $P_i = (min_i, max_i)$ [S2].
    *   Store a pointer to cell $c_i$ associated with $P_i$ [S2].
2.  **Tree Construction:**
    *   **Interval Tree:** Collect all intervals $[min_i, max_i]$ [S2]. Find the median $m$ of all endpoints ($min_i$ and $max_i$) [S2]. The root node stores $m$ and a list of all intervals that contain $m$ [S2]. Recursively construct the left subtree with intervals strictly below $m$ and the right subtree with intervals strictly above $m$ [S2]. Construction time is $O(N \log N)$ and storage is $O(N)$ [S2].
    *   **K-D Tree:** Map all cells to points $P_i$ in span-space [S2]. Construct a K-D tree on these 2D points. Each non-leaf node represents a splitting hyperplane (axis-aligned split in 2D) [S2]. Every node stores a point from the span-space, not just leaves [S2]. Construction time is $O(N \log N)$ and storage is $O(N)$ [S2].
    *   **Quadtree:** Define the span-space bounds. If values are stored in $l$ bits, the range is $[0, 2^l - 1]$ [S2]. The span-space is a $2^l \times 2^l$ region [S2]. Partition the region into four quadrants [S2]. Recursively subdivide quadrants containing mapped points until leaf nodes are reached [S2]. Handle collisions by storing a pointer to a list of cells mapped to the same span-space coordinate [S2].
3.  **Querying Active Cells:** Given an isovalue $h$:
    *   **Interval Tree:** Traverse the tree [S2]. At each node with median $m$, if $h < m$ traverse left, if $h > m$ traverse right, if $h = m$ report all intervals stored in the node [S2]. Report all intervals on the traversal path that contain $h$ [S2]. Query time is $O(\log N + p)$, where $p$ is the number of active cells [S2].
    *   **K-D Tree:** Traverse the tree starting from the root [S2]. At each node, check if the splitting hyperplane is crossed by the query condition [S2]. Traverse only subtrees that correspond to active cells. Query time is $O(\sqrt{N} + p)$, where $p$ is the number of active cells [S2].
    *   **Quadtree:** Traverse nodes corresponding to the region in span-space where $x \le h \le y$. Retrieve cell lists from relevant nodes [S2].

### Parameters and Edge Cases
*   **Precision:** The span-space resolution is limited by the bit-depth of the scalar values. If multiple cells have identical min/max values, they map to the same point. The Quadtree variant explicitly handles this via linked lists in nodes [S2].
*   **Degenerate Input:** Cells with $min_i = max_i$ map to the diagonal $x=y$. These are valid but represent constant-value cells [S2].
*   **Memory:** Interval Trees demand higher memory space than K-D Trees for the same dataset [S2].

## Subtopics

### Interval Tree for Marching Cubes
An ordered tree used to store intervals of values in 1D, extending the binary search tree to efficiently search for intervals overlapping a given point or interval [S2]. It is constructed by recursively splitting intervals at their median endpoints [S2]. It is more efficient than K-D Trees for search ($O(\log N + p)$ vs $O(\sqrt{N} + p)$) but demands higher memory space [S2].

### K-D Tree for Span-Space
A binary search tree used to organize points in a 2D span-space (where $x=min, y=max$) to accelerate active cell queries [S2]. It maps volume data to span-space and constructs the tree such that every node stores a point [S2]. It is less efficient in query time than the Interval Tree but uses less memory [S2].

### Quadtree for Marching Cubes
A tree data structure where every non-leaf node has exactly four children, used to partition the 2D span-space into four equal regions [S2]. It handles multiple cells mapping to the same span-space point by storing a pointer to a list of volume data cells in each node [S2]. It is suitable when the span-space can be discretized into a grid based on value bit-depth [S2].

## Operating characteristics
*   **Time Complexity:** Construction is $O(N \log N)$ for Interval and K-D Tree variants [S2]. Query time for the Interval Tree is $O(\log N + p)$ and for the K-D Tree is $O(\sqrt{N} + p)$, where $N$ is the total number of cells and $p$ is the number of active cells [S2].
*   **Memory Complexity:** Storage is $O(N)$ for all variants, though Interval Trees demand higher memory space than K-D Trees [S2].
*   **Strengths:** Significantly reduces the number of cells visited during Marching Cubes by culling cells whose value ranges do not overlap the isovalue [S2]. Existing techniques enable interactive rendering of data up to $512^3$ resolution, though interactive visualization remains difficult for larger data [S1, unofficial].
*   **Limitations:** Requires preprocessing to build the span-space tree. Memory overhead can be significant for the Interval Tree variant [S2].

## Background
The Span Space Tree concept was refined by Livnat et al. as a refinement of the interval tree concept to manage scene complexity for larger volume data [S1, unofficial]. It builds on the min-max hierarchy proposed by Wilhelms and Van Gelder, which embedded min-max bounds in a branch-on-need octree to ignore empty regions [S1, unofficial]. Livnat and Hansen extended this with view-dependent frustum culling using a shearing transformation visibility test [S1, unofficial]. The specific application of span-space trees (Interval, K-D, Quadtree) for accelerating Marching Cubes is detailed in comparative studies of spatial data structures for volumetric visualization [S2].

## Sources
[S1] RAY TRACING IMPLICIT SURFACES FOR INTERACTIVE ..., https://www.sci.utah.edu/~knolla/thesis.pdf, fetched 2026-10-29 (unofficial)
[S2] Marching cubes technique for volumetric visualization accelerated with graphics processing units, https://doi.org/10.1007/s13173-012-0097-z, fetched 2026-10-26
