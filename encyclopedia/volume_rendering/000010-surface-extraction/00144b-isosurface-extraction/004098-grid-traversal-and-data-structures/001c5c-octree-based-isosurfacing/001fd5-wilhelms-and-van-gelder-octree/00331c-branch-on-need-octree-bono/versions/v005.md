# Branch-on-Need Octree (BONO)
Also called: Branch-On-Need Octree (BONO) for time-variant data, Incomplete pointer octree, branch-on-need strategy
## Overview
The Branch-on-Need Octree (BONO) is a space-efficient geometric decomposition technique for accelerating isosurface extraction from volumetric data [S1]. It solves the problem of excessive memory usage and traversal overhead in traditional uniform octrees by partitioning space non-uniformly to minimize empty subtrees and the ratio of nodes to data points [S4, unofficial]. Unlike its parent [Wilhelms and Van Gelder Octree](2107-wilhelms-and-van-gelder-octree.md), which may use uniform subdivision, BONO specifically optimizes for non-power-of-two dimensions and groups eight cells under a single leaf node to reduce memory footprint [S1]. It sets itself apart from sibling [View-Dependent Isosurface Extraction](2309-view-dependent-isosurface-extraction.md) by focusing on static spatial pruning via extreme value propagation rather than view-dependent level-of-detail adjustments [S1].
## How it works
**Inputs:** A 3D scalar volume grid $V$ of dimensions $D_x \times D_y \times D_z$ [S1].
**Outputs:** An octree data structure where leaf nodes contain bounding box extremes for groups of cells, enabling traversal to skip branches that do not contain the isosurface [S1].

**Data Structures:**
The BONO is an incomplete pointer octree [S4, unofficial]. Each internal node contains pointers to up to eight children [S4, unofficial]. Each leaf node references a group of eight cells (or fewer if at volume boundaries) [S1].
*   **Leaf Node Layout:** Contains two scalar values: `min_value` and `max_value` [S1].
*   **Cell Indexing:** One leaf node covers eight adjacent cells in the volume grid [S1]. This indexing scheme reduces memory requirements because one pair of extreme values covers eight cells, whereas span space methods typically index individual cells [S1].

**Construction Algorithm:**
1.  **Non-Uniform Subdivision:** Partition the volume such that the "lower" subdivision in each direction covers the largest possible power of two cells [S1]. This strategy ensures that when volume dimensions are not powers of two, the resulting tree has fewer nodes than even-subdivision schemes, making traversal more efficient [S1].
2.  **Leaf Assignment:** Assign groups of eight cells to leaf nodes. Along the edges of the volume, leaf nodes may reference fewer than eight cells [S1].
3.  **Extreme Value Propagation:** During construction, propagate the minimum and maximum scalar values up the tree [S1]. For each node, compute its `min_value` as the minimum of its children's `min_value`s and its `max_value` as the maximum of its children's `max_value`s [S1].
4.  **Edge Hashing (Optional):** To exploit spatial coherence, maintain a hash table of edges [S1]. After the initial interpolation of a point along an edge, cells sharing that edge access the hash entry to avoid recomputing the interpolation [S1].

**Extraction Algorithm:**
1.  **Traversal:** Traverse the octree starting from the root [S1].
2.  **Pruning Condition:** For each node, check if the isovalue $I$ lies within the node's range:
    $$ \text{min\_value} < I < \text{max\_value} $$
    If this condition is false, the branch does not contain the isosurface and is pruned (skipped) [S1].
3.  **Leaf Processing:** If the condition is true, process the leaf node [S1]. Extract the mesh from the eight cells referenced by the leaf using a standard method (e.g., Marching Cubes) [S1].
4.  **Seed Set Propagation (Alternative):** An alternative technique involves propagating the isosurface from a set of seed cells [S1]. This method groups seed cells into a hierarchical search structure and combines aspects of geometric decomposition and span space algorithms [S1].

**Parameters:**
*   **Isovalue ($I$):** The scalar threshold for extraction.
*   **Hash Table:** Optional. Size depends on the number of unique edges in the volume [S1].

**Edge Cases and Pitfalls:**
*   **Non-Power-of-Two Dimensions:** The branch-on-need strategy produces fewer nodes than even-subdivision schemes when dimensions are not powers of two, which is an efficiency gain [S1].
*   **Multiple Parent Nodes:** Due to non-uniform subdivision, BONO may require multiple parent nodes at deeper regions of the tree where a traditional octree would only require one, potentially causing added traversal steps [S4, unofficial].
*   **Hashing Limitations:** The non-uniform nature of subdivision precludes the use of a pure coordinate system (e.g., Frisken and Perry) as a direct hashing scheme [S4, unofficial].
*   **Memory vs. Performance:** While BONO reduces memory by indexing groups of cells, algorithms consuming larger amounts of memory do not necessarily yield better performance compared to simpler methods [S1].

## Subtopics

### [Temporal Hierarchical Index Tree (THITree)](15641-temporal-hierarchical-index-tree-thitree.md)
The Temporal Hierarchical Index Tree (THITree) is a data structure designed for fast isosurface extraction from time-dependent volumetric data sets. It solves the problem that extracting and rendering isosurfaces separately for each time step is too slow for interactive exploration of large time-varying data. It sets itself apart from its parent [Branch-on-Need Octree (BONO)](6935-branch-on-need-octree-bono.md) and its temporal sibling Temporal Branch-on-Need Tree (TBON) by exploiting temporal coherence: cells that remain constant over a time interval are stored only once for that entire span, achieving significant data reduction and faster retrieval. Unlike the Temporal Spanning Tree (TSP Tree), which incurs huge storage overhead by storing a full binary time tree per spatial node, THITree uses a shared Binary Time Tree with Span Space and Interval Trees for efficient interval search.

### Temporal Branch-on-Need Octree (T-BON)
A temporal indexing scheme that exploits spatial coherence by creating a BON octree for each time step, using the same branching structure for all time steps to amortize storage overhead [S3]. It creates a BON octree for each time step and stores data for each time step independently [S3]. It does not incorporate any optimization for exploiting temporal coherence [S3].

## Operating characteristics
**Time Complexity:** Traversal time is proportional to the number of nodes containing the isosurface [S1]. The non-uniform subdivision reduces the total number of nodes compared to uniform octrees for non-power-of-two dimensions, improving traversal efficiency [S1].
**Memory Complexity:** Memory usage is significantly reduced compared to span space methods because leaf nodes index eight cells simultaneously rather than individual cells [S1]. The ratio of nodes to data points is minimized [S4, unofficial].
**Measured Performance:** On the Head128 dataset ($128^3$ CT scan), the average execution time for BONO is 0.58 seconds [S1]. On the Rage256 dataset ($256^3$ CFD simulation), BONO achieves the best performance among the listed geometric decomposition techniques, though the exact average execution time is not explicitly listed in the source text [S1].
**Strengths:** Space-efficient; fewer nodes for non-power-of-two dimensions; reduced memory footprint by grouping cells [S1].
**Limitations:** Non-uniform subdivision may cause added traversal steps due to multiple parent nodes at deeper regions [S4, unofficial]; non-uniformity prevents direct coordinate hashing [S4, unofficial].
**When to use:** When memory is constrained and the volume dimensions are not powers of two [S1].
**When not to use:** When uniform subdivision allows for simpler coordinate-based hashing or when the overhead of multiple parent nodes outweighs the benefits of reduced node count [S4, unofficial].

## Background
The Branch-on-Need Octree (BONO) was proposed by Wilhelms and Van Gelder in 1992 [S4, unofficial]. It is a geometric decomposition technique that uses an octree structure to accelerate isosurface extraction [S1]. The method was designed to create an octree with the fewest possible empty subtrees within interior nodes, thereby minimizing the ratio of nodes to data points and saving space [S4, unofficial]. It has been compared against other isosurface extraction techniques such as Marching Cubes, Seed Set propagation, NOISE, ISSUE, and the Interval Tree [S1]. Sutton and Hansen later expanded the BONO approach to time-variant data for fast out-of-core rendering of time-variant isosurfaces [S2, unofficial].

## Sources
[S1] A Case Study of Isosurface Extraction Algorithm Performance, https://doi.org/10.1007/978-3-7091-6783-0_26, fetched 2026-10-26
[S2] Overview of Volume Rendering - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/papers/volvisOverview.pdf, fetched 2026-10-27 (unofficial)
[S3] Modeling and Visualization Approaches for Time-Varying Volumetric Data, https://doi.org/10.1007/978-3-540-89646-3_100, fetched 2026-10-30
[S4] PDF A Survey of Octree Volume Rendering Methods, https://www.sci.utah.edu/~knolla/octsurvey.pdf, fetched 2026-10-29 (unofficial)
