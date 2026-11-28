# Temporal Hierarchical Index Tree (THITree)
Also called: THIT, THITree, Time Tree

## Overview
The Temporal Hierarchical Index Tree (THITree) is a data structure for fast isosurface extraction from time-dependent volumetric data sets [S1][S2, unofficial]. It solves the problem that extracting and rendering isosurfaces separately for each time step is too slow for interactive exploration of large time-varying data [S1]. Unlike the Temporal Branch-on-Need Tree (TBON), which does not exploit temporal coherence, THITree stores cells that remain constant throughout a time span only once for that entire span, achieving significant data reduction [S1]. Unlike the TSP Tree, which suffers from huge storage overhead by maintaining a full binary tree per octree node, THITree uses a Span Space and Interval Tree approach for efficiency [S1].

## How it works
The THITree consists of three elements: a Binary Time Tree representing time, a Span Space data structure for efficient interval search, and an Interval Tree for diagonal elements [S1][S2, unofficial].

### Data Structures

**Binary Time Tree**
A binary tree where nodes represent time spans. The top node represents the entire time range; leaf nodes represent single time steps at the highest temporal resolution. Temporal resolution doubles with each level [S1]. Cells that remain constant throughout a time interval are stored only in the node representing that interval, not in any descendant nodes [S1].

**Span Space**
A 2D space where each cell is represented as a point $(min, max)$, where $min$ and $max$ are the extreme scalar values of the cell [S1][S2, unofficial].
*   **Lattice Subdivision:** The Span Space is subdivided into an $L \times L$ grid of non-uniformly spaced rectangles called lattice elements [S2, unofficial].
    1.  Collect all distinct extreme values (min/max) of all cells in the time interval [S1][S2, unofficial].
    2.  Sort them in ascending order.
    3.  Find $L+1$ scalar values $\{d_0, d_1, ..., d_L\}$ that evenly separate the list into $L$ sublists of equal length [S2, unofficial].
    4.  Use these values to draw $L+1$ vertical and $L+1$ horizontal lines [S2, unofficial].
*   **Row Lists:** For each row $R$ in the Span Space, two lists of points are maintained:
    1.  Sorted by minimum value in ascending order.
    2.  Sorted by maximum value in descending order.
    *   Points on the diagonal element are excluded from these lists [S1].
*   **Diagonal Interval Trees:** For lattice elements on the diagonal line, a separate Interval Tree is created for each element to store cell intervals [S1][S2, unofficial].

**Cell Representation**
*   **Non-leaf nodes:** Cells are represented by their temporal extreme values $(min_i^j, max_i^j)$ over the node's time span $[i, j]$ [S2, unofficial].
*   **Leaf nodes:** Cells are represented by their time-specific extreme values $(min_t, max_t)$ [S2, unofficial].

### Parameters
*   **$L$:** The resolution of the lattice subdivision ($L \times L$). Controls the trade-off between storage space and extraction time. Higher $L$ reduces false positives but increases index size [S2, unofficial].
*   **MaxVariation:** The maximum allowed variation of a "constant" cell, defined as the number of grid elements a single cell may occupy in Span Space while still being called constant. Typically $2 \times 2$ elements [S1][S2, unofficial]. Increasing this parameter increases false positives but reduces memory size [S1].

### Construction Algorithm
1.  **Classify Cells:** For each cell, plot points in Span Space (one per time step). Quantify temporal variation by the number of grid elements these points occupy [S1].
2.  **Recursive Tree Construction:**
    *   For a time interval $[i, j]$, consider node $N_i^j$.
    *   Check if a cell has low temporal variation: if all its $j-i+1$ corresponding points in Span Space are located within an area of $MaxVariation \times MaxVariation$ lattice elements (typically $2 \times 2$) [S2, unofficial].
    *   If yes: Store the cell in $N_i^j$ using its temporal extreme values $(min_i^j, max_i^j)$. Do not store it in descendants [S1][S2, unofficial].
    *   If no: Split the interval $[i, j]$ into two halves: $[i, i + (j - i + 1)/2 - 1]$ and $[i + (j - i + 1)/2, j]$. Recursively classify the cell into the subtrees [S2, unofficial].
3.  **Build Search Index:** For each node $N_i^j$:
    *   Create the Span Space lattice for the cells in this node [S1].
    *   Sort cells in each row (excluding diagonal) into the two lists (min ascending, max descending) [S2, unofficial].
    *   Create Interval Trees for diagonal lattice elements [S2, unofficial].

### Query Algorithm
To query an isosurface at time step $t$ with isovalue $V_{iso}$:

1.  **Traverse Tree:** Recursively traverse from root $N_i^j$ to child $N_a^b$ such that $a \le t \le b$, until the leaf node $N_t^t$ is reached [S2, unofficial].
2.  **Search at Each Node:** For each node on the path:
    *   Locate the lattice element $[l, l]$ containing the point $(V_{iso}, V_{iso})$ in the Span Space [S2, unofficial].
    *   Candidate isosurface cells are in the upper-left corner defined by vertical line $x = V_{iso}$ and horizontal line $y = V_{iso}$ [S2, unofficial].
    *   Collect candidates from three categories:
        1.  **Rows $R$ where $l < R \le L-1$:** In the list sorted by minimum values, collect cells from the start until the first cell has $min > V_{iso}$ [S2, unofficial].
        2.  **Row $l$:** In the list sorted by maximum values, collect cells from the start until the first cell has $max < V_{iso}$ [S2, unofficial].
        3.  **Diagonal $[l, l]$:** Collect cells from the Interval Tree at lattice element $[l, l]$ [S2, unofficial].
3.  **Triangulation:** Use the cells' actual data at time step $t$ to perform triangulation [S2, unofficial].
    *   *Note:* Cells found in non-leaf nodes are represented by temporal extremes. If the temporal extremes contain $V_{iso}$ but the time-specific extremes do not, these are false positives. Triangulation detects this and creates no triangles [S1][S2, unofficial].

### Incremental Search
An optimization for searching consecutive time steps by reusing results [S1].
*   **State:** Maintain a result vector of found cells and an integer vector $V[d]$ storing the cumulative number of cells found at each depth $d$ [S1].
*   **Step:** For time step $t_n$ given previous result and time step $t_o$:
    1.  Traverse binary tree from root to leaf for $t_n$ [S1].
    2.  In each node $N_i$ at depth $i$, check if $t_n$ and $t_o$ are in $N_i$'s time span [S1].
    3.  If yes: Reuse the first $V[i]$ cells from the result vector (skip search) [S1].
    4.  If no: Truncate result vector after $V[i-1]$ cells (common cells) and search the rest of the tree normally, updating $V$ [S1].
*   **Overhead:** At most $d$ integers, where $d$ is the maximum tree depth [S1].

### Memory Management
Nodes are output to a file. On query, nodes on the traversal path are brought into main memory [S2, unofficial].
*   **Replacement Policy:** Assign priority based on node depth. Smaller depth = higher priority (root has highest priority) [S2, unofficial].
*   **Eviction:** If memory is exceeded, replace the node with the lowest priority. If tied, use Least Recently Used (LRU) [S2, unofficial].

## Subtopics

### Incremental Search
An optimization for searching consecutive time steps in the THITree by reusing results from previous searches, leveraging the binary tree structure where parent nodes cover time spans of child nodes [S1]. It uses a result vector and an integer vector $V[d]$ for space overhead [S1]. Offers a performance gain of factor 3 when searching 50 consecutive time steps compared to 50 full searches [S1].

## Operating characteristics
*   **Complexity:** The interval tree method for diagonal elements has optimal efficiency of $O(\log N)$, where $N$ is the number of cells [S2, unofficial]. Collection from categories 1 and 2 requires no search [S2, unofficial].
*   **False Positives:** Rates range from ~0.1% (largest tree size) to 5% (smallest tree size). Default settings yield approximately 0.5% false positives [S1].
*   **Trade-offs:** Increasing $L$ reduces false positives but increases memory usage [S1]. Increasing MaxVariation reduces memory size but increases false positives [S1].
*   **Strengths:** Fast retrieval for time-dependent data; data reduction via temporal coherence; supports incremental search [S1].
*   **Limitations:** Non-leaf nodes may yield false positives (performance overhead, not incorrect geometry) [S2, unofficial]. High memory requirement for the full tree, requiring node fetching/replacement [S2, unofficial].

## Background
The THITree was introduced by Shen in 1998 [S1]. It was extended and implemented by the authors of [S1] for fast time-dependent isosurface extraction. It builds on the concept of Span Space [Livnat et al. 1996] and Interval Trees [Cignani et al. 1997]. It is used in visualization techniques such as ShellSplatting for hardware-accelerated direct volume rendering [S1].

## Sources
[S1] Fast time-dependent isosurface extraction and rendering, https://doi.org/10.1145/1037210.1037217, fetched 2026-10-30
[S2] NASA Technical Reports Server (NTRS) 20020073378: Isosurface Extraction in Time-Varying Fields Using a Temporal Hierarchical Index Tree, https://archive.org/details/NASA_NTRS_Archive_20020073378, fetched 2026-10-26 (unofficial)
