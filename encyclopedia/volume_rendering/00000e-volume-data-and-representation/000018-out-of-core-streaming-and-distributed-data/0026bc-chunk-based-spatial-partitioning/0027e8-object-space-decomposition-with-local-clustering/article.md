# Object-space decomposition with local clustering

Also called: Graph-based clustering for unstructured grids, Local clustering via MeTiS

## Overview
Object-space decomposition with local clustering partitions an unstructured volume grid into topological subsets (clusters) to enable parallel rendering across distributed processors [S1, unofficial]. It differs from spatial partitioning methods by grouping cells based on mesh connectivity rather than geometric bounding boxes, which minimizes inter-processor communication during ray casting [S1, unofficial]. This method serves as the static preprocessing foundation for the Remapping Phase, which dynamically adjusts these clusters based on viewing parameters [S1, unofficial].

## How it works
**Inputs:**
*   Unstructured grid (e.g., tetrahedral mesh) with cell data.
*   Number of processors $A$.
*   Global graph $Q_v$ (view-independent visualization graph) representing cell adjacencies.

**Outputs:**
*   Global clustering solution $C = [C_1, C_2, ..., C_A]$.
*   Local data structures (`ClusterData`, `ClusterMapArray`) on each processor.
*   Contracted visualization graph $Q_c$ and migration cost function $T_c$.

**Data Structures:**
1.  **ClusterData**: Maintained by each processor for its local cell-clusters.
    *   `VertexArray`: Local vertex indices for the cluster.
    *   `CellArray`: Local cell indices. Neighbor-cell identifiers (originally global cell IDs) are replaced by `(global_cluster_id, local_index)` tuples to maintain inter-cluster connectivity [S1, unofficial].
    *   `TotalVolume`: Sum of volumes (in World Space Coordinates, WSC) of constituent cells [S1, unofficial].
2.  **ClusterMapArray**: Maintained by each processor for *all* clusters in the environment.
    *   Component 1: Current mapping $M_e$ (processor ID owning the cluster).
    *   Component 2: Pointer to `ClusterData` if the cluster is local to this processor [S1, unofficial].
3.  **Contracted Visualization Graph $Q_c = (N_c, E_c)$**:
    *   Nodes $N_c$: Clusters $C_i$.
    *   Edges $E_c$: Edge $C_{ij}$ exists if clusters $C_i$ and $C_j$ share at least one face via neighbor cells [S1, unofficial].

**Algorithm Steps:**

1.  **Construct Local Graphs:** Each processor $P_k$ ($k=1 \dots A$) constructs its local view-independent visualization graph $Q_k$. This is the subgraph of the global graph $Q_v$ induced by the node subset corresponding to the local cells of $P_k$ [S1, unofficial].

2.  **Local Clustering:** Each processor $P_k$ concurrently performs $q_k$-way partitioning of its local graph $Q_k$ using MeTiS [S1, unofficial].
    *   **Parameter Selection:** The local number of resulting clusters $q_k$ is selected to be proportional to the total node weight $w_k$ in the local graph:
        $$ \sum_{k=1}^{A} q_k = c \cdot a $$
        where $c$ is a constant and $a$ relates to the total weight distribution. This scheme reduces variations in cluster weights to improve the efficiency of subsequent graph-partitioning based remapping [S1, unofficial].
    *   The $q_k$-way partitioning in each processor effectively constitutes a $A$-way global clustering solution $C$ [S1, unofficial].

3.  **Update Connectivity:** In the `CellArray` structure of each local cluster, the four global neighbor-cell identifiers of each cell are replaced by the global-cluster identifiers and local indices of the respective neighbor cells. This maintains connectivity information between cell-clusters within the local indexing scheme [S1, unofficial].

4.  **Construct Contracted Graph $Q_c$:**
    *   Each processor constructs its local portion of $Q_c$.
    *   An all-to-all broadcast operation replicates the global graph $Q_c$ in each processor. This ensures every processor knows the topology of the entire clustered mesh [S1, unofficial].

5.  **Compute Migration Costs $T_c$:**
    *   The migration cost function $T_c$ is defined as the ratio of the size of the `ClusterData` structure to the size of an individual ray-segment information.
    *   Since each processor only knows the costs of its local clusters, an all-to-all broadcast operation distributes these costs so every processor is aware of the migration costs of all clusters [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Parallel Local Clustering
# Input: Global Graph Q_v, Number of Processors A, Local Cells for P_k
# Output: Local Clusters, ClusterData, ClusterMapArray, Global Graph Q_c, Migration Costs T_c

def local_clustering(P_k, Q_v, local_cells):
    # Step 1: Construct Local Graph
    # Q_k is the subgraph of Q_v induced by local_cells
    Q_k = construct_subgraph(Q_v, local_cells)
    
    # Step 2: Determine Local Cluster Count
    # w_k is total node weight in Q_k
    w_k = sum_node_weights(Q_k)
    # q_k is proportional to w_k such that sum(q_k) = c * a
    # [G] Implementation detail: q_k = round(w_k / sum(all w_j) * A)
    q_k = determine_cluster_count(w_k, A)
    
    # Step 3: Perform MeTiS Partitioning
    # Returns partition labels for nodes in Q_k
    partition_labels = metis_partition(Q_k, q_k)
    
    # Step 4: Build ClusterData Structures
    local_clusters = []
    for cluster_id in range(q_k):
        cluster_cells = [cell for cell, label in zip(local_cells, partition_labels) if label == cluster_id]
        
        # Build VertexArray and CellArray with local indexing
        vertex_array = build_vertex_array(cluster_cells)
        cell_array = build_cell_array(cluster_cells)
        
        # Replace global neighbor IDs with (global_cluster_id, local_index)
        update_neighbor_references(cell_array, partition_labels, local_cells)
        
        # Compute total volume in WSC
        total_volume = sum(cell.volume for cell in cluster_cells)
        
        cluster_data = ClusterData(vertex_array, cell_array, total_volume)
        local_clusters.append(cluster_data)
        
    # Step 5: Construct Local Portion of Contracted Graph Q_c
    # Nodes are clusters, edges exist if clusters share a face
    local_Q_c_nodes = local_clusters
    local_Q_c_edges = find_inter_cluster_faces(local_clusters)
    
    # Step 6: Compute Local Migration Costs
    # Cost = size(ClusterData) / size(ray_segment_info)
    ray_segment_size = get_ray_segment_size()
    local_migration_costs = []
    for cluster_data in local_clusters:
        cost = size_of(cluster_data) / ray_segment_size
        local_migration_costs.append(cost)
        
    # Step 7: All-to-All Broadcasts
    # Broadcast local Q_c portions to build global Q_c
    global_Q_c = all_to_all_broadcast_graph(local_Q_c_nodes, local_Q_c_edges)
    
    # Broadcast migration costs so every processor knows all costs
    global_migration_costs = all_to_all_broadcast_costs(local_migration_costs)
    
    # Step 8: Build ClusterMapArray
    # Each processor maintains mapping for ALL clusters
    # [G] Requires global knowledge of cluster IDs and owning processors
    # This is typically established via the partition labels and global ID assignment
    cluster_map_array = build_global_cluster_map(local_clusters, global_Q_c)
    
    return local_clusters, global_Q_c, global_migration_costs, cluster_map_array
```

## Subtopics

### [Remapping Phase](4250-remapping-phase.md)
A view-dependent preprocessing step that reassigns data clusters to processors based on current viewing parameters to handle load imbalance. It uses the static clusters generated by this method as its initial state and applies a dynamic graph-partitioning loop (Graph Update, Graph Partitioning, Cluster Migration).

## Operating characteristics
*   **Time Complexity:** Dominated by the MeTiS partitioning step on each processor and the all-to-all broadcast operations. The complexity is $O(N \log A)$ for partitioning and $O(A \cdot N_{graph})$ for broadcasting the graph, where $N$ is the number of local cells and $N_{graph}$ is the size of the contracted graph [S1, unofficial].
*   **Memory Complexity:** Each processor stores the global contracted graph $Q_c$ and global migration costs $T_c$ in addition to its local `ClusterData`. This increases memory overhead compared to pure spatial partitioning [S1, unofficial].
*   **Strengths:** Minimizes inter-processor communication during rendering by keeping topologically connected cells together. Reduces view-independent preprocessing overhead by caching the contracted graph and migration costs [S1, unofficial].
*   **Limitations:** Static nature requires the Remapping Phase for view-dependent load balancing. All-to-all broadcasts can become a bottleneck for very large numbers of processors or complex meshes [S1, unofficial].
*   **When to use:** Unstructured grids where topological connectivity is critical for minimizing communication, and where the view changes frequently enough to warrant the Remapping Phase [S1, unofficial].

## Background
Originated in the context of parallel direct volume rendering of unstructured grids [S1, unofficial]. The method was developed to address the inefficiencies of spatial partitioning on unstructured data, where cells are not regularly spaced. It leverages MeTiS, a widely used library for graph partitioning, to achieve balanced and low-cut partitions [S1, unofficial].

## Sources
[S1] Parallel direct volume rendering of unstructured grids based on object-space decomposition, http://hdl.handle.net/11693/17897, fetched 2026-10-26 (unofficial)
