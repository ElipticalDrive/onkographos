# Bucket-based LBVH construction

Also called: BVH Building, Bvh constructor, Fernandes and Walter LBVH construction

## Overview
Bucket-based LBVH construction builds a Bounding Volume Hierarchy (BVH) from a voxel model by grouping voxels into dynamic buckets rather than a fixed uniform grid [S2, unofficial]. This approach addresses the inefficiency of fixed-brick methods on sparse volumes by adapting the spatial partitioning to the actual data distribution [S2, unofficial]. It distinguishes itself from siblings like [Bounding Volume Optimization](3583-bounding-volume-optimization.md) by utilizing compute shaders for neighbor finding and cluster merging, enabling parallel construction on the GPU [S1, unofficial].

## How it works
The method initializes a BVH from a model using a series of compute shader passes [S1, unofficial]. It replaces the fixed brick size used in uniform grids with a bucketing strategy to group voxels [S2, unofficial].

**Inputs and Outputs:**
*   **Input:** A set of active voxels representing the model.
*   **Output:** A BVH structure ready for traversal.

**Data Structures:**
*   `volatileGPUData`: A temporary storage structure holding intermediate voxel states during construction.
*   **BVH Nodes:** Implicitly constructed via cluster merging; specific node layout is not detailed in the sources.

**Parameters:**
*   `BVH_BUILDING_RADIUS`: A static constant set to 100, likely defining the search radius for neighbor identification [S1, unofficial].

**Steps:**
1.  **Aggregate Data:** Call `aggregateSSBOData(volatileGPUData)` to prepare voxel data for processing [S1, unofficial].
2.  **Neighbor Finding:** Execute the `FIND_BEST_NEIGHBOR` compute shader to identify spatial relationships between voxels [S1, unofficial].
3.  **Cluster Merging:** Execute the `CLUSTER_MERGING` compute shader to group voxels into clusters based on neighbor data [S1, unofficial].
4.  **Reallocation:** Execute the `REALLOCATE_CLUSTERS` compute shader to organize the clusters [S1, unofficial].
5.  **Prefix Scan:** Perform a parallel prefix scan using `REDUCE_PREFIX_SCAN`, `DOWN_SWEEP_PREFIX_SCAN`, and `RESET_LAST_POSITION_PREFIX_SCAN` to compute offsets and finalize bucket boundaries [S1, unofficial].
6.  **Loop Termination:** Execute `END_LOOP_COMPUTATIONS` to finalize the construction phase [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_bucket_lbvhi(voxels):
    # Prepare data for GPU processing
    volatileGPUData = aggregateSSBOData(voxels)
    
    # Step 1: Find neighbors within BVH_BUILDING_RADIUS (100)
    # Uses FIND_BEST_NEIGHBOR compute shader
    neighbors = find_best_neighbor(volatileGPUData, radius=100)
    
    # Step 2: Merge voxels into clusters
    # Uses CLUSTER_MERGING compute shader
    clusters = cluster_merging(volatileGPUData, neighbors)
    
    # Step 3: Reallocate clusters for efficient storage
    # Uses REALLOCATE_CLUSTERS compute shader
    reallocated_clusters = reallocate_clusters(clusters)
    
    # Step 4: Compute prefix sums for bucket boundaries
    # Uses REDUCE_PREFIX_SCAN, DOWN_SWEEP_PREFIX_SCAN, RESET_LAST_POSITION_PREFIX_SCAN
    prefix_sums = reduce_prefix_scan(reallocated_clusters)
    down_sweep = down_sweep_prefix_scan(prefix_sums)
    reset_last_pos = reset_last_position_prefix_scan(down_sweep)
    
    # Step 5: Finalize
    # Uses END_LOOP_COMPUTATIONS compute shader
    bvh = end_loop_computations(reallocated_clusters, reset_last_pos)
    
    return bvh
```

**Edge Cases and Pitfalls:**
*   **Precision:** The use of a fixed radius (`BVH_BUILDING_RADIUS = 100`) may lead to excessive neighbor comparisons in dense regions or missed connections in sparse regions if the voxel scale varies significantly [S1, unofficial].
*   **Boundaries:** The sources do not specify how boundary voxels (those at the edge of the volume) are handled during neighbor finding [S1, unofficial].

## Subtopics

### [Bucket Size Computation](8964-bucket-size-computation.md)
Computes the bucket texture storing the index of the last voxel of each bucket. It splits computation into fragment and vertex shader steps to avoid inefficient vertex shader texture fetches.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** The bucketing approach allows for two to three times higher frame rates than using no empty-space skipping during ray marching [S2, unofficial].
*   **Strengths:** Adapts well to sparse volumes by avoiding fixed brick sizes [S2, unofficial].
*   **Limitations:** Relies on complex compute shader pipelines (neighbor finding, prefix scans) which may have high overhead for very small volumes [S1, unofficial].
*   **When to use:** Suitable for sparse volumes on GPU architectures where parallel compute shaders are available [S2, unofficial].
*   **When not to use:** Not suitable for uniform, dense volumes where fixed-brick methods (like Zellmann et al.'s) might be simpler [S2, unofficial].

## Background
The method is attributed to Fernandes and Walter [FW20], as cited in Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze [S2, unofficial]. It is implemented in the VoxelFragmentML repository by AlfonsoLRz [S1, unofficial]. The approach contrasts with Zellmann et al.'s fixed-brick LBVH construction [S2, unofficial].

## Sources
[S1] AlfonsoLRz/VoxelFragmentML, https://github.com/AlfonsoLRz/VoxelFragmentML, fetched 2026-10-26 (unofficial)
[S2] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
