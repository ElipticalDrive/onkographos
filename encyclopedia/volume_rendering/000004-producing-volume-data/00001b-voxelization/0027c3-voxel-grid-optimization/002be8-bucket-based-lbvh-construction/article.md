# Bucket-based LBVH construction

Also called: BVH Building, Bvh constructor, Fernandes and Walter LBVH construction

## Overview
Bucket-based LBVH construction builds a Bounding Volume Hierarchy (BVH) from a voxel model by grouping voxels into dynamic buckets rather than a uniform grid with fixed brick sizes [S2, unofficial]. This approach addresses the inefficiency of uniform grids in sparse volumes by adapting the spatial partitioning to the actual data distribution [S2, unofficial]. It sets itself apart from fixed-brick methods by using compute shaders for neighbor finding, cluster merging, and prefix scans to dynamically determine bucket boundaries [S1, unofficial].

## How it works
The method initializes a BVH from a model using a series of compute shader stages. The core logic involves aggregating data, finding neighbors, merging clusters, and resolving bucket boundaries via prefix scans.

**Inputs and Outputs:**
- Input: A voxel model (implicit or explicit grid).
- Output: A BVH structure representing the spatial hierarchy of the voxels.

**Data Structures:**
- `volatileGPUData`: A temporary storage structure holding intermediate voxel states, neighbor indices, and cluster identifiers. The exact layout is not specified in the sources, but it supports the compute shader operations [S1, unofficial].
- `BVH_BUILDING_RADIUS`: A static constant used to define the search radius for neighbor finding. The typical value is 100 [S1, unofficial].

**Steps:**
1. **Aggregate Data:** Call `aggregateSSBOData(volatileGPUData)` to prepare the voxel data for processing [S1, unofficial].
2. **Find Best Neighbor:** Execute the `FIND_BEST_NEIGHBOR` compute shader. This step identifies the most suitable neighbor for each voxel within the `BVH_BUILDING_RADIUS` [S1, unofficial].
3. **Cluster Merging:** Execute the `CLUSTER_MERGING` compute shader. Voxels are grouped into clusters based on the neighbor relationships found in the previous step [S1, unofficial].
4. **Reallocate Clusters:** Execute the `REALLOCATE_CLUSTERS` compute shader. This step adjusts cluster assignments to optimize the spatial coherence of the buckets [S1, unofficial].
5. **End Loop Computations:** Execute the `END_LOOP_COMPUTATIONS` compute shader to finalize the clustering phase [S1, unofficial].
6. **Prefix Scan (Reduce):** Execute the `REDUCE_PREFIX_SCAN` compute shader. This performs a parallel reduction to compute the sizes of the buckets [S1, unofficial].
7. **Prefix Scan (Down Sweep):** Execute the `DOWN_SWEEP_PREFIX_SCAN` compute shader. This completes the prefix scan to determine the starting indices for each bucket [S1, unofficial].
8. **Reset Last Position:** Execute the `RESET_LAST_POSITION_PREFIX_SCAN` compute shader. This step finalizes the bucket boundary positions [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for Bucket-based LBVH construction
# Based on [S1, unofficial]

def build_bvh(voxel_model):
    # Step 1: Aggregate data
    volatileGPUData = aggregateSSBOData(voxel_model)
    
    # Step 2: Find best neighbor
    # Uses BVH_BUILDING_RADIUS = 100
    FIND_BEST_NEIGHBOR(volatileGPUData, BVH_BUILDING_RADIUS)
    
    # Step 3: Merge clusters
    CLUSTER_MERGING(volatileGPUData)
    
    # Step 4: Reallocate clusters
    REALLOCATE_CLUSTERS(volatileGPUData)
    
    # Step 5: End loop computations
    END_LOOP_COMPUTATIONS(volatileGPUData)
    
    # Step 6: Prefix scan - Reduce
    REDUCE_PREFIX_SCAN(volatileGPUData)
    
    # Step 7: Prefix scan - Down sweep
    DOWN_SWEEP_PREFIX_SCAN(volatileGPUData)
    
    # Step 8: Reset last position
    RESET_LAST_POSITION_PREFIX_SCAN(volatileGPUData)
    
    return volatileGPUData.bvh_structure
```

**Edge Cases and Pitfalls:**
- The `BVH_BUILDING_RADIUS` is a static constant. If the voxel distribution is significantly denser or sparser than expected, the neighbor finding may be suboptimal [S1, unofficial].
- The exact data layout of `volatileGPUData` is not specified, which may require implementation-specific adjustments [S1, unofficial].

## Subtopics

### [Bucket Size Computation](8964-bucket-size-computation.md)
Computes the bucket texture storing the position of the last voxel of each bucket. It splits computation into fragment and vertex shader steps to avoid inefficient vertex shader texture fetches.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Performance:** Allows for two to three times higher frame rates than using no empty-space skipping during ray marching [S2, unofficial].
- **Strengths:** Adapts to sparse volumes better than uniform grids [S2, unofficial].
- **Limitations:** Requires multiple compute shader passes, which may increase overhead for very small volumes [S1, unofficial].

## Background
The method is based on the work of Fernandes and Walter [FW20], who proposed grouping voxels into buckets for LBVH construction [S2, unofficial]. It is implemented in the `VoxelFragmentML` project by AlfonsoLRz [S1, unofficial].

## Sources
[S1] AlfonsoLRz/VoxelFragmentML, https://github.com/AlfonsoLRz/VoxelFragmentML, fetched 2026-10-26 (unofficial)
[S2] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
