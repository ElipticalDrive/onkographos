# Remapping Phase

Also called: Graph Partitioning, Graph Update

## Overview
The Remapping Phase is a view-dependent preprocessing step in chunk-based spatial partitioning for parallel volume rendering, designed to reassign data clusters to processors based on the current viewing parameters [S1, unofficial]. It solves the problem of load imbalance and communication overhead that arises when the camera or transfer function changes, ensuring that clusters with high interaction (pseudo-boundary faces) remain co-located on the same processor [S1, unofficial]. It is distinguished from its parent method, Object-space decomposition with local clustering, by introducing a dynamic graph-partitioning loop (Graph Update, Graph Partitioning, Cluster Migration) that adapts the static clustering solution to the current view [S1, unofficial].

## How it works
The phase consists of three sequential steps: Graph Update (GU), Graph Partitioning (GP), and Cluster Migration (CM) [S1, unofficial]. The GU and GP steps constitute the view-dependent preprocessing overhead [S1, unofficial].

### Inputs and Outputs
*   **Inputs:**
    *   Current viewing parameter $v$ (camera position, orientation, image size) [S1, unofficial].
    *   Previous remapping graph $Q_c$ (or the view-independent graph for the first instance) [S1, unofficial].
    *   Current mapping function $M_e$ (cluster-to-processor assignment) [S1, unofficial].
    *   Cluster data structures (`ClusterData`) containing precomputed view-independent volumes [S1, unofficial].
    *   Number of processors $K$ [S1, unofficial].
*   **Outputs:**
    *   New mapping function $M_e$ (new cluster-to-processor assignment) [S1, unofficial].
    *   Migrated `ClusterData` structures [S1, unofficial].

### Data Structures
*   **Remapping Graph $Q_c$:** A weighted graph where nodes $n_i$ represent clusters $C_i$ and edges $e_{ij}$ represent interactions between neighbor clusters [S1, unofficial].
*   **ClusterMapArray:** An array maintained by each processor that maps cluster IDs to their current home processors [S1, unofficial].
*   **ClusterData:** A structure storing metadata for each cluster, including its volume in the View-Space Coordinate (WSC) system [S1, unofficial].

### Definitions
*   **Pseudo-boundary face:** A face of a cell in cluster $C_i$ that is shared by a cell in a different cluster $C_j$ [S1, unofficial].
*   **Boundary face:** A pseudo-boundary face that becomes a boundary face after remapping if the two clusters it separates are mapped to different processors [S1, unofficial].
*   **bf-face:** Boundary face (relative to the current mapping) [S1, unofficial].
*   **ff-face:** Face relative to the local subvolume (all local clusters treated as one) [S1, unofficial].

### Node Weight Estimation
The weight $w(n_i)$ of a node $n_i$ representing cluster $C_i$ is the estimated rendering cost for viewing parameter $v$ [S1, unofficial]:

$$w(n_i) = I_{\mathcal{C}}^v t_I + S_{\mathcal{C}}^v t_S + R_{\mathcal{C}}^v t_R \quad (6.1)$$

Where:
*   $t_I, t_S, t_R$ are unit costs for ray-face intersections, samples, and ray-segments, determined empirically [S1, unofficial].
*   $I_{\mathcal{C}}^v$: Number of ray-face intersections.
*   $S_{\mathcal{C}}^v$: Number of samples.
*   $R_{\mathcal{C}}^v$: Number of ray-segments.

**1. Estimating $I_{\mathcal{C}}^v$ (Ray-Face Intersections):**
Approximated by summing the projection areas $a_f$ of the bf-faces of cells in $C_i$ [S1, unofficial]. The projection area $a_f$ of a triangular face $f$ in Normalized Device Coordinates (NPC) is:

$$a_f = x_1(y_2 - y_3) + x_2(y_3 - y_1) + x_3(y_1 - y_2) \quad (6.2)$$

Where $(x_k, y_k)$ are the NPC coordinates of the $k$-th vertex of the face [S1, unofficial].
*   **Optimization:** The implementation exploits that a bf-internal face of a cell is an ff-internal face of a neighbor cell. This allows computing the projection area of each internal face only once, avoiding expensive face orientation checks [S1, unofficial].
*   **Error:** Discretization errors from screen pixelation cancel out due to the summation of contiguous floating-point area values [S1, unofficial].

**2. Estimating $S_{\mathcal{C}}^v$ (Samples):**
Estimated by calculating the volume of the cluster in NPC [S1, unofficial].
*   Volume in NPC is the number of unit cubes.
*   Samples per unit cube = $1/A_c$, where $A_c$ is the distance between successive sampling points along a ray [S1, unofficial].
*   **Efficient Calculation:** Instead of summing volumes in NPC for each view, the method uses a constant scaling factor $k$ between the view-independent WSC system and the view-dependent NPC system [S1, unofficial]. The volume of the cluster in WSC is stored in `ClusterData`. Thus:
    $$S_{\mathcal{C}}^v \propto \text{Volume}_{WSC} \times \frac{k}{A_c}$$
    The estimation error is confined to the boundary surface of the cluster [S1, unofficial].

**3. Estimating $R_{\mathcal{C}}^v$ (Ray-Segments):**
Approximated by the sum of projection areas of ff-external and ff-pseudo-boundary faces using Eq. 6.2 [S1, unofficial].
*   **Note:** Ray-segment generation costs are **not** included in the node weighting $w(n_i)$ because the local rendering scheme treats all local clusters as a single subvolume, generating ray-segments only for the ff-boundary faces of that subvolume [S1, unofficial]. Minimizing the cutsize in partitioning implicitly minimizes ray-segment generation [S1, unofficial].

### Edge Weight Estimation
The weight $w(n_i, n_j)$ of edge $e_{ij}$ represents the interaction between clusters $C_i$ and $C_j$ [S1, unofficial].
*   Computed by summing the pixel-coverage counts of pseudo-boundary faces shared by neighbor cells in $C_i$ and $C_j$ [S1, unofficial].
*   Pixel-coverage counts are approximated by the projection areas of these faces using Eq. 6.2 [S1, unofficial].
*   If $C_i$ and $C_j$ are mapped to different processors, this weight incurs redundant computation (ray-segment generation) and communication overhead (pixel merging) [S1, unofficial].

### Step 1: Graph Update (GU)
1.  **Local Graph Construction:** Each processor concurrently computes the local portion of the weighted visualization graph by recomputing node and edge weights for the new viewing parameter $v$ [S1, unofficial]. For the first instance, the view-independent graph is used [S1, unofficial].
2.  **Local Augmentation:** Processors augment the local graph using the cluster-migration cost function $T_c$ and the current mapping $M_e$ [S1, unofficial].
3.  **All-to-All Broadcast:** An all-to-all broadcast operation is performed on the local portions of the remapping graph so that every processor gathers a complete copy of the global remapping graph $Q_c$ [S1, unofficial].

### Step 2: Graph Partitioning (GP)
1.  **Concurrent Partitioning:** All processors concurrently execute RM-MeTiS for $K$-way partitioning of their copies of $Q_c$, using different random seeds [S1, unofficial].
2.  **Global Minimum:** A global-minimum operation is performed on the cutsize values of the local partition solutions to determine the best partition [S1, unofficial].
3.  **Broadcast Solution:** The processor producing the best solution broadcasts its partition vector, which corresponds to the new mapping function $M_e$ [S1, unofficial]. This avoids the worst-case behavior of the randomized RM-MeTiS algorithm [S1, unofficial].

### Step 3: Cluster Migration (CM)
1.  **Migration:** Clusters whose new mappings differ from current mappings migrate to their new home processors via personalized communication of their `ClusterData` structures [S1, unofficial].
2.  **Update Map:** Each processor updates its `ClusterMapArray` structure according to the new mapping [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def remapping_phase(viewing_param v, previous_graph Q_c, current_mapping M_e, K, ClusterData):
    """
    Performs the Remapping Phase to reassign clusters to processors.
    """
    # Step 1: Graph Update (GU)
    
    # 1.1 Compute local graph portions
    local_graph = compute_local_graph(Q_c, v, M_e, ClusterData)
    
    # 1.2 Augment with migration costs
    # T_c is the cluster-migration cost function
    augmented_local_graph = augment_with_migration_costs(local_graph, M_e, T_c)
    
    # 1.3 All-to-All Broadcast to gather global graph
    global_Q_c = all_to_all_broadcast(augmented_local_graph)
    
    # Step 2: Graph Partitioning (GP)
    
    # 2.1 Concurrent RM-MeTiS with different seeds
    my_seed = get_unique_seed()
    local_partition, local_cutsize = rm_metis(global_Q_c, K, my_seed)
    
    # 2.2 Global Minimum to find best partition
    best_cutsize, best_processor_id = global_minimum(local_cutsize)
    
    # 2.3 Broadcast new mapping from best processor
    if my_processor_id == best_processor_id:
        new_mapping = local_partition
    else:
        new_mapping = receive_broadcast_from(best_processor_id)
        
    # Step 3: Cluster Migration (CM)
    
    # 3.1 Migrate clusters
    for cluster_id, new_processor in new_mapping.items():
        if new_processor != current_mapping[cluster_id]:
            migrate_cluster(cluster_id, new_processor, ClusterData[cluster_id])
            
    # 3.2 Update local map
    current_mapping = new_mapping
    update_cluster_map_array(current_mapping)
    
    return new_mapping

def compute_projection_area(vertices):
    """
    Computes projection area of a triangular face in NPC.
    vertices: list of 3 tuples (x, y)
    """
    x1, y1 = vertices[0]
    x2, y2 = vertices[1]
    x3, y3 = vertices[2]
    return x1 * (y2 - y3) + x2 * (y3 - y1) + x3 * (y1 - y2)

def estimate_node_weight(cluster_id, viewing_param v, ClusterData):
    """
    Estimates w(n_i) for cluster C_i.
    """
    # 1. Estimate I_C^v (Ray-Face Intersections)
    # Sum projection areas of bf-faces
    # Optimization: compute internal face areas only once
    bf_faces = get_boundary_faces(cluster_id, current_mapping)
    I_C_v = sum(compute_projection_area(face.vertices) for face in bf_faces)
    
    # 2. Estimate S_C^v (Samples)
    # Volume in WSC * (k / A_c)
    volume_wsc = ClusterData[cluster_id].volume_wsc
    A_c = get_sampling_distance(viewing_param)
    k = get_scaling_factor()
    S_C_v = volume_wsc * (k / A_c)
    
    # 3. Estimate R_C^v (Ray-Segments)
    # Not used in node weight, but computed for edge weight logic if needed
    # R_C_v = sum(projection_areas(ff_external_faces) + projection_areas(ff_pseudo_boundary_faces))
    
    # Unit costs t_I, t_S, t_R are pre-determined
    return I_C_v * t_I + S_C_v * t_S + 0 * t_R # R_C_v cost excluded from node weight

def estimate_edge_weight(cluster_i, cluster_j, viewing_param v):
    """
    Estimates w(n_i, n_j) based on pseudo-boundary faces.
    """
    shared_faces = get_shared_faces(cluster_i, cluster_j)
    weight = sum(compute_projection_area(face.vertices) for face in shared_faces)
    return weight
```

### Edge Cases and Pitfalls
*   **Transfer Function Changes:** Node and edge weights do not depend on the transfer function (mapping scalar values to color/opacity). If only the transfer function changes, the remapping phase should be skipped [S1, unofficial].
*   **Image Size Changes:** While node and edge weights change with image size, their relative weights remain the same. If only image size changes, the remapping phase can be skipped [S1, unofficial].
*   **First Visualization:** For the first instance, the graph is effectively the view-independent contracted visualization graph $Q_c$ [S1, unofficial].

## Subtopics

### Graph Update
The first step of the Remapping Phase where the remapping graph $Q_c$ is updated in parallel for the new viewing parameter. It involves computing local graph portions, augmenting them with migration costs, and performing an all-to-all broadcast to construct the global graph [S1, unofficial].

### Graph Partitioning
The second step where all processors concurrently execute RM-MeTiS for $K$-way partitioning using different random seeds. The best partition is selected via a global-minimum operation on cutsize values, and the solution is broadcast to all processors to determine the new mapping [S1, unofficial].

### Cluster Migration
The third step where clusters with changed mappings migrate to new home processors via personalized communication of `ClusterData` structures. Each processor updates its `ClusterMapArray` to reflect the new mapping [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Dominated by the all-to-all broadcast of the graph and the RM-MeTiS partitioning. RM-MeTiS is a randomized algorithm, so performance may vary slightly between runs due to different seeds [S1, unofficial].
*   **Memory Complexity:** Requires each processor to store a copy of the global remapping graph $Q_c$ during the GP step [S1, unofficial].
*   **Strengths:**
    *   Reduces communication overhead by keeping interacting clusters (pseudo-boundary faces) on the same processor [S1, unofficial].
    *   Adapts to view-dependent load changes (camera movement) [S1, unofficial].
    *   Efficient sample estimation using precomputed WSC volumes [S1, unofficial].
*   **Limitations:**
    *   View-dependent preprocessing overhead (GU and GP steps) [S1, unofficial].
    *   RM-MeTiS is randomized; worst-case behavior is mitigated by global minimum selection but adds communication overhead [S1, unofficial].
*   **When to use:** When the viewing parameter (camera position/orientation) changes significantly, affecting the projected areas and thus the load distribution [S1, unofficial].
*   **When not to use:** When only the transfer function changes, or when only the image size changes (relative weights remain constant) [S1, unofficial].

## Background
The Remapping Phase is part of a parallel direct volume rendering system for unstructured grids based on object-space decomposition [S1, unofficial]. It was developed to address the limitations of static clustering in dynamic visualization scenarios where the view-dependent load distribution changes [S1, unofficial]. The method uses RM-MeTiS, a variant of the MeTiS graph partitioning library, adapted for the specific cost model of volume rendering (ray-face intersections, samples, and ray-segments) [S1, unofficial].

## Sources
[S1] Parallel direct volume rendering of unstructured grids based on object-space decomposition, http://hdl.handle.net/11693/17897, fetched 2026-10-26 (unofficial)
