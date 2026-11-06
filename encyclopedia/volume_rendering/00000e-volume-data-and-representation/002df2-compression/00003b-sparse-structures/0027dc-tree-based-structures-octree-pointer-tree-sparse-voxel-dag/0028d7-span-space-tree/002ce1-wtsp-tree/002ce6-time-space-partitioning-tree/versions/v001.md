# Time-Space Partitioning Tree


## Overview
The Time-Space Partitioning Tree (TSP) is a hierarchical data structure designed for the compression and rendering of time-varying volumetric data [S1]. It addresses the challenge of exploiting both spatial coherence (via an octree) and temporal coherence (via a binary tree) within a single unified structure [S1]. Unlike methods that process space and time separately, TSP integrates them by embedding temporal information directly within spatial nodes, allowing for efficient traversal of dynamic volumes [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A sequence of 3D volumetric datasets (time steps).
*   **Output:** A compressed hierarchical structure enabling direct volume rendering.

**Data Structures:**
The TSP consists of two nested hierarchical components:
1.  **Spatial Skeleton:** An octree that partitions the 3D spatial domain.
2.  **Temporal Extension:** Each node in the octree contains a binary tree that stores temporal information [S1].

**Structure Layout:**
*   **Octree Node:** Represents a spatial region.
*   **Binary Tree Field:** Attached to each octree node. This binary tree spans the entire time sequence [S1]. It combines data from multiple time steps, effectively linking spatially corresponding regions across time.

**Algorithm Steps:**
1.  **Spatial Decomposition:** Construct an octree over the spatial domain of the volume data. Nodes are subdivided based on spatial coherence or data density [S1].
2.  **Temporal Linking:** For each octree node, construct a binary tree that represents the evolution of that spatial region across time steps [S1].
3.  **Data Integration:** The binary tree nodes store temporal data, allowing the structure to capture changes in the volume over time while maintaining spatial locality [S1].
4.  **Compression:** The combined structure is compressed, typically by encoding the octree structure and the temporal binary tree data together [S1].

**Parameters:**
*   **Octree Depth:** Determines spatial resolution. Not given in the sources.
*   **Binary Tree Depth:** Determines temporal resolution or granularity. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Temporal Coherence:** If there is little temporal coherence, the binary tree may become large, reducing compression efficiency.
*   **Spatial Sparsity:** If the volume is sparse, the octree may have many empty nodes, requiring efficient handling of null pointers or empty branches.

## Subtopics

### Wavelet-based TSP
A variant of the TSP that applies a 3D wavelet transformation to the original data before constructing the tree structure [S1]. This approach leverages the energy compaction properties of wavelets to improve compression ratios, particularly for smooth or structured data [S1]. It is chosen when the data benefits from frequency-domain decomposition, offering potentially better compression than standard TSP for certain data types [S1].

## Operating characteristics
**Time and Memory Complexity:**
*   **Memory:** Depends on the sparsity of the data and the depth of the octree and binary trees. The structure stores both spatial and temporal information, which can be memory-intensive if temporal coherence is low.
*   **Time:** Construction time involves building the octree and then the binary trees for each node. Rendering time depends on the traversal efficiency of the combined structure.

**Measured Performance:**
*   Specific performance metrics (compression ratio, rendering speed) are not provided in the sources [S1].

**Strengths and Limitations:**
*   **Strengths:** Exploits both spatial and temporal coherence; unified structure simplifies data management for time-varying volumes.
*   **Limitations:** Complexity of implementation; potential for high memory usage if temporal variations are complex.

**When to Use:**
*   Suitable for time-varying volumetric data with significant spatial and temporal coherence.

**When Not to Use:**
*   Not ideal for data with low temporal coherence or high spatial complexity without sparsity.

## Background
The TSP structure was introduced by Meier et al. [MS00] as a method to handle time-varying data by combining spatial and temporal partitioning [S1]. Subsequent work by Wang et al. [WCCA04] and She [She06] extended this concept to wavelet-based transformations [S1]. Other related approaches include hierarchical structures built per time step and grouped by Westermann et al. [Wes95] and Wang et al. [WYM10], as well as methods using vector quantization by Fout et al. [FAM∗ 05] and packing time steps into RGB channels by Nagayasu et al. [NIH08] [S1].

## Sources
[S1] A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
