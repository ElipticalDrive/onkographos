# WTSP Tree Traversal and Parallel Rendering

Also called: Run-Time Parallel Volume Rendering, WTSP tree traversal, parallel volume rendering

## Overview
This method traverses the Wavelet-based Time-Space Partitioning (WTSP) tree at run time to reconstruct and render data blocks of varying spatio-temporal resolutions in parallel, composing the final image [S1]. It solves the problem of efficiently visualizing large-scale time-varying data on parallel clusters by eliminating parent-child data dependencies for reconstruction and balancing workload among processors [S1]. It is distinguished from its parent [WTSP Tree](5371-wtsp-tree.md) by its runtime execution strategy and from its sibling [Time-Space Partitioning Tree](5382-time-space-partitioning-tree.md) by its specific use of wavelet compression and error-guided bucketization for distributed rendering [S1].

## How it works
**Inputs:**
*   A pre-constructed WTSP tree containing wavelet-compressed multiresolution data [S1].
*   User-specified time step for visualization [S1].
*   Tolerances for spatial and temporal errors [S1].
*   A hierarchical spatial and temporal error metric calculated for each tree node [S1].

**Outputs:**
*   A final rendered image composed of parallel rendering tasks [S1].

**Data Structures:**
*   **WTSP Tree:** As defined in [WTSP Tree](5371-wtsp-tree.md), consisting of an octree skeleton where each node contains a binary time tree spanning the entire time sequence [S1].
*   **Distribution Units:** Partitions of the WTSP tree designed to eliminate data dependency [S1].
*   **Reconstructed Data:** Stored at nodes in selective WTSP tree levels to alleviate long chains of parent-child dependencies [S1].

**Steps:**
1.  **Partitioning:** Partition the WTSP tree into distribution units. This is done to eliminate parent-child data dependency for reconstruction and achieve load-balanced rendering [S1].
2.  **Distribution:** Distribute the partitioned data along hierarchical space-filling curves among different processors [S1].
3.  **Bucketization:** Use the hierarchical spatial and temporal error metric to guide the bucketization during distribution. This metric controls the tradeoff between image quality and rendering speed [S1].
4.  **Traversal:** At run time, traverse the WTSP tree according to the user-specified time step [S1].
5.  **Error-Guided Selection:** Traverse the tree according to tolerances of both spatial and temporal errors [S1].
6.  **Reconstruction:** Reconstruct data blocks of different spatio-temporal resolutions. To reduce reconstruction cost, reconstructed data is stored at nodes in selective WTSP tree levels, breaking long dependency chains [S1].
7.  **Parallel Rendering:** Render the reconstructed data blocks in parallel to compose the final image [S1].

**Pitfalls:**
*   Parent-child data dependency for reconstruction can cause long chains of dependencies, increasing reconstruction cost. This is mitigated by storing reconstructed data at nodes in selective WTSP tree levels [S1].

## Subtopics

### WTSP Tree Partition and Data Distribution
An algorithm to partition the WTSP tree into distribution units and distribute them among processors to eliminate parent-child data dependency for reconstruction and achieve load-balanced rendering [S1]. It partitions the tree, distributes data along hierarchical space-filling curves among processors, and uses a hierarchical spatial and temporal error metric to guide bucketization [S1]. The error metric is calculated for each tree node to control the tradeoff between image quality and rendering speed [S1].

## Operating characteristics
The algorithm reduces run-time communication cost to a minimum and ensures a well-balanced workload among processors when visualizing gigabytes of time-varying data on a PC cluster [S1]. It exploits both spatial and temporal locality and coherence of the underlying time-varying data, allowing flexible spatio-temporal level-of-detail data selection and retrieval at run time [S1]. The hierarchical spatial and temporal error metric is used to calculate approximate spatial and temporal errors for each tree node, controlling the run-time tradeoff between image quality and rendering speed [S1].

## Background
The method was introduced in a multiresolution volume rendering framework for large time-varying data visualization [S1]. It utilizes a wavelet-based time-space partitioning (WTSP) tree to convert time-varying data into a wavelet-compressed multiresolution data representation [S1]. The framework was designed to deploy multiresolution volume rendering over a PC cluster, addressing issues of data dependency and workload imbalance in parallel processing [S1].

## Sources
[S1] A Multiresolution Volume Rendering Framework for Large Time-Varying Data Visualization, https://diglib.eg.org/bitstream/handle/10.2312/VG.VG05.011-019/011-019.pdf?sequence=1, fetched 2026-10-26
