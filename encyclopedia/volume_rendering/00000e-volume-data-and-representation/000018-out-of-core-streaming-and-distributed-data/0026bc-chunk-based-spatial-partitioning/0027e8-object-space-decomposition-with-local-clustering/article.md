# Object-space decomposition with local clustering

Also called: Graph-based clustering for unstructured grids, Local clustering via MeTiS

## Overview
Object-space decomposition with local clustering partitions an unstructured volume dataset into spatially coherent clusters to enable efficient parallel rendering by minimizing inter-processor communication and load imbalance [S1, unofficial]. Unlike uniform chunking, this method groups irregular tetrahedral cells into clusters based on topological connectivity using graph partitioning, ensuring that cells sharing faces remain within the same cluster [S1, unofficial]. It is distinguished from its siblings by its reliance on a view-independent preprocessing phase that constructs a contracted visualization graph, allowing the subsequent view-dependent remapping phase to adapt quickly to camera changes without re-partitioning the underlying geometry [S1, unofficial].

## How it works
The method operates in two distinct phases: a view-independent **Local Clustering Phase** and a view-dependent **Remapping Phase**. The clustering phase builds the static data structures, while the remapping phase adapts the distribution of these clusters to the current view.

### Inputs and Outputs
*   **Input:** Unstructured grid (tetrahedral cells), number of processors $A$, viewing parameters $v$ (used in remapping).
*   **Output:** A set of clusters $C = [C_1, C_2, ..., C_A]$ mapped to processors, and the data structures required for rendering and remapping.

### Data Structures
The method relies on specific structures to maintain connectivity and mapping information [S1, unofficial]:

1.  **CellArray**: Stores the cells belonging to a cluster.
    *   **Layout:** For each cell, it stores four neighbor-cell identifiers. In the context of clustering, these identifiers are replaced by **global-cluster identifiers** and **local indices** of the respective neighbor cells to maintain connectivity between clusters within the local indexing scheme.
2.  **VertexArray**: Stores the vertices belonging to a cluster. Uses local cell and vertex indexing.
3.  **ClusterData**: Maintained by each processor for its local cell-clusters.
    *   **Fields:**
        *   `VertexArray`: List of vertices.
        *   `CellArray`: List of cells.
        *   `TotalVolume`: The sum of the volumes (in World Space Coordinates, WSC) of all constituent cells in the cluster. This is used for estimating view-dependent sampling computational weights.
4.  **ClusterMapArray**: Maintained by each processor globally (replicated across all processors).
    *   **Layout:** For each cell-cluster in the overall environment, it holds:
        1.  The current mapping of the cluster (the processor ID $M_e$).
        2.  A pointer to the `ClusterData` structure if the cluster is local to that processor.
5.  **Contracted Visualization Graph ($Q_c$)**:
    *   **Nodes:** Clusters $C_i$.
    *   **Edges:** An edge $C_{ij}$ exists between clusters $C_i$ and $C_j$ if they share at least one face through two neighbor cells.
    *   **Weights:** Node weights depend on view-dependent sampling costs; edge weights depend on communication volume.

### Numbered Steps

#### Phase 1: Local Clustering (View-Independent)
1.  **Construct Local Graph:** Each processor $P_k$ constructs its local view-independent visualization graph $Q_v$. This graph is the subgraph of the global graph induced by the node subset corresponding to the local cells of $P_k$ [S1, unofficial].
2.  **Local Partitioning:** Each processor $P_k$ concurrently performs a $q_k$-way clustering of its local graph using the MeTiS graph partitioning library [S1, unofficial].
    *   **Parameter Selection:** The local number of resulting clusters $q_k$ is selected to be proportional to the total node weight in its local graph such that:
        $$ \sum_{k=1}^{A} q_k = c \cdot a $$
        where $A$ is the total number of processors, $a$ is a scaling factor, and $c$ is a constant. This scheme reduces variations in cluster weights to improve the efficiency of the subsequent graph-partitioning based remapping [S1, unofficial].
    *   **Global Solution:** The $q_k$-way partitioning performed in each processor $P_k$ effectively constitutes a $K$-way global clustering solution $C = [C_1, C_2, ..., C_K]$ [S1, unofficial].
3.  **Update Connectivity:** In the `CellArray` structure, the four global neighbor-cell identifiers of each cell are replaced by the global-cluster identifiers and local indices of the respective neighbor cells. This maintains connectivity information between cell-clusters within the local indexing scheme [S1, unofficial].
4.  **Assign Global IDs:** Each cluster in the overall environment is given a unique global cluster identifier, and the global mapping function $M_e$ is determined [S1, unofficial].
5.  **Build ClusterData:** Each processor maintains a `ClusterData` structure for each of its local cell-clusters. The `TotalVolume` is computed as the sum of the volumes of the constituent cells [S1, unofficial].
6.  **Build ClusterMapArray:** Each processor maintains a global `ClusterMapArray` holding the current mapping ($M_e$) and a pointer to the local `ClusterData` if applicable [S1, unofficial].
7.  **Construct Contracted Graph ($Q_c$):** Each processor constructs its local portion of the contracted visualization graph $Q_c$. The topology of $Q_c$ is identical to the view-independent contracted graph because the topology does not change with viewing parameters; only node and edge weights change [S1, unofficial].
8.  **Broadcast Graph:** A copy of the global graph $Q_c$ is replicated in each processor by performing an all-to-all-broadcast operation. This allows each processor to know the topology of the entire cluster graph [S1, unofficial].
9.  **Compute Migration Costs ($T_c$):** The cluster-migration-cost function $T_c$ is computed. The migration cost of a cell-cluster is defined as the ratio of the size of its associated `ClusterData` structure to the size of an individual ray-segment information. Since this cost is view-independent, it is computed once [S1, unofficial].
10. **Broadcast Costs:** An all-to-all broadcast operation is performed to let each processor become aware of the migration costs of all clusters [S1, unofficial].

#### Phase 2: Remapping (View-Dependent)
*Note: The specific algorithmic steps for the Remapping Phase (Graph Update, Graph Partitioning, Cluster Migration) are described in the [Remapping Phase](4250-remapping-phase.md) article. This article defines the inputs and structures used by that phase.*

### Pseudocode
The following pseudocode implements the Local Clustering Phase. It assumes access to MeTiS for partitioning and standard MPI for communication.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def local_clustering_phase(cells, vertices, num_processors, processor_id):
    """
    Performs the view-independent local clustering phase.
    
    Args:
        cells: List of tetrahedral cells for this processor.
        vertices: List of vertices for this processor.
        num_processors: Total number of processors A.
        processor_id: ID of the current processor k.
        
    Returns:
        local_clusters: List of ClusterData objects for local clusters.
        global_cluster_map: ClusterMapArray entry for local clusters.
        contracted_graph_local: Local portion of Qc.
        migration_costs: Local migration costs Tc.
    """
    
    # Step 1: Construct Local Graph
    # Build adjacency list based on shared faces among local cells
    local_graph = build_graph_from_cells(cells)
    
    # Step 2: Determine local number of clusters q_k
    # q_k is proportional to total node weight (e.g., cell count or volume)
    total_node_weight = compute_total_node_weight(local_graph)
    # Formula: sum(q_k) = c * a. 
    # Implementation detail: c and a are global constants. 
    # q_k = round((total_node_weight / global_total_weight) * total_clusters)
    q_k = compute_local_cluster_count(total_node_weight)
    
    # Step 3: Perform k-way clustering using MeTiS
    # MeTiS returns a partition array mapping each node (cell) to a cluster ID [0, q_k-1]
    cluster_ids = metis_partition(local_graph, q_k)
    
    # Step 4: Group cells into clusters and build local structures
    local_clusters = []
    for cluster_id in range(q_k):
        cluster_cells = [cells[i] for i, cid in enumerate(cluster_ids) if cid == cluster_id]
        cluster_vertices = get_vertices_for_cells(cluster_cells)
        
        # Build CellArray with updated connectivity
        # Replace global neighbor-cell IDs with (global_cluster_id, local_index)
        cell_array = build_cell_array(cluster_cells, cluster_ids, global_cluster_mapping)
        
        # Build VertexArray
        vertex_array = build_vertex_array(cluster_vertices)
        
        # Compute TotalVolume (in WSC)
        total_volume = sum(cell.volume() for cell in cluster_cells)
        
        cluster_data = ClusterData(
            vertex_array=vertex_array,
            cell_array=cell_array,
            total_volume=total_volume
        )
        local_clusters.append(cluster_data)
        
    # Step 5: Assign Global Cluster Identifiers
    # This requires coordination with other processors to ensure unique global IDs.
    # Assuming a global mapping function M_e is established via MPI Alltoall.
    global_cluster_ids = assign_global_cluster_ids(local_clusters, processor_id, num_processors)
    
    # Step 6: Build ClusterMapArray entries for local clusters
    global_cluster_map = []
    for i, cluster_data in enumerate(local_clusters):
        global_id = global_cluster_ids[i]
        # Mapping M_e is determined during global coordination
        mapping = get_global_mapping(global_id) 
        global_cluster_map.append({
            'global_id': global_id,
            'mapping': mapping,
            'pointer': cluster_data
        })
        
    # Step 7: Construct Local portion of Contracted Graph Qc
    # Nodes are clusters. Edges exist if clusters share a face.
    contracted_graph_local = build_contracted_graph(local_clusters, global_cluster_ids)
    
    # Step 8: Broadcast Global Graph Qc
    # Each processor sends its local portion of Qc to all others
    global_contracted_graph = mpi_alltoall_broadcast(contracted_graph_local)
    
    # Step 9: Compute Migration Costs Tc
    # Cost = Size(ClusterData) / Size(RaySegment)
    ray_segment_size = get_ray_segment_size()
    migration_costs = []
    for cluster_data in local_clusters:
        cluster_data_size = get_structure_size(cluster_data)
        cost = cluster_data_size / ray_segment_size
        migration_costs.append(cost)
        
    # Step 10: Broadcast Migration Costs
    global_migration_costs = mpi_alltoall_broadcast(migration_costs)
    
    return local_clusters, global_cluster_map, global_contracted_graph, global_migration_costs
```

### Edge Cases and Pitfalls
*   **Precision:** Volume calculations must use sufficient precision to avoid errors in `TotalVolume`, which influences load balancing in the remapping phase [S1, unofficial].
*   **Boundary Artifacts:** While this method handles connectivity, the underlying chunk-based partitioning may still require overlap handling if the clusters are mapped to spatial chunks. The replacement of neighbor-cell identifiers with cluster identifiers must correctly handle cells on the boundary of a cluster to ensure correct compositing across cluster boundaries [S1, unofficial].
*   **Degenerate Input:** If a processor has no cells, $q_k$ should be 0, and it should not participate in the local graph construction or broadcasting of graph structures, though it must still receive the global broadcast [S1, unofficial].

## Subtopics

### [Remapping Phase](4250-remapping-phase.md)
The view-dependent preprocessing step that adapts the static clustering to the current view. It uses the structures built here to perform graph updates, partitioning, and cluster migration. Choose this when the camera or transfer function changes frequently, requiring dynamic load balancing without re-partitioning the geometry.

## Operating characteristics
*   **Time Complexity:** The local clustering phase is dominated by the MeTiS partitioning step, which is generally sub-linear or linear with respect to the number of cells per processor. The all-to-all broadcasts for the graph and migration costs are $O(A)$ communication complexity, where $A$ is the number of processors. The remapping phase (not detailed here) adds view-dependent overhead but is optimized by the precomputed topology [S1, unofficial].
*   **Memory Complexity:** Each processor stores the `ClusterData` for its local clusters and a copy of the global `Contracted Visualization Graph` $Q_c$. The graph size is proportional to the number of clusters and their interconnections. The `ClusterMapArray` is replicated on all processors, adding $O(K)$ memory per processor, where $K$ is the total number of clusters [S1, unofficial].
*   **Strengths:** Minimizes view-independent preprocessing overhead by decoupling geometry partitioning from view-dependent remapping. Reduces load imbalance variations by proportionally distributing clusters based on node weight.
*   **Limitations:** Requires significant communication (all-to-all broadcasts) during the setup phase. The static nature of the clustering may lead to suboptimal load distribution if the view-dependent sampling weights vary drastically across clusters [S1, unofficial].
*   **When to use:** Unstructured grids with irregular cell sizes where uniform partitioning leads to poor load balance. When the rendering pipeline supports a two-phase approach (static clustering + dynamic remapping).

## Background
This method originates from research on parallel direct volume rendering of unstructured grids. It was developed to address the limitations of uniform partitioning and pure object-space decomposition by introducing a graph-based clustering approach that leverages the MeTiS library for efficient partitioning [S1, unofficial]. The approach separates the view-independent geometric clustering from the view-dependent remapping, allowing for efficient adaptation to changing viewing parameters [S1, unofficial].

## Sources
[S1] Parallel direct volume rendering of unstructured grids based on object-space decomposition, http://hdl.handle.net/11693/17897, fetched 2026-10-26 (unofficial)
