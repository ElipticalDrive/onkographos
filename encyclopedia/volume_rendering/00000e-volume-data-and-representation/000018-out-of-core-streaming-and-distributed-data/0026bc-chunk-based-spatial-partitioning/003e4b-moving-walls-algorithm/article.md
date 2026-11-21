# Moving Walls Algorithm
## Overview
The Moving Walls Algorithm partitions a volume dataset into cells, where each cell contains one or more closely cropped blocks of volume data, using the bounding rectangle parameters of cropped slices as input [S1, unofficial]. It solves the problem of distributing non-uniform, sparse volume data across a cluster by creating a view-independent partition that matches data block sizes to target memory constraints [S1, unofficial]. Unlike sibling methods that rely on uniform grid partitioning or object-space decomposition, this approach drives partitioning by tightly cropped subvolumes to follow natural volume boundaries [S1, unofficial].

## How it works
**Inputs:**
* Cropped slices of the volume data, defined by bounding rectangles of regions of interest derived from a mask volume [S1, unofficial].
* Target memory size for each group of data blocks [S1, unofficial].

**Outputs:**
* A flex-block partition: a view-independent partition of cells containing a combination of empty space and cropped subvolumes [S1, unofficial].
* A kd-tree extracted from the partition output [S1, unofficial].
* A dependency graph (look-up table) representing image priority order for eight quantized view directions, derived from the kd-tree [S1, unofficial].

**Data Structures:**
* **Flex-block:** A cell in the partition containing cropped subvolumes and empty space [S1, unofficial].
* **Dependency Graph:** A look-up table storing the image priority order for each of eight quantized view directions, used for ray-task scheduling [S1, unofficial].

**Steps:**
1. **Partitioning:** The algorithm partitions the scene into cells such that the total size of each group of data blocks is closely matched to the target memory size [S1, unofficial].
2. **Space Partitioning:** The bounding rectangle parameters of the cropped slices are used to partition the scene with a kd-tree space partition [S1, unofficial].
3. **Dynamic Programming Optimization:** Dynamic programming is applied to find a solution to a relaxed version of the Load-Balanced Node Distribution (LBND) problem, creating flex-blocks within the kd-tree-like partition [S1, unofficial].
4. **Tree Extraction:** The kd-tree is extracted from the output of the moving walls procedure [S1, unofficial].
5. **Dependency Graph Construction:** The image priority order for each of eight quantized view directions is extracted from the kd-tree and stored in a look-up table [S1, unofficial].
6. **Distribution:** Flex-block parameters, including the dependency graph, are distributed across the cluster along with the cropped slices [S1, unofficial].

**Formulas:**
Not given in the sources.

**Parameters:**
* **Target Memory Size:** The desired size for each group of data blocks [S1, unofficial].
* **Search Tree Depth:** The depth of the search tree is limited to prevent the recursion from becoming intractable, which limits the size of the data bricks with respect to the whole dataset [S1, unofficial].

**Edge Cases and Pitfalls:**
* **Memory Restriction:** Blocks are never created until the end of preprocessing because memory used by preprocessing unnecessarily restricts volume block sizes [S1, unofficial].
* **Intractability:** Without limiting the search tree depth, the recursion may become intractable [S1, unofficial].
* **Viewpoint Optimization:** The initial partitioning may be optimized with respect to a single viewpoint, though the dynamic programming output yields a view-independent partition [S1, unofficial].

## Subtopics

### Moving Walls DP
A dynamic programming algorithm that finds a solution to a relaxed version of the LBND problem [S1, unofficial]. It applies dynamic programming to the slab-projection slices [S1, unofficial]. The output is a flex-block partition, a view-independent partition of cells which contain a combination of empty space and cropped subvolumes [S1, unofficial]. It is chosen when user control over the tradeoff between distribution quality and algorithm run time is required [S1, unofficial].

## Operating characteristics
* **Time Complexity:** The algorithm's run time is influenced by the depth of the search tree, which is limited to prevent intractability [S1, unofficial]. The dynamic programming approach allows the user to control the tradeoff between distribution quality and run time [S1, unofficial].
* **Memory Complexity:** Memory usage is managed by delaying block creation until the end of preprocessing [S1, unofficial].
* **Strengths:** Follows natural volume boundaries via tightly cropped subvolumes; provides view-independent partitioning; allows control over distribution quality vs. run time [S1, unofficial].
* **Limitations:** The search tree depth limit restricts data brick size relative to the whole dataset [S1, unofficial]. The partition may initially be optimized with respect to a single viewpoint [S1, unofficial].
* **When to use:** When distributing sparse, non-uniform volume data across a cluster and needing to match data block sizes to target memory constraints while following natural boundaries [S1, unofficial].
* **When not to use:** Not given in the sources.

## Background
The Moving Walls Algorithm was introduced in the context of massive data management for distributed volume visualization [S1, unofficial]. It is part of a system that includes cell-tree peeling algorithms and ray dependency graphs for efficient cell-processing scheduling [S1, unofficial]. The algorithm was developed to handle large datasets like the Visible Korean data set, where slices are cropped and stored on disk for distribution to render-nodes [S1, unofficial]. Volume blocks are created locally by render-nodes or during preprocessing for distribution [S1, unofficial].

## Sources
[S1] Massive Data Management for Distributed Volume Visualization, http://hdl.handle.net/1951/47671, fetched 2026-10-26 (unofficial)
