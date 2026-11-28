# Hypergraph Coarsening

Also called: Coarsening Clustering

## Overview
Hypergraph coarsening is the reduction phase of multilevel hypergraph partitioning, where vertices are clustered into supernodes to create a smaller, structurally similar hypergraph [S1, unofficial]. It differs from graph coarsening by handling hyperedges (which connect arbitrary numbers of vertices) rather than simple pairwise edges, allowing it to preserve complex multi-way dependencies critical for volume rendering data decomposition. It serves as the first step in the multilevel paradigm, reducing problem size to enable efficient initial partitioning before uncoarsening [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A hypergraph $H = (V, E)$, where $V$ is the set of vertices and $E$ is the set of hyperedges.
*   **Output:** A coarser hypergraph $H' = (V', E')$ with $|V'| < |V|$.

**Data Structures:**
*   **Hypergraph:** Represented as an adjacency structure. Each vertex $v \in V$ maintains a list of incident hyperedges. Each hyperedge $e \in E$ maintains a list of incident vertices.
*   **Matched Status:** A boolean flag or set associated with each vertex to track if it has been assigned to a supernode.

**Steps:**
1.  **Initialize:** Mark all vertices in $V$ as unmatched. Initialize $V'$ and $E'$ as empty.
2.  **Cluster Vertices:** Select a clustering strategy (Matching-based or Agglomerative) to group vertices into supernodes.
    *   *Matching-based:* Iterate through vertices in a random or predetermined order. For each unmatched vertex $u$, find an unmatched neighbor $v$ (based on interaction metrics) and form a supernode $\{u, v\}$. Mark both as matched.
    *   *Agglomerative:* Allow a vertex to join an existing supernode or form a new one with multiple vertices. A single vertex may also form a supernode if no suitable partners exist.
3.  **Construct Coarser Hypergraph:**
    *   For each supernode $s \in V'$, create a new vertex.
    *   For each original hyperedge $e \in E$, determine the set of supernodes $S_e$ that contain at least one vertex from $e$.
    *   Create a new hyperedge $e' \in E'$ connecting all supernodes in $S_e$. If $|S_e| = 1$, the hyperedge may be discarded or kept as a self-loop depending on implementation [S1, unofficial].
4.  **Repeat:** Use $H'$ as input for the next coarsening level.

**Formulas:**
*   Let $M: V \to V'$ be the mapping from original vertices to supernodes.
*   $V' = \{ s_1, s_2, ..., s_k \}$ where each $s_i$ is a subset of $V$.
*   $E' = \{ e' \mid e' = \{ M(v) \mid v \in e \}, e \in E \}$.

**Parameters:**
*   **Clustering Metric:** Not given in the sources. Literature mentions metrics for determining vertex visit order and identifying highly interacting vertices [S1, unofficial].
*   **Order:** Random or predetermined [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Isolated Vertices:** Vertices with no hyperedges or unmatched vertices at the end of a pass must be handled (e.g., forming singleton supernodes) [S1, unofficial].
*   **Hyperedge Collapse:** If all vertices in a hyperedge map to the same supernode, the hyperedge becomes redundant.

## Subtopics

### [RM-MeTiS Coarsening](23606-rm-metis-coarsening.md)
RM-MeTiS Coarsening is a graph coarsening strategy for object-space decomposition that maintains a mapping constraint by distinguishing between 't-nodes' (texture/data nodes) and 'p-nodes' (processor nodes), ensuring each supernode contains at most one p-node.

## Operating characteristics
*   **Time Complexity:** Not given in the sources. Typically linear or near-linear in the size of the hypergraph per level.
*   **Memory Complexity:** Proportional to the size of the hypergraph. Memory usage decreases as the graph coarsens.
*   **Scaling:** Reduces problem size significantly, enabling partitioning of large graphs that would otherwise be intractable.
*   **Strengths:** Preserves multi-way connectivity; reduces computational cost for subsequent partitioning steps.
*   **Limitations:** Quality of coarsening affects final partition quality; choice of clustering metric is critical.

## Background
Hypergraph coarsening is a fundamental technique in multilevel partitioning algorithms. It was developed to handle the complexity of hypergraphs in parallel computing contexts, such as volume rendering remapping. The specific application to image-space parallel volume rendering is described in [S1, unofficial].

## Sources
[S1] A hypergraph-partitioning based remapping model for image-space parallel volume rendering, http://hdl.handle.net/11693/18191, fetched 2026-10-26 (unofficial)
