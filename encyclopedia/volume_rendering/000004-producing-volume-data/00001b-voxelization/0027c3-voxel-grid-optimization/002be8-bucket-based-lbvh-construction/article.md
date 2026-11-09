# Bucket-based LBVH construction

Also called: BVH Building, Bvh constructor, Fernandes and Walter LBVH construction

## Overview
Bucket-based LBVH construction builds a Bounding Volume Hierarchy (BVH) from a voxel model by grouping voxels into dynamic buckets rather than using a uniform grid with a fixed brick size [S2, unofficial]. This approach is designed to handle sparse volumes more efficiently than fixed-brick methods [S2, unofficial]. It sets itself apart from siblings like [Bounding Volume Optimization](3583-bounding-volume-optimization.md) by utilizing compute shaders for neighbor finding, cluster merging, and prefix scans to construct the hierarchy [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A voxel model (discrete voxel grid).
*   **Output:** A constructed BVH structure.

**Data Structures:**
*   `volatileGPUData`: A temporary storage structure used during the construction process, likely containing voxel positions and cluster information.
*   **BVH Nodes:** Hierarchical bounding volumes enclosing clusters of voxels.

**Parameters:**
*   `BVH_BUILDING_RADIUS`: A static constant set to 100 [S1, unofficial]. This likely defines the search radius for neighbor finding or cluster merging.

**Steps:**
1.  **Aggregate Data:** Call `aggregateSSBOData(volatileGPUData)` to prepare voxel data for GPU processing [S1, unofficial].
2.  **Neighbor Finding:** Execute the `FIND_BEST_NEIGHBOR` compute shader to identify adjacent voxels or clusters [S1, unofficial].
3.  **Cluster Merging:** Execute the `CLUSTER_MERGING` compute shader to combine neighboring voxels into larger clusters [S1, unofficial].
4.  **Reallocation:** Execute the `REALLOCATE_CLUSTERS` compute shader to adjust cluster boundaries or memory layout [S1, unofficial].
5.  **Prefix Scan (Up-sweep):** Execute the `REDUCE_PREFIX_SCAN` compute shader to perform a parallel reduction [S1, unofficial].
6.  **Prefix Scan (Down-sweep):** Execute the `DOWN_SWEEP_PREFIX_SCAN` compute shader to complete the prefix sum operation [S1, unofficial].
7.  **Reset Position:** Execute the `RESET_LAST_POSITION_PREFIX_SCAN` compute shader to finalize position tracking [S1, unofficial].
8.  **End Loop:** Execute the `END_LOOP_COMPUTATIONS` compute shader to finalize the construction step [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_bucket_lbvhe(voxel_model):
    # Initialize volatile GPU data structure
    volatileGPUData = aggregateSSBOData(voxel_model)
    
    # Define compute shader programs
    shaders = {
        "FIND_BEST_NEIGHBOR": ...,
        "CLUSTER_MERGING": ...,
        "REALLOCATE_CLUSTERS": ...,
        "END_LOOP_COMPUTATIONS": ...,
        "REDUCE_PREFIX_SCAN": ...,
        "DOWN_SWEEP_PREFIX_SCAN": ...,
        "RESET_LAST_POSITION_PREFIX_SCAN": ...
    }
    
    # Step 1: Find best neighbors for each voxel/cluster
    # Uses BVH_BUILDING_RADIUS (100) for search scope
    shaders["FIND_BEST_NEIGHBOR"].dispatch(volatileGPUData)
    
    # Step 2: Merge clusters based on neighbor relationships
    shaders["CLUSTER_MERGING"].dispatch(volatileGPUData)
    
    # Step 3: Reallocate clusters in memory
    shaders["REALLOCATE_CLUSTERS"].dispatch(volatileGPUData)
    
    # Step 4: Perform parallel reduction (prefix scan up-sweep)
    shaders["REDUCE_PREFIX_SCAN"].dispatch(volatileGPUData)
    
    # Step 5: Perform parallel prefix scan (down-sweep)
    shaders["DOWN_SWEEP_PREFIX_SCAN"].dispatch(volatileGPUData)
    
    # Step 6: Reset last position tracking
    shaders["RESET_LAST_POSITION_PREFIX_SCAN"].dispatch(volatileGPUData)
    
    # Step 7: Finalize loop computations
    shaders["END_LOOP_COMPUTATIONS"].dispatch(volatileGPUData)
    
    return volatileGPUData.bvh_structure
```

**Edge Cases and Pitfalls:**
*   **Precision:** GPU-based parallel reductions require careful handling of atomic operations and memory synchronization to avoid race conditions [S1, unofficial].
*   **Boundaries:** The `BVH_BUILDING_RADIUS` of 100 must be sufficient to capture relevant neighbors but not so large as to cause excessive memory usage or computation [S1, unofficial].
*   **Sparse Volumes:** The bucket-based approach is specifically beneficial for sparse volumes where fixed-brick methods waste memory on empty regions [S2, unofficial].

## Subtopics

### [Bucket Size Computation](8964-bucket-size-computation.md)
Computes the bucket texture storing the position of the last voxel of each bucket. It splits computation into fragment and vertex shader steps to avoid inefficient vertex shader texture fetches.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Efficient for sparse volumes by avoiding fixed brick sizes [S2, unofficial]. Parallelizable on GPU using compute shaders [S1, unofficial].
*   **Limitations:** Requires complex compute shader implementation for neighbor finding and prefix scans [S1, unofficial].
*   **When to Use:** Sparse voxel volumes where memory efficiency and traversal speed are critical [S2, unofficial].
*   **When Not to Use:** Dense volumes where fixed-brick methods might be simpler and equally effective [S2, unofficial].

## Background
The method is attributed to Fernandes and Walter [S2, unofficial]. It was developed as an alternative to Zellmann et al.'s fixed-brick LBVH construction [S2, unofficial]. The implementation described in [S1, unofficial] is part of a procedural soft mesher tool.

## Sources
[S1] AlfonsoLRz/VoxelFragmentML, https://github.com/AlfonsoLRz/VoxelFragmentML, fetched 2026-10-26 (unofficial)
[S2] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
