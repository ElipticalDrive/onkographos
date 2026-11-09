# Distributed Volume Ray Tracing with Global Kd-Tree
Also called: Distributed Cell Projection, Distributed server-side rendering, Ma and Crockett Distributed Cell Projection, Ma and Crockett distributed rendering

## Overview
A distributed rendering method for load balancing where adjacent tetrahedra are scattered across nodes rather than clustered, using a global kd-tree to restore ordering during rendering [S1, unofficial]. Unlike Kd-partitioning which clusters adjacent tetrahedra, this method scatters them to maintain load balance for close views, at the cost of requiring a global kd-tree [S1, unofficial]. It builds on [Scalable Distributed Volume Rendering (SDVR)](863-scalable-distributed-volume-rendering-sdvr.md) [S1, unofficial].

## How it works
**Inputs:** Volumetric tetrahedral dataset, camera parameters, number of rendering nodes [S1, unofficial].
**Outputs:** Rendered image [S1, unofficial].
**Data Structures:**
*   **Global Kd-Tree:** A spatial acceleration structure built over the entire tetrahedral dataset, accessible by all nodes [S1, unofficial].
*   **Local Cell List:** Per-node list of cells traversed by assigned rays [S1, unofficial].

**Steps:**
1.  **Data Distribution:** Distribute volumetric data using MPI so that adjacent tetrahedra are scattered over all available nodes rather than clustered [S1, unofficial].
2.  **Ray Assignment:** Interleave scan-lines among the rendering nodes to improve load balancing [S1, unofficial].
3.  **Traversal & Clustering:** Each node clusters all cells traversed by the assigned rays before sending the results back to the render nodes [S1, unofficial].
4.  **Ordering Restoration:** Use the global kd-tree to restore spatial ordering during the rendering process [S1, unofficial].

**Parameters:**
*   **MPI:** Used for distributed rendering communication [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Close Views:** The scattering strategy is specifically designed to handle load balancing issues that arise in close views, where clustering might lead to uneven workloads [S1, unofficial].
*   **Global Kd-Tree Overhead:** The requirement for a global kd-tree introduces additional complexity and potential synchronization overhead compared to partitioned approaches [S1, unofficial].

## Subtopics

### KD-Tree and Dynamic Programming for Clusters
A technique using a kd-tree and dynamic programming to optimize non-empty space tracking and data distribution balance on a volume-rendering cluster [S3].

## Operating characteristics
*   **Scaling:** Designed to maintain load balance for close views by scattering adjacent tetrahedra across nodes [S1, unofficial].
*   **Strengths:** Good load balancing for close views due to scattering of adjacent tetrahedra [S1, unofficial].
*   **Limitations:** Requires a global kd-tree, which may introduce overhead [S1, unofficial].
*   **When to use:** When load balancing for close views is critical and a global kd-tree can be maintained [S1, unofficial].
*   **When not to use:** When memory overhead for the global kd-tree is prohibitive or when close views are not a primary concern [S1, unofficial].

## Background
Originated from work by Ma and Crockett [Ma97], who distributed data for load balancing and used MPI for distributed rendering [S1, unofficial]. The goal was to present a benchmark system for comparison for which all mathematical operations are described in detail [S1, unofficial]. It is part of a broader set of distributed server-side rendering schemes proposed for large-scale volume rendering [S2, unofficial].

## Sources
[S1] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S2] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
[S3] Volume Visualization: A Technical Overview with a Focus on ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC3138940/, fetched 2026-10-27
