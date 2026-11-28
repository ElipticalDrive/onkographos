# Cluster Migration


## Overview
Cluster Migration (CM) is the data movement phase in distributed volume rendering that transfers cluster data structures to their newly assigned processors after the remapping phase [S2, unofficial]. It solves the problem of synchronizing the distributed spatial partition with the updated processor assignments, ensuring that each processor holds the geometric data required for its assigned screen regions [S2, unofficial]. Unlike the remapping phase which computes the assignment, CM physically relocates the `ClusterData` via personalized communication [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** The new cluster-to-processor mapping function $M$ (derived from the remapping phase), the current `ClusterMapArray` on each processor, and the `ClusterData` structures currently resident on each processor [S1, S2, unofficial].
*   **Output:** Updated `ClusterMapArray` on each processor and the `ClusterData` structures distributed such that each processor holds the clusters assigned to it [S1, S2, unofficial].

**Data Structures:**
*   `ClusterData`: The data structure representing a cluster's geometric and volumetric information. The exact internal layout is not specified in the sources, but it is the unit of communication [S1, S2, unofficial].
*   `ClusterMapArray`: An array or structure on each processor that tracks which clusters are currently mapped to that processor [S1, unofficial].

**Steps:**
1.  **Identify Migrations:** Determine which clusters have a new mapping $M(c)$ that differs from their current processor assignment [S1, unofficial].
2.  **Personalized Communication:** Perform an all-to-all personalized communication among all processors [S2, unofficial]. Each processor sends the `ClusterData` of its migrating clusters to the destination processors specified by $M$ [S1, S2, unofficial].
3.  **Replication:** Replicate the data structures of the migrating clusters on the destination processors [S2, unofficial].
4.  **Update Mapping:** Each processor updates its `ClusterMapArray` structure to reflect the new mapping $M$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Assume:
# - processors: list of all processor IDs
# - my_id: ID of the current processor
# - cluster_data: dict mapping cluster_id -> ClusterData object (local clusters)
# - current_map: dict mapping cluster_id -> current_processor_id
# - new_map: dict mapping cluster_id -> new_processor_id (from remapping phase)

# Step 1: Identify clusters to send and receive
send_list = {} # dest_processor_id -> list of (cluster_id, ClusterData)
recv_list = {} # src_processor_id -> list of cluster_id

for cluster_id, data in cluster_data.items():
    dest = new_map[cluster_id]
    src = current_map[cluster_id]
    
    # If the cluster is moving to a different processor
    if dest != my_id:
        if dest not in send_list:
            send_list[dest] = []
        send_list[dest].append((cluster_id, data))
        
    # If the cluster is coming from a different processor
    if src != my_id:
        if src not in recv_list:
            recv_list[src] = []
        recv_list[src].append(cluster_id)

# Step 2: Perform all-to-all personalized communication
# This step involves sending send_list data to respective destinations
# and receiving data from respective sources into recv_list
# Implementation depends on MPI or similar library
# For each dest, send send_list[dest]
# For each src, receive into recv_list[src]

# Step 3: Update local state
# Remove migrated clusters from local storage
for cluster_id in list(cluster_data.keys()):
    if new_map[cluster_id] != my_id:
        del cluster_data[cluster_id]

# Add received clusters to local storage
for src, cluster_ids in recv_list.items():
    for cluster_id in cluster_ids:
        # Data received in Step 2 is assumed to be available
        cluster_data[cluster_id] = received_data[cluster_id]

# Step 4: Update ClusterMapArray
# Update the local mapping array to reflect new_map
update_cluster_map_array(new_map)
```

**Edge Cases and Pitfalls:**
*   **Precision:** Not given in the sources.
*   **Boundaries:** Not given in the sources.
*   **Degenerate Input:** Not given in the sources.

## Subtopics

### Local Rendering Phase
The phase immediately following Cluster Migration where processors render their assigned screen regions using the newly arrived clusters. It leverages the high interaction among local clusters to optimize ray casting. [Local Rendering Phase](4260-local-rendering-phase.md)

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Not given in the sources.
*   **Strengths and Limitations:** The method avoids worst-case behavior of randomized algorithms like RM-Metis by explicitly handling migrations [S1, unofficial]. It ensures that processors hold highly interacting clusters, which benefits the subsequent local rendering phase [S2, unofficial].
*   **When to use it:** Use when the cluster-to-processor mapping has changed and data consistency across distributed nodes is required before rendering.
*   **When not to use it:** Not given in the sources.

## Background
Cluster Migration is part of a hypergraph-partitioning based remapping model for image-space parallel volume rendering [S2, unofficial]. It was described in the context of parallel direct volume rendering of unstructured grids based on object-space decomposition [S1, unofficial]. The method is used in conjunction with graph partitioning tools like PaToIl to calculate cluster-processor mappings [S2, unofficial].

## Sources
[S1] Parallel direct volume rendering of unstructured grids based on object-space decomposition, http://hdl.handle.net/11693/17897, fetched 2026-10-26 (unofficial)
[S2] A hypergraph-partitioning based remapping model for image-space parallel volume rendering, http://hdl.handle.net/11693/18191, fetched 2026-10-26 (unofficial)
