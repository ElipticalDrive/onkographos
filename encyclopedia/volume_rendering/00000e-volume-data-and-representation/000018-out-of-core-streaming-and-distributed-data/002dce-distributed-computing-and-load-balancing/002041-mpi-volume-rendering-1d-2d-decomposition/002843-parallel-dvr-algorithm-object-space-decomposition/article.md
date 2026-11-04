# Parallel DVR Algorithm (Object-Space Decomposition)
Also called: OS-DVR, OS-Parallel DVR Algorithm, View-Dependent Remapping DVR, Proposed OS-parallel DVR algorithm

## Overview
The Parallel DVR Algorithm (Object-Space Decomposition), also known as OS-DVR, is a parallel direct volume rendering method that decomposes the volumetric dataset into clusters to minimize inter-processor communication and redundant ray calculations [S1, unofficial]. Unlike standard domain decomposition methods that fix the data distribution, OS-DVR employs a view-dependent remapping phase to dynamically reassign clusters based on the current viewing parameters, thereby optimizing load balance and communication overhead for each visualization instance [S1, unofficial]. It builds on [MPI Volume Rendering (1D/2D Decomposition)](2249-mpi-volume-rendering-1d-2d-decomposition.md) by introducing a graph-partitioning-based remapping step that adapts the static data distribution to the view.

## How it works
The algorithm consists of four consecutive phases: Clustering, Remapping, Local Rendering, and Pixel Merging [S1, unofficial]. The Clustering phase is a view-independent preprocessing step performed once. The subsequent three phases are executed for each visualization instance [S1, unofficial].

### Inputs and Outputs
*   **Inputs:**
    *   Volumetric dataset $V$ (unstructured grid, e.g., tetrahedral cells).
    *   Number of processors $K$.
    *   Total number of clusters $a$ (empirically determined).
    *   Viewing parameters (camera position, direction, etc.).
*   **Outputs:**
    *   Final rendered image.

### Data Structures
1.  **Visualization Graph ($Q_v$):**
    *   Nodes: Represent cells of $V$.
    *   Edges: Exist between nodes if cells share a face.
    *   Node Weight: Proportional to cell volume (in World Space Coordinates, WSC). This approximates the view-independent load (number of samples) [S1, unofficial].
    *   Edge Weight: Proportional to the area of the shared face (in WSC). This approximates the view-independent interaction (ray intersections between cells) [S1, unofficial].
2.  **Simulation Graph ($Q_s$):**
    *   Represents the initial distribution of $V$ among $K$ processors from the simulation phase. Used to initialize local clustering [S1, unofficial].
3.  **Cluster Data:**
    *   Each cluster $C_i$ has a unique global identifier.
    *   Each processor maintains a `ClusterData` structure for each of its local clusters, storing the associated tetrahedral cell data [S1, unofficial].
4.  **Remapping Graph:**
    *   Constructed by augmenting the contracted visualization graph with the current cluster mapping [S1, unofficial].

### Steps

#### 1. Clustering Phase (View-Independent Preprocessing)
This phase is executed only once before the first visualization instance [S1, unofficial].

1.  **Construct Local Visualization Graphs:**
    *   Each processor $P_k$ ($k=1 \dots K$) constructs its local view-independent visualization graph $Q_{v_k}$, which is the subgraph of the global $Q_v$ induced by the nodes (cells) currently local to $P_k$ [S1, unofficial].
2.  **Local Partitioning:**
    *   Each processor $P_k$ performs $a_k$-way partitioning of $Q_{v_k}$ using MeTiS [S1, unofficial].
    *   The local number of clusters $a_k$ is selected to be proportional to the total node weight in $Q_{v_k}$ such that $\sum_{k=1}^{K} a_k = a$ [S1, unofficial].
3.  **Global Clustering Solution:**
    *   The collection of local partitions constitutes the global clustering solution $C = [C_1, C_2, \dots, C_a]$ [S1, unofficial].
    *   Assign unique global cluster identifiers and determine the initial global mapping function $M$ for these clusters [S1, unofficial].

#### 2. Remapping Phase (View-Dependent)
This phase is executed for each visualization instance [S1, unofficial].

1.  **Weight Estimation:**
    *   Weight nodes and edges of the contracted visualization graph according to the current viewing parameters using efficient node and edge weight estimation schemes [S1, unofficial].
    *   *Note:* The specific formulas for view-dependent weighting are not detailed in the sources, but the goal is to reflect the computational cost and communication volume for the current view [S1, unofficial].
2.  **Construct Remapping Graph:**
    *   Augment the contracted visualization graph to construct a remapping graph using the current cluster mapping $M$ [S1, unofficial].
3.  **Partition Remapping Graph:**
    *   Partition the remapping graph using RM-MeTiS (a modified MeTiS package) to find new mappings for the clusters [S1, unofficial].
    *   The partitioning minimizes cutsize (minimizing inter-processor communication volume and redundant ray segments) while maintaining balance (computational load balance) [S1, unofficial].
4.  **Migration:**
    *   Identify clusters whose new mappings differ from their current mappings.
    *   Migrate these clusters to their new home processors [S1, unofficial].

#### 3. Local Rendering Phase
1.  Each processor $P_k$ concurrently renders its local subvolume (formed by the set of local clusters) using Koyamada's sequential DVR algorithm [S1, unofficial].
2.  Koyamada's algorithm is a standard sequential volume ray casting method [S1, unofficial].

#### 4. Pixel Merging Phase
1.  Each processor produces the final image of a distinct screen subregion [S1, unofficial].
2.  Three screen decomposition and assignment schemes are implemented to minimize load imbalance and/or communication overhead due to ray-segment migration [S1, unofficial].
3.  The specific schemes are not detailed in the sources, but they involve assigning screen regions to processors based on the current cluster distribution to reduce communication [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# OS-DVR Algorithm Implementation

def os_dvr(volumetric_data, K, a, viewing_params):
    """
    volumetric_data: List of cells with geometry and attributes
    K: Number of processors
    a: Total number of clusters
    viewing_params: Camera position, direction, etc.
    """
    
    # Phase 1: Clustering (View-Independent, executed once)
    # Assume data is initially distributed among K processors based on simulation graph Qs
    clusters = []
    global_cluster_id = 0
    
    for processor_id in range(K):
        # Get local cells for this processor
        local_cells = get_local_cells(processor_id, volumetric_data)
        
        # Construct local visualization graph Qv_k
        # Nodes: local_cells, Edges: shared faces
        local_graph = build_visualization_graph(local_cells)
        
        # Calculate total node weight for this processor
        total_node_weight = sum(cell_volume(cell) for cell in local_cells)
        
        # Determine local number of clusters a_k proportional to total node weight
        # sum(a_k) = a
        a_k = proportional_to_weight(total_node_weight, total_node_weight_all_processors, a)
        
        # Perform a_k-way partitioning using MeTiS
        local_partitions = metis_partition(local_graph, a_k)
        
        # Create ClusterData structures for each local partition
        for partition in local_partitions:
            cluster = Cluster(
                id=global_cluster_id,
                cells=partition.cells,
                processor_id=processor_id
            )
            clusters.append(cluster)
            global_cluster_id += 1
            
    # Initial global mapping M
    mapping = {cluster.id: cluster.processor_id for cluster in clusters}
    
    # Phase 2: Remapping (View-Dependent, executed for each instance)
    # Construct contracted visualization graph from clusters
    contracted_graph = build_contracted_graph(clusters)
    
    # Weight nodes and edges based on viewing_params
    weighted_graph = estimate_view_dependent_weights(contracted_graph, viewing_params)
    
    # Construct remapping graph by augmenting with current mapping
    remapping_graph = augment_graph(weighted_graph, mapping)
    
    # Partition remapping graph using RM-MeTiS
    new_mapping = rm_metis_partition(remapping_graph)
    
    # Migrate clusters if mapping changes
    for cluster in clusters:
        if new_mapping[cluster.id] != mapping[cluster.id]:
            migrate_cluster(cluster, new_mapping[cluster.id])
            mapping[cluster.id] = new_mapping[cluster.id]
            
    # Phase 3: Local Rendering
    # Each processor renders its local subvolume using Koyamada's algorithm
    local_images = []
    for processor_id in range(K):
        local_subvolume = get_local_subvolume(processor_id, clusters, mapping)
        image = koyamada_dvr(local_subvolume, viewing_params)
        local_images.append(image)
        
    # Phase 4: Pixel Merging
    # Assign screen subregions to processors
    # Three schemes are available; select one based on optimization criteria
    screen_decomposition = select_screen_decomposition_scheme(clusters, mapping, viewing_params)
    
    final_image = merge_images(local_images, screen_decomposition)
    
    return final_image

def build_visualization_graph(cells):
    """
    Constructs a graph where nodes are cells and edges connect cells sharing a face.
    Node weight = cell volume.
    Edge weight = shared face area.
    """
    graph = Graph()
    for cell in cells:
        node = graph.add_node(cell, weight=cell.volume())
        
    for i, cell1 in enumerate(cells):
        for j, cell2 in enumerate(cells):
            if i < j and cell1.shares_face(cell2):
                area = cell1.shared_face_area(cell2)
                graph.add_edge(cell1.node, cell2.node, weight=area)
                
    return graph

def proportional_to_weight(local_weight, total_weight, total_clusters):
    """
    Calculates a_k proportional to local_weight such that sum(a_k) = total_clusters.
    """
    return round(local_weight / total_weight * total_clusters)

def estimate_view_dependent_weights(contracted_graph, viewing_params):
    """
    Estimates node and edge weights based on viewing parameters.
    Specific formulas not provided in sources.
    """
    # Placeholder for view-dependent weighting logic
    # Node weight might depend on ray density, cell volume, and view direction
    # Edge weight might depend on ray-segment migration probability
    pass

def augment_graph(contracted_graph, mapping):
    """
    Augments the contracted graph with the current cluster mapping to form the remapping graph.
    """
    # Implementation details not provided in sources
    pass

def rm_metis_partition(graph):
    """
    Partitions the graph using RM-MeTiS to minimize cutsize and maintain balance.
    Returns new mapping of cluster IDs to processor IDs.
    """
    # RM-MeTiS is a modified MeTiS package
    # Minimizes cutsize (communication) and maintains balance (load)
    return metis_partition(graph, K, objective='cutsize_and_balance')

def migrate_cluster(cluster, new_processor_id):
    """
    Migrates the cluster data to the new processor.
    """
    # Send cluster data to new_processor_id
    # Update local data structures on new_processor_id
    pass

def koyamada_dvr(subvolume, viewing_params):
    """
    Renders the subvolume using Koyamada's sequential DVR algorithm.
    """
    # Koyamada's algorithm implementation
    # Standard volume ray casting
    pass

def select_screen_decomposition_scheme(clusters, mapping, viewing_params):
    """
    Selects one of three screen decomposition schemes to minimize load imbalance
    and/or communication overhead.
    """
    # Scheme selection logic not provided in sources
    pass

def merge_images(local_images, screen_decomposition):
    """
    Merges local images into the final image based on screen decomposition.
    """
    # Combine image regions
    pass
```

### Parameters
*   **$a$ (Total number of clusters):** Determined empirically [S1, unofficial].
*   **$a_k$ (Local number of clusters):** Proportional to the total node weight in the local graph such that $\sum_{k=1}^{K} a_k = a$ [S1, unofficial].
*   **RM-MeTiS:** A MeTiS-based remapping tool developed by modifying and enhancing the original MeTiS package for partitioning the remapping graph [S1, unofficial].

### Edge Cases and Pitfalls
*   **Global Clustering Overhead:** Constructing the global view-independent visualization graph $Q_v$ for global clustering may be very expensive. The algorithm uses parallel local clustering to minimize this overhead [S1, unofficial].
*   **Cluster Weight Variation:** The local number of clusters $a_k$ is selected to be proportional to the total node weight to reduce variations in cluster weights, which improves the efficiency of graph-partitioning based remapping [S1, unofficial].
*   **Communication Overhead:** The algorithm aims to minimize inter-processor communication volume during pixel merging (due to ray-segment migration) and redundant calculations during local rendering (due to additional ray-segments generated by partitioning) by minimizing the cutsize of the partition [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The clustering phase is executed once, providing a preprocessing cost. The remapping, local rendering, and pixel merging phases are executed for each visualization instance. The complexity of remapping depends on the graph partitioning cost (MeTiS/RM-MeTiS).
*   **Memory Complexity:** Requires storing the visualization graph, simulation graph, cluster data, and remapping graph. The use of local clustering reduces the memory overhead compared to global clustering.
*   **Performance:** The algorithm minimizes cutsize to minimize both inter-processor communication volume during pixel merging and redundant calculations during local rendering [S1, unofficial].
*   **Strengths:** View-dependent remapping allows for dynamic load balancing and communication optimization. Local clustering reduces preprocessing overhead.
*   **Limitations:** Requires implementation of RM-MeTiS and three screen decomposition schemes. The specific view-dependent weighting formulas are not detailed.
*   **When to use:** Suitable for parallel volume rendering on distributed memory systems where the viewing parameters change frequently, and dynamic load balancing is beneficial.
*   **When not to use:** If the viewing parameters are static, the remapping overhead may not be justified. If the preprocessing cost is prohibitive, global clustering might be considered (though it has higher overhead).

## Background
The OS-DVR algorithm was proposed in a thesis titled "Parallel direct volume rendering of unstructured grids based on object-space decomposition" [S1, unofficial]. The work models the OS decomposition problem for OS-parallel DVR as a Graph Partitioning (GP) problem with correct view-dependent node and edge weighting. The proposed model generates a remapping graph by node and edge augmentations to the graph representing the computational structure of the parallel application. A MeTiS-based remapping tool, RM-MeTiS, was developed by modifying and enhancing the original MeTiS package. The algorithm is designed for parallel machines where simulations are performed, leveraging the existing distribution of volumetric data from the simulation phase [S1, unofficial].

## Sources
[S1] Parallel direct volume rendering of unstructured grids based on object-space decomposition, http://hdl.handle.net/11693/17897, fetched 2026-10-26 (unofficial)
