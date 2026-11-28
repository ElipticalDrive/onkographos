# Neighbor Cluster Assignment (NCA)


## Overview
Neighbor Cluster Assignment (NCA) is a static data distribution scheme for parallel volume rendering where processors are assigned sets of spatially adjacent clusters. It solves the problem of load balancing in distributed rendering by leveraging object-space coherency to minimize communication costs during remapping. Unlike Scattered Cluster Assignment (SCA), which distributes clusters to balance preprocessing load, NCA groups neighbor clusters together, making it superior for visualizations involving many rotation operations due to its efficient hypergraph topology.

## How it works
NCA operates as a static distribution method, meaning the assignment of clusters to processors is determined at program startup and remains fixed throughout the execution. While cluster replication may occur, the "first owner" of any given cluster never changes [S1, unofficial].

**Inputs and Outputs:**
*   **Input:** A set of clusters derived from the volume data, where each cluster contains cells (e.g., tetrahedra) with associated geometric properties (size, shape). A hypergraph representation of the data where nodes are clusters and edges represent adjacency or interaction.
*   **Output:** A static mapping of clusters to processors.

**Data Structures:**
*   **Cluster:** A group of contiguous cells. Attributes include cell count, total volume, and geometric centroid.
*   **Hypergraph:** A graph structure where nodes represent clusters and hyperedges represent the projection overlap or spatial adjacency between clusters.

**Algorithm Steps:**
1.  **Cluster Generation:** Partition the volume data into clusters based on spatial locality.
2.  **Hypergraph Construction:** Build a hypergraph where connectivity reflects the likelihood of clusters projecting onto the same screen cells. Neighbor clusters have a higher probability of sharing screen space [S1, unofficial].
3.  **Assignment:** Assign sets of neighbor clusters to processors. The goal is to ensure that each processor receives a contiguous set of clusters in object space.
4.  **Static Lock:** The assignment is locked at startup. Subsequent remapping steps use this initial distribution as the baseline [S1, unofficial].

**Formulas and Parameters:**
*   Let $C$ be the set of clusters.
*   Let $P$ be the set of processors.
*   Let $f: C \rightarrow P$ be the assignment function.
*   For NCA, if $c_i, c_j$ are neighbor clusters, $f(c_i) \approx f(c_j)$ is encouraged to maintain object-space coherency.
*   **Typical Values:** Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Preprocessing Load Imbalance:** NCA assumes that neighbor clusters have similar preprocessing costs. If the volume data contains cells with varying sizes or shapes (e.g., mixed tetrahedral sizes), a processor assigned a set of large neighbor clusters will incur significantly higher preprocessing overhead than a processor assigned small clusters. This creates load imbalance in the preprocessing step, increasing execution time [S1, unofficial].
*   **Static Nature:** Because the assignment is static, it cannot adapt to dynamic changes in data density or view-dependent complexity during runtime.

## Subtopics

### Scattered Cluster Assignment
Scattered Cluster Assignment (SCA) is a static data distribution scheme where clusters are distributed as scattered as possible among processors, rather than grouped by spatial adjacency. Like NCA, it is static at startup and the first owner never changes [S1, unofficial]. SCA reduces preprocessing overhead by ensuring that processors handle clusters with varying cell sizes and shapes, thereby balancing the preprocessing load across the system. It is best suited for visualizations involving zooming operations, where local detail varies significantly. The disadvantage of SCA is a slight increase in communication amount due to a more complicated hypergraph topology during the remapping phase [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Static assignment implies $O(1)$ runtime reconfiguration cost, but preprocessing time is dependent on the uniformity of cell sizes within assigned clusters.
*   **Memory Complexity:** Requires storage for the static mapping and the hypergraph topology.
*   **Performance:** NCA minimizes communication cost during hypergraph partitioning because neighbor clusters are likely to project onto the same screen cells, creating a better hypergraph topology [S1, unofficial]. It is the best possible static data distribution scheme for visualizations with many rotation operations [S1, unofficial].
*   **Strengths:** High object-space coherency; minimized communication cost; optimal for rotational views.
*   **Limitations:** Prone to preprocessing load imbalance if cell sizes/shapes vary significantly; static nature prevents adaptation to dynamic scene changes.
*   **When to Use:** Use NCA when the visualization involves frequent rotations and the volume data has relatively uniform cell sizes.
*   **When Not to Use:** Avoid NCA if the volume data has highly variable cell sizes/shapes (leading to preprocessing imbalance) or if the visualization involves significant zooming operations (where SCA is superior) [S1, unofficial].

## Background
NCA was proposed as part of a hypergraph-partitioning based remapping model for image-space parallel volume rendering. It is designed to address the inefficiencies of static data distribution in parallel rendering environments by exploiting spatial locality. The method is detailed in research concerning remapping models for parallel volume rendering [S1, unofficial].

## Sources
[S1] A hypergraph-partitioning based remapping model for image-space parallel volume rendering, http://hdl.handle.net/11693/18191, fetched 2026-10-26 (unofficial)
