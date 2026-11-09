# Graph-based Load Balancing for Parallel Subdivision

Also called: Bajaj’s Parallel Algorithm, Octree-based Partitioning with Metis Merging, Static and random subdivision with graph-based balancing

## Overview
This method balances parallel workloads for isosurface extraction by distributing volumetric blocks such that each process holds an equal number of active cells for any isovalue [S1]. It differs from spatial-only partitioning by analyzing the topological connectivity of active cells via graph partitioning, ensuring uniform computational load across parallel processes regardless of the specific isosurface threshold [S1].

## How it works
**Inputs:** A volumetric dataset; a target number of parallel processes $P$.
**Outputs:** A partitioning of the volume into $P$ segments, where each segment contains a subset of the data.

**Data Structures:**
1.  **Octree:** A space-partitioning structure where leaves are cubic boxes.
2.  **Adjacency Graph:** An undirected graph $G=(V, E)$ where $V$ is the set of octree leaves and $E$ represents valid merge connections.
3.  **Segment Map:** A mapping from each leaf to a segment ID $s \in \{0, \dots, P-1\}$.

**Steps:**

1.  **Initial Subdivision:** Subdivide the volume into fine-grain blocks using static and random subdivision among the $P$ nodes [S1]. Alternatively, construct an octree where leaves are subdivided at their center if the number of vertices exceeds a maximal limit $N_{max}$, reflecting point density [S2].

2.  **Graph Construction:**
    *   Create an undirected graph where nodes are the octree leaves [S2].
    *   Add an edge between two leaves $u$ and $v$ if:
        1.  They are spatial neighbors in the octree [S2].
        2.  Their volumes (or point counts) do not differ by a factor of more than 2 [S2].
    *   *Note:* In the static/random approach, blocks are examined to extract a graph representing the range of values assumed by cell points and the number of active cells intersected for each value [S1].

3.  **Graph Partitioning:**
    *   Run a graph partitioning algorithm (specifically Metis 4.0) on the constructed graph to merge leaves into $P$ balanced segments [S2].
    *   The goal is to ensure each parallel process has the same amount of active cells for any isovalue [S1].

4.  **Assignment:**
    *   **For Tetrahedral Meshes:** Iterate over all tetrahedra. For each tetrahedron, determine the segment indices of all four vertices. Assign the tetrahedron to the segment with the smallest index among its vertices [S2].
    *   **For Volumetric Blocks:** Distribute blocks having similar graph characteristics among the nodes [S1].

**Parameters:**
*   $N_{max}$: Maximum vertices per octree leaf. Not given in the sources.
*   Partitioning Tool: Metis 4.0 [S2].

**Edge Cases and Pitfalls:**
*   **Data Duplication:** The main drawback is the production of a large number of duplicated data, particularly when assigning tetrahedra based on vertex segment indices, as a single tetrahedron may be duplicated across multiple segments if its vertices belong to different segments [S1, S2].
*   **Boundary Shape:** The method creates well-shaped boundaries between segments sufficient for multi-triangulations, avoiding the need to compute tetrahedron centers [S2].

## Subtopics

## Operating characteristics
*   **Time Complexity:** Preprocessing analysis is executed mostly in parallel [S1]. Graph partitioning is NP-hard but approximated efficiently by Metis [S2].
*   **Memory Complexity:** High memory overhead due to data duplication [S1].
*   **Strengths:** Ensures uniform load for any isovalue; suitable for interactive environments [S1].
*   **Limitations:** Significant data duplication; requires preprocessing [S1].
*   **When to use:** When isovalue is unknown or varies, and uniform load is critical [S1].
*   **When not to use:** When memory bandwidth is constrained due to duplication [S1].

## Background
Originating from work by Bajaj et al. (referenced as [44] in [S1]), this algorithm was designed for parallel isosurface extraction in distributed environments [S1]. It was evaluated against pure data parallel and dynamic distribution algorithms [S1]. The octree-based variant was developed for segment-based tetrahedral meshing and rendering [S2].

## Sources
[S1] Parallel Isosurface Extraction for 3D Data Analysis Workflows in Distributed Environments, https://doi.org/10.1002/cpe.1710, fetched 2026-10-26
[S2] Segment-based tetrahedral meshing and rendering, https://doi.org/10.1145/1174429.1174507, fetched 2026-11-06
