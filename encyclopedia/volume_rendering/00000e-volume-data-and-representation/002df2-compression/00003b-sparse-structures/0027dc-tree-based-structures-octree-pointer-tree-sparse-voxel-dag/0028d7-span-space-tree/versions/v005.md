# Span Space Tree

Also called: Span Space Data Structure

## Overview
The Span Space Tree is a spatial data structure designed to accelerate isosurface extraction (specifically Marching Cubes) and volume rendering by organizing volume cells based on their value ranges rather than their spatial coordinates. It maps 3D volume cells into a 2D "span-space" defined by the minimum and maximum scalar values within each cell, allowing efficient querying of active cells for a given isovalue. It sets itself apart from standard spatial trees (like Octrees) by decoupling spatial locality from value-based culling, making it particularly effective for reducing the visited set of cells during isosurface generation in large datasets [S1, unofficial][S2].

## How it works
The method involves mapping volume data into a 2D coordinate system (span-space) and organizing these mappings into a tree structure. The specific implementation details vary depending on the underlying tree type (Interval Tree, K-D Tree, or Quadtree) used to manage the span-space.

### Inputs and Outputs
*   **Input:** A 3D volume dataset $V$ consisting of $N$ cells. Each cell $c_i$ has a scalar field value range $[min_i, max_i]$. An isovalue $h$ (for Marching Cubes) or a transfer function range (for ray casting).
*   **Output:** A tree structure containing references to cells that are "active" (i.e., their value range overlaps with the query isovalue or range), enabling rapid traversal without checking every cell.

### Data Structures
The core concept is the **Span-Space Point**. For each cell $c_i$, a point $P_i$ is defined in a 2D space:
$$ P_i = (x_i, y_i) = (min_i, max_i) $$
Constraints: $min_i \le max_i$. This restricts points to the upper-left triangle of the 2D plane where $x \le y$ [S2].

The tree structure depends on the variant:
1.  **Interval Tree:** A binary search tree where each node stores a median value and a list of intervals.
2.  **K-D Tree:** A binary search tree where non-leaf nodes represent splitting hyperplanes in the 2D span-space.
3.  **Quadtree:** A 4-ary tree partitioning the 2D span-space into quadrants.

### Steps

#### 1. Span-Space Mapping
For every cell $c_i$ in the volume:
1.  Compute $min_i = \min(\text{scalar values at vertices of } c_i)$.
2.  Compute $max_i = \max(\text{scalar values at vertices of } c_i)$.
3.  Create a span-space point $P_i = (min_i, max_i)$.
4.  Store a pointer to cell $c_i$ associated with $P_i$.

#### 2. Tree Construction
*   **For Interval Tree [S2]:**
    1.  Collect all intervals $[min_i, max_i]$.
    2.  Find the median $m$ of all endpoints ($min_i$ and $max_i$).
    3.  The root node stores $m$ and a list of all intervals that contain $m$.
    4.  Recursively construct the left subtree with intervals strictly below $m$ and the right subtree with intervals strictly above $m$.
    5.  Construction time: $O(N \log N)$. Storage: $O(N)$.

*   **For K-D Tree [S2]:**
    1.  Map all cells to points $P_i$ in span-space.
    2.  Construct a K-D tree on these 2D points.
    3.  Each non-leaf node represents a splitting hyperplane (axis-aligned split in 2D).
    4.  Every node stores a point from the span-space (not just leaves).
    5.  Construction time: $O(N \log N)$. Storage: $O(N)$.

*   **For Quadtree [S2]:**
    1.  Define the span-space bounds. If values are stored in $l$ bits, the range is $[0, 2^l - 1]$. The span-space is a $2^l \times 2^l$ region.
    2.  Partition the region into four quadrants.
    3.  Recursively subdivide quadrants containing mapped points until leaf nodes are reached.
    4.  Handle collisions: Multiple cells may map to the same point $(min, max)$. Each node stores a pointer to a list of cells mapped to that specific span-space coordinate.

#### 3. Querying Active Cells
Given an isovalue $h$:
*   **Interval Tree [S2]:**
    1.  Traverse the tree.
    2.  At each node with median $m$:
        *   If $h < m$, traverse left.
        *   If $h > m$, traverse right.
        *   If $h = m$, report all intervals stored in the node (as they contain $m$).
    3.  Report all intervals in the path that contain $h$.
    4.  Query time: $O(\log N + p)$, where $p$ is the number of active cells.

*   **K-D Tree [S2]:**
    1.  Traverse the tree starting from the root.
    2.  At each node, check if the splitting hyperplane is crossed by the query condition (isovalue $h$).
    3.  Traverse only subtrees that correspond to active cells (cells whose span overlaps $h$).
    4.  Query time: $O(\sqrt{N} + p)$, where $p$ is the number of active cells.

*   **Quadtree [S2]:**
    1.  Traverse nodes corresponding to the region in span-space where $x \le h \le y$.
    2.  Retrieve cell lists from relevant nodes.

### Parameters and Edge Cases
*   **Precision:** The span-space resolution is limited by the bit-depth of the scalar values. If multiple cells have identical min/max values, they map to the same point. The Quadtree variant explicitly handles this via linked lists in nodes [S2].
*   **Degenerate Input:** Cells with $min_i = max_i$ map to the diagonal $x=y$. These are valid but represent constant-value cells.
*   **Memory:** Interval Trees demand higher memory space than K-D Trees for the same dataset [S2].

## Subtopics

### Interval Tree for Marching Cubes
An ordered tree used to store intervals of values in 1D, extending the binary search tree to efficiently search for intervals overlapping a given point or interval. It is constructed by recursively splitting intervals at their median endpoints. It is more efficient than K-D Trees for search ($O(\log N + p)$ vs $O(\sqrt{N} + p)$) but demands higher memory space [S2].

### K-D Tree for Span-Space
A binary search tree used to organize points in a 2D span-space (where $x=min, y=max$) to accelerate active cell queries. It maps volume data to span-space and constructs the tree such that every node stores a point. It is less efficient in query time than the Interval Tree but uses less memory [S2].

### Quadtree for Marching Cubes
A tree data structure where every non-leaf node has exactly four children, used to partition the 2D span-space into four equal regions. It handles multiple cells mapping to the same span-space point by storing a pointer to a list of volume data cells in each node. It is suitable when the span-space can be discretized into a grid based on value bit-depth [S2].

## Operating characteristics
*   **Time Complexity:**
    *   Construction: $O(N \log N)$ for Interval and K-D Tree variants [S2].
    *   Query (Interval Tree): $O(\log N + p)$ [S2].
    *   Query (K-D Tree): $O(\sqrt{N} + p)$ [S2].
    *   Here, $N$ is the total number of cells and $p$ is the number of active cells (cells intersecting the isovalue).
*   **Memory Complexity:** $O(N)$ for all variants [S2]. Interval Trees require more memory than K-D Trees [S2].
*   **Strengths:** Significantly reduces the number of cells visited during Marching Cubes by culling cells whose value ranges do not overlap the isovalue. Effective for large datasets where interactive rendering is difficult with brute-force methods [S1, unofficial].
*   **Limitations:** Requires pre-processing to build the span-space tree. Memory overhead can be significant for the Interval Tree variant. Does not inherently provide spatial locality for ray casting unless combined with spatial structures (like Octrees) [S1, unofficial].

## Background
The Span Space Tree concept was refined by Livnat et al. as a refinement of the interval tree concept to manage scene complexity for larger volume data [S1, unofficial]. It builds on the min-max hierarchy proposed by Wilhelms and Van Gelder, which embedded min-max bounds in an octree to ignore empty regions [S1, unofficial]. Livnat and Hansen extended this with view-dependent frustum culling [S1, unofficial]. The specific application of span-space trees (Interval, K-D, Quadtree) for accelerating Marching Cubes is detailed in comparative studies of spatial data structures for volumetric visualization [S2].

## Sources
[S1] RAY TRACING IMPLICIT SURFACES FOR INTERACTIVE ..., https://www.sci.utah.edu/~knolla/thesis.pdf, fetched 2026-10-29 (unofficial)
[S2] Marching cubes technique for volumetric visualization accelerated with graphics processing units, https://doi.org/10.1007/s13173-012-0097-z, fetched 2026-10-26
