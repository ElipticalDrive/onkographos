# Object-space decomposition with local clustering

Also called: Graph-based clustering for unstructured grids, Local clustering via MeTiS

## Overview
Object-space decomposition with local clustering is a parallel volume rendering technique for unstructured grids that partitions the global dataset into static, view-independent clusters to minimize preprocessing overhead [S1, unofficial]. It differs from uniform block partitioning by using graph-based clustering (via MeTiS) to group geometrically connected tetrahedral cells into irregular clusters, thereby reducing inter-processor communication during the subsequent view-dependent remapping phase [S1, unofficial].

## How it works
The method operates in two distinct phases: a view-independent preprocessing phase (local clustering) and a view-dependent remapping phase. This article details the local clustering phase, which builds the static foundation for the remapping phase described in [Remapping Phase](4250-remapping-phase.md).

### Inputs and Outputs
*   **Input:** A global unstructured grid (tetrahedral mesh) distributed across $A$ processors $P_1, \dots, P_A$. Each processor holds a subset of cells.
*   **Output:** A global clustering solution $C = [C_1, C_2, \dots, C_K]$ and associated data structures (`ClusterData`, `ClusterMapArray`, `Qc`, `Tc`) replicated or distributed across processors.

### Data Structures
The method relies on specific structures to maintain connectivity and mapping information within the local indexing scheme of each processor [S1, unofficial].

1.  **`ClusterData`**: Maintained by each processor for each of its local cell-clusters.
    *   **`VertexArray`**: Stores vertex data for the cluster. Uses local indexing.
    *   **`CellArray`**: Stores tetrahedral cell data for the cluster. Uses local indexing.
        *   **Connectivity**: The four global neighbor-cell identifiers of each cell are replaced by **global-cluster identifiers** and **local indices** of the respective neighbor cells. This maintains connectivity between cell-clusters within the local indexing scheme.
    *   **`TotalVolume`**: The sum of the volumes (in World Space Coordinates, WSC) of all constituent cells in the cluster. This is used later for estimating view-dependent sampling computational weights.

2.  **`ClusterMapArray`**: Maintained by each processor for *all* cell-clusters in the global environment.
    *   **Component 1**: The current mapping of the cluster (the function $M_e$), indicating which processor owns the cluster.
    *   **Component 2**: A pointer to the `ClusterData` structure if the cluster is local to the current processor; otherwise, null or a placeholder.

3.  **`Qc` (Contracted Visualization Graph)**: A view-independent graph $Q_c = (N_c, E_c)$.
    *   **Nodes ($N_c$)**: Represent the clusters $C_i$.
    *   **Edges ($E_c$)**: An edge $C_{ij}$ exists between clusters $C_i$ and $C_j$ if they share at least one face through two neighbor cells in these two clusters.
    *   **Properties**: The topology of $Q_c$ is identical to the view-dependent visualization graph topology but is static. It differs from the view-dependent graph only in node and edge weights.

4.  **`Tc` (Cluster-Migration-Cost Function)**: A view-independent cost function.
    *   **Definition**: The migration cost of a cell-cluster is the ratio of the size of its associated `ClusterData` structure to the size of an individual ray-segment information.
    *   **Property**: Does not change with viewing parameters.

### Algorithm Steps
The local clustering phase proceeds as follows [S1, unofficial]:

1.  **Construct Local Visualization Graph**: Each processor $P_k$ constructs its local view-independent visualization graph $Q_v^k$. This graph is the subgraph of the global graph $Q_v$ induced by the node subset corresponding to the local cells of $P_k$.
2.  **Local Clustering via MeTiS**: Each processor $P_k$ concurrently performs a $q_k$-way clustering of its local graph $Q_v^k$ using the MeTiS graph partitioning library.
    *   **Parameter Selection**: The local number of resulting clusters $q_k$ is selected to be proportional to the total node weight in its local graph. Specifically, $\sum_{k=1}^{A} q_k = c \cdot a$, where $c$ and $a$ are constants. This scheme reduces variations in cluster weights to improve the efficiency of the subsequent graph-partitioning based remapping.
    *   **Global Solution**: The $q_k$-way partitioning performed in each processor $P_k$ effectively constitutes a $K$-way global clustering solution $C = [C_1, C_2, \dots, C_K]$.
3.  **Update Connectivity**: In the `CellArray` structure of each local cluster, the four global neighbor-cell identifiers of each cell are replaced by the global-cluster identifiers and local indices of the respective neighbor cells. This ensures connectivity information between cell-clusters is preserved within the local indexing scheme.
4.  **Construct Local Contracted Graph**: Each processor constructs its local portion of the contracted visualization graph $Q_c$.
5.  **Replicate Global Graph**: An all-to-all broadcast operation is performed so that a copy of the global graph $Q_c$ is replicated in each processor. This allows each processor to know the topology of the entire cluster interaction graph.
6.  **Broadcast Migration Costs**: Since each processor only knows the migration costs of its local clusters, an all-to-all broadcast operation is performed to let each processor become aware of the migration costs of all clusters. This completes the construction of the view-independent portions of the remapping tuple $(M_e, T_c)$.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Assumptions:
# - Global grid is distributed across A processors.
# - Each processor P_k has local cells LocalCells_k.
# - MeTiS library is available for graph partitioning.

def local_clustering_phase(processor_id, A, local_cells):
    """
    Performs the view-independent local clustering phase.
    """
    # Step 1: Construct Local Visualization Graph
    # Build graph Q_v^k induced by local cells.
    # Nodes are cells. Edges connect cells sharing a face.
    local_graph = build_local_visualization_graph(local_cells)
    
    # Step 2: Determine Local Cluster Count
    # Calculate total node weight (e.g., cell count) in local graph
    local_node_weight = count_nodes(local_graph)
    
    # Select q_k proportional to local node weight
    # Formula: sum(q_k for k in 1..A) = c * a
    # q_k = (local_node_weight / total_global_weight) * (c * a)
    # Note: c and a are system constants defined by the implementation.
    q_k = calculate_proportional_clusters(local_node_weight, A)
    
    # Step 3: Perform Local Clustering using MeTiS
    # Partition local_graph into q_k clusters
    # Returns: cluster_assignments (map: cell_id -> cluster_id)
    cluster_assignments = metis_partition(local_graph, q_k)
    
    # Step 4: Update Connectivity in CellArrays
    # For each local cluster, update neighbor references
    for cluster_id, cluster_cells in cluster_assignments.items():
        cluster_data = ClusterData()
        
        # Initialize VertexArray and CellArray with local indexing
        cluster_data.vertex_array = extract_vertices(cluster_cells)
        cluster_data.cell_array = extract_cells(cluster_cells)
        
        # Calculate total volume for this cluster
        cluster_data.total_volume = sum(cell.volume for cell in cluster_cells)
        
        # Replace global neighbor-cell IDs with (global_cluster_id, local_index)
        for cell in cluster_data.cell_array:
            for neighbor_cell_id in cell.original_neighbors:
                # Find which cluster the neighbor cell belongs to
                neighbor_cluster_id = cluster_assignments.get_neighbor_cluster(neighbor_cell_id)
                
                # Find the local index of the neighbor cell within its cluster
                # This requires a reverse map from cell_id to (cluster_id, local_index)
                local_neighbor_index = find_local_index(neighbor_cell_id, neighbor_cluster_id)
                
                cell.neighbors[neighbor_cell_id] = (neighbor_cluster_id, local_neighbor_index)
        
        # Store ClusterData
        store_cluster_data(cluster_id, cluster_data)

    # Step 5: Construct Local Portion of Contracted Graph Qc
    # Nodes are clusters. Edges exist if clusters share a face.
    local_contracted_graph = build_contracted_graph(cluster_assignments)
    
    # Step 6: All-to-All Broadcast for Global Graph Qc
    # Each processor sends its local portion of Qc to all others
    global_contracted_graph = all_to_all_broadcast(local_contracted_graph)
    
    # Step 7: Calculate and Broadcast Migration Costs Tc
    # Migration cost = size(ClusterData) / size(ray-segment info)
    migration_costs = {}
    for cluster_id, cluster_data in local_cluster_data.items():
        cost = size(cluster_data) / size(ray_segment_info)
        migration_costs[cluster_id] = cost
    
    # Broadcast local migration costs to all processors
    global_migration_costs = all_to_all_broadcast(migration_costs)
    
    # Store global mapping and costs for remapping phase
    store_cluster_map_array(global_contracted_graph, global_migration_costs)
```

### Parameters and Edge Cases
*   **Proportionality Constant**: The sum of local cluster counts $\sum q_k$ is set to $c \cdot a$. The exact values of $c$ and $a$ are not specified in the sources [S1, unofficial].
*   **MeTiS Configuration**: The specific MeTiS parameters (e.g., edge weight type, balancing factor) are not detailed, but the goal is to minimize edge cuts between clusters while balancing cluster sizes [S1, unofficial].
*   **Connectivity Replacement**: Replacing global cell IDs with (cluster ID, local index) is critical for maintaining connectivity across cluster boundaries without requiring global cell lookups during rendering [S1, unofficial].
*   **Volume Calculation**: The total volume is computed in World Space Coordinates (WSC) and is used for estimating view-dependent sampling weights in the remapping phase [S1, unofficial].

## Subtopics

### [Remapping Phase](4250-remapping-phase.md)
The view-dependent preprocessing step that adapts the static clustering to the current view by updating graph weights, repartitioning, and migrating clusters.

## Operating characteristics
*   **Preprocessing Overhead**: The local clustering phase is view-independent and performed once (or infrequently). It minimizes the overhead of the subsequent remapping phase by creating a stable, coarse-grained partitioning [S1, unofficial].
*   **Communication**: Requires two all-to-all broadcast operations (for $Q_c$ and $T_c$) after local clustering. This is a one-time cost per rendering session or significant view change [S1, unofficial].
*   **Load Balancing**: The proportional selection of $q_k$ aims to balance cluster weights, reducing load imbalance during the remapping phase [S1, unofficial].
*   **Scalability**: The method scales with the number of processors $A$ as each processor performs independent local clustering. However, the all-to-all broadcasts may become a bottleneck for very large $A$ [S1, unofficial].
*   **Strengths**: Reduces view-independent preprocessing overhead; handles unstructured grids effectively; minimizes inter-processor communication during rendering by grouping connected cells [S1, unofficial].
*   **Limitations**: Static clustering may not be optimal for all views; requires significant memory for global graph replication; migration costs are approximated [S1, unofficial].

## Background
The method originates from research on parallel direct volume rendering of unstructured grids. It was proposed to address the inefficiencies of uniform partitioning and the high cost of frequent view-dependent repartitioning. By decoupling the coarse, view-independent clustering from the fine, view-dependent remapping, the method achieves a balance between preprocessing cost and rendering performance [S1, unofficial].

## Sources
[S1] Parallel direct volume rendering of unstructured grids based on object-space decomposition, http://hdl.handle.net/11693/17897, fetched 2026-10-26 (unofficial)
