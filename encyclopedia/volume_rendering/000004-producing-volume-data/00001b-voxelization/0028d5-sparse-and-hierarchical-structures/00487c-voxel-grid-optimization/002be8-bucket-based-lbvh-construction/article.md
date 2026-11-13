# Bucket-based LBVH construction

Also called: BVH Building, Bvh constructor, Fernandes and Walter LBVH construction

## Overview
Bucket-based LBVH construction builds a Bounding Volume Hierarchy (BVH) from a voxelized model by grouping voxels into dynamic buckets rather than a uniform grid with fixed brick sizes [S2, unofficial]. This approach addresses the inefficiency of fixed-brick methods in sparse volumes by adapting the spatial partitioning to the actual distribution of active voxels [S2, unofficial]. It sets itself apart from Zellmann et al.'s fixed-brick LBVH [S2, unofficial] by using a bucket texture to store boundary indices, enabling efficient neighbor finding and cluster merging on the GPU [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of active voxels (often derived from a signed distance field or occupancy grid) and a static constant `BVH_BUILDING_RADIUS` [S1, unofficial].
*   **Output:** A BVH data structure ready for traversal, constructed via parallel compute shaders [S1, unofficial].

**Data Structures:**
*   **Bucket Texture:** Stores the index of the last voxel of each bucket (the bucket boundary voxel) [S1, unofficial]. This allows efficient identification of bucket boundaries without costly texture fetches in vertex shaders [S1, unofficial].
*   **Volatile GPU Data:** Intermediate storage for neighbor relationships and cluster states during construction [S1, unofficial].

**Parameters:**
*   `BVH_BUILDING_RADIUS`: A static constant controlling the search radius for neighbors. The source sets this to 100 [S1, unofficial].

**Steps:**
1.  **Aggregate Data:** Call `aggregateSSBOData(volatileGPUData)` to prepare voxel data for processing [S1, unofficial].
2.  **Bucket Assignment:** Group voxels into buckets based on their spatial proximity, replacing the fixed brick size used in uniform grids [S2, unofficial].
3.  **Neighbor Finding:** Execute the `FIND_BEST_NEIGHBOR` compute shader to identify adjacent voxels or clusters within the `BVH_BUILDING_RADIUS` [S1, unofficial].
4.  **Cluster Merging:** Execute the `CLUSTER_MERGING` compute shader to merge neighboring voxels into larger clusters based on the neighbor data [S1, unofficial].
5.  **Reallocation:** Execute the `REALLOCATE_CLUSTERS` compute shader to reorganize clusters for efficient processing [S1, unofficial].
6.  **Prefix Scan for Boundaries:**
    *   Execute `REDUCE_PREFIX_SCAN` to compute aggregate counts [S1, unofficial].
    *   Execute `DOWN_SWEEP_PREFIX_SCAN` to finalize prefix sums [S1, unofficial].
    *   Execute `RESET_LAST_POSITION_PREFIX_SCAN` to determine final bucket boundaries [S1, unofficial].
    *   *Note:* The bucket texture is constructed by splitting this computation: a fragment shader step identifies boundaries, and a vertex shader step constructs the texture, avoiding inefficient vertex shader texture fetches [S1, unofficial].
7.  **Finalization:** Execute `END_LOOP_COMPUTATIONS` to finalize the BVH structure [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_bucket_lbvhi(voxels, bvh_building_radius=100):
    # Step 1: Aggregate data
    volatile_gpu_data = aggregateSSBOData(voxels)
    
    # Step 2: Group voxels into buckets
    # Note: Specific bucketing logic (e.g., hashing) is not detailed in sources [S2, unofficial]
    buckets = group_voxels_into_buckets(voxels)
    
    # Step 3: Find neighbors within radius
    neighbors = find_best_neighbor(buckets, bvh_building_radius)
    
    # Step 4: Merge clusters
    clusters = cluster_merging(buckets, neighbors)
    
    # Step 5: Reallocate clusters
    clusters = reallocate_clusters(clusters)
    
    # Step 6: Compute prefix scans for bucket boundaries
    # The bucket texture stores the index of the last voxel of each bucket
    reduced = reduce_prefix_scan(clusters)
    final_prefix = down_sweep_prefix_scan(reduced)
    last_positions = reset_last_position_prefix_scan(final_prefix)
    
    # Construct bucket texture via fragment and vertex shader steps
    bucket_texture = construct_bucket_texture(last_positions)
    
    # Step 7: Finalize
    end_loop_computations(clusters, bucket_texture)
    
    return clusters, bucket_texture
```

**Edge Cases and Pitfalls:**
*   **Precision:** The use of prefix scans requires careful handling of integer overflow for large voxel counts [G].
*   **Boundaries:** The bucket texture construction relies on correct boundary identification; errors here lead to incorrect BVH structure [S1, unofficial].
*   **Radius Sensitivity:** The `BVH_BUILDING_RADIUS` parameter affects the granularity of neighbor finding; too small a radius may result in fragmented clusters, while too large a radius increases computational cost [S1, unofficial].

## Subtopics

### [Bucket Size Computation](8964-bucket-size-computation.md)
Computes the bucket texture storing the last voxel index of each bucket. It sets itself apart by splitting boundary identification into fragment and vertex shader steps to avoid inefficient texture fetches. Choose it when efficient GPU-based bucket boundary detection is required.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Allows for two to three times higher frame rates than using no empty-space skipping during ray marching [S2, unofficial].
*   **Scaling:** Not given in the sources.
*   **Strengths:** Adapts to sparse volumes better than fixed-brick methods; efficient GPU implementation using compute shaders [S2, unofficial].
*   **Limitations:** Relies on specific compute shader pipelines; parameter sensitivity to `BVH_BUILDING_RADIUS` [S1, unofficial].
*   **When to use:** Sparse volume rendering where fixed-brick LBVH is inefficient [S2, unofficial].
*   **When not to use:** Dense volumes where fixed-brick methods may be more efficient [S2, unofficial].

## Background
The method is attributed to Fernandes and Walter [FW20] and is implemented in the VoxelFragmentML repository [S1, unofficial]. It is discussed in the context of scalable web-based volume rendering of large biomedicinal datasets [S2, unofficial]. The approach was developed to improve upon Zellmann et al.'s fixed-brick LBVH construction [S2, unofficial].

## Sources
[S1] AlfonsoLRz/VoxelFragmentML, https://github.com/AlfonsoLRz/VoxelFragmentML, fetched 2026-10-26 (unofficial)
[S2] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
