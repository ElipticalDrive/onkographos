# Dynamic load balancing using kd-trees

Also called: kd-tree based dynamic load balancing

## Overview
This method uses a hierarchical kd-tree to dynamically redistribute rendering work among distributed processors during volume rendering, addressing load imbalance caused by non-uniform data density or view-dependent occlusion [S1, unofficial]. It sets itself apart from static decomposition methods by allowing runtime adjustment of partitions based on load statistics stored in inner tree nodes, and from queue-based approaches by explicitly managing spatial partitioning and compositing order [S1, unofficial].

## How it works
**Inputs:**
*   Distributed volume data, partitioned into bricks or subdomains.
*   Current load statistics for each processing node (e.g., rendering time, work remaining).
*   View parameters (camera position, direction) if view-dependent balancing is used.

**Outputs:**
*   Reassigned partitions of work (bricks) to processing nodes.
*   Updated kd-tree structure reflecting the new distribution.

**Data Structures:**
*   **kd-tree:** A binary tree where each inner node represents a splitting plane.
    *   **Inner Nodes:** Store load statistics (e.g., total estimated cost, number of active rays, or occlusion metrics) aggregated from their children [S1, unofficial].
    *   **Leaf Nodes:** Store the effective partitions of work (e.g., specific bricks or voxel ranges) assigned to a processor [S1, unofficial].
    *   **Splitting:** At each level, the data space is split alternating along orthogonal axes (X, Y, Z) [S1, unofficial].

**Steps:**
1.  **Initialize kd-tree:** Construct a kd-tree over the volume domain. Assign initial partitions to leaf nodes. Populate inner nodes with initial load statistics [S1, unofficial].
2.  **Estimate Load:** For the current frame or time step, estimate the computational cost for each partition. This may involve ray casting costs, occlusion information, or data density metrics [S1, unofficial].
3.  **Aggregate Statistics:** Propagate load statistics up the tree. Each inner node aggregates the statistics of its children (e.g., sum of costs) [S1, unofficial].
4.  **Traverse and Balance:** Traverse the kd-tree to identify imbalances. Strategies include:
    *   Identifying overloaded subtrees and underloaded subtrees.
    *   Reassigning leaf nodes (partitions) from overloaded to underloaded processors to balance the load statistics at the root or relevant subtrees [S1, unofficial].
    *   Traversal order may vary based on the balancing strategy (e.g., depth-first, breadth-first) [S1, unofficial].
5.  **Redistribute Work:** Move the effective partitions (bricks) associated with reassigned leaf nodes to the target processors.
6.  **Update Tree:** Update the kd-tree structure and load statistics to reflect the new distribution [S1, unofficial].
7.  **Render:** Processors render their assigned partitions. If determining compositing order, the kd-tree can be used to establish the front-to-back order of partial rays [S1, unofficial].

**Formulas:**
*   **Splitting Plane:** Not explicitly defined in sources. Typically, the median value along the chosen axis is used to split the data points or volume domain [G].
*   **Load Aggregation:** $L_{node} = \sum_{c \in children} L_c$ for inner nodes, where $L$ is the load statistic [G].

**Parameters:**
*   **Splitting Axis:** Alternates X, Y, Z. Not given in the sources.
*   **Load Statistic Type:** Cost, occlusion, etc. Not given in the sources.
*   **Balancing Threshold:** Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Boundary Data Replication:** When using the kd-tree to determine the compositing order of partial rays, replication of boundary data between adjacent partitions is necessary to ensure correct blending at partition boundaries [S1, unofficial].
*   **Precision:** Not given in the sources.
*   **Degenerate Input:** Not given in the sources.

## Subtopics

### [Incremental Load Balancing with Brick Caching](9862-incremental-load-balancing-with-brick-caching.md)
A dynamic load balancing technique for distributed volume rendering that incrementally moves bricks between cluster nodes to correct imbalance, using a caching mechanism to reduce transfer costs when reversing moves. It sets itself apart from full-tree balancing methods by processing only one node per frame to respect limited computation time, and from naive incremental methods by employing a brick cache to mitigate oscillation and stagnation caused by high network transfer costs.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Not given in the sources.
*   **Strengths:** Allows dynamic adjustment to non-uniform workloads; supports empty-space skipping and compositing order determination [S1, unofficial].
*   **Limitations:** Requires replication of boundary data for correct compositing order; overhead of tree maintenance and traversal [S1, unofficial].
*   **When to Use:** When load imbalance is significant and dynamic, and view-dependent effects are important [S1, unofficial].
*   **When Not to Use:** When static decomposition is sufficient, or when boundary replication overhead is prohibitive [S1, unofficial].

## Background
Originated in the context of volume visualization and compositing on large-scale displays, using hierarchical data structures like kd-trees to manage load statistics and redistribute work [S1, unofficial]. It builds on earlier work using kd-trees for empty-space skipping and spatial partitioning [S1, unofficial].

## Sources
[S1] Volume Visualization and Compositing on Large-Scale Displays Using Handheld Touchscreen Interaction, https://doi.org/10.25781/kaust-d5m7l, fetched 2026-10-26 (unofficial)
