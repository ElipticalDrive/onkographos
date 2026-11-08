# Bucket-based LBVH construction

Also called: BVH Building, Bvh constructor, Fernandes and Walter LBVH construction

## Overview
Bucket-based LBVH construction builds a Linear Bounding Volume Hierarchy (LBVH) from a discrete voxel grid by grouping voxels into dynamic buckets rather than a fixed uniform grid [S2, unofficial]. It solves the inefficiency of fixed-brick approaches for sparse volumes by allowing bucket sizes to adapt to local voxel density [S2, unofficial]. It sets itself apart from sibling methods like [3D Voxel Grid Smoothing](833-3d-voxel-grid-smoothing.md) by focusing on spatial acceleration structure construction rather than data preprocessing, and from [Binary Voxel RLE Encoding](1071-binary-voxel-rle-encoding.md) by maintaining hierarchical bounding volumes for traversal rather than linear compression.

## How it works
**Inputs and Outputs:**
*   **Input:** A discrete voxel grid representing the volume data.
*   **Output:** An LBVH structure ready for traversal.

**Data Structures:**
*   **Voxel Grid:** The source occupancy/density data.
*   **Buckets:** Dynamic groups of voxels. Unlike fixed bricks, buckets are defined by boundaries that may vary or be computed dynamically [S2, unofficial].
*   **BVH Nodes:** Standard LBVH node structure (bounding volume, children indices).

**Parameters:**
*   `BVH_BUILDING_RADIUS`: A static constant used in neighbor finding, set to 100 [S1, unofficial].

**Steps:**
1.  **Aggregate Data:** Collect voxel data into a volatile GPU data structure [S1, unofficial].
2.  **Bucket Grouping:** Group voxels into buckets based on spatial proximity or density, deviating from the fixed brick size used in uniform grid approaches [S2, unofficial].
3.  **Neighbor Finding:** Use a compute shader (`FIND_BEST_NEIGHBOR`) to identify neighboring voxels or clusters within the search radius defined by `BVH_BUILDING_RADIUS` [S1, unofficial].
4.  **Cluster Merging:** Use a compute shader (`CLUSTER_MERGING`) to merge neighboring voxels/clusters into larger bounding volumes [S1, unofficial].
5.  **Reallocation:** Use a compute shader (`REALLOCATE_CLUSTERS`) to manage memory and cluster assignments dynamically [S1, unofficial].
6.  **Prefix Scan:** Perform a parallel prefix scan to determine bucket boundaries and node offsets. This involves:
    *   `REDUCE_PREFIX_SCAN`: Reduction phase.
    *   `DOWN_SWEEP_PREFIX_SCAN`: Down-sweep phase.
    *   `RESET_LAST_POSITION_PREFIX_SCAN`: Resetting position markers [S1, unofficial].
7.  **Finalization:** Use `END_LOOP_COMPUTATIONS` to finalize the BVH structure [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for Bucket-based LBVH Construction
# Based on [S1, unofficial] and [S2, unofficial]

def build_bucket_lbvhi(voxel_grid):
    # 1. Aggregate data into volatile GPU structure
    volatile_data = aggregateSSBOData(voxel_grid)
    
    # 2. Group voxels into buckets (dynamic, not fixed brick)
    # Implementation detail: This step replaces fixed grid partitioning
    buckets = group_voxels_into_buckets(volatile_data)
    
    # 3. Find best neighbors for each voxel/cluster
    # Uses BVH_BUILDING_RADIUS = 100
    neighbors = find_best_neighbor(volatile_data, radius=100)
    
    # 4. Merge clusters based on neighbors
    merged_clusters = cluster_merging(volatile_data, neighbors)
    
    # 5. Reallocate clusters in memory
    reallocated_clusters = reallocate_clusters(merged_clusters)
    
    # 6. Compute prefix scans for bucket boundaries and offsets
    # This determines the structure of the LBVH
    prefix_scan_result = reduce_prefix_scan(reallocated_clusters)
    prefix_scan_result = down_sweep_prefix_scan(prefix_scan_result)
    prefix_scan_result = reset_last_position_prefix_scan(prefix_scan_result)
    
    # 7. Finalize BVH
    bvh = end_loop_computations(prefix_scan_result)
    
    return bvh
```

**Edge Cases and Pitfalls:**
*   **Precision:** GPU-based parallel prefix scans require careful handling of atomic operations and memory synchronization to avoid race conditions [S1, unofficial].
*   **Boundary Conditions:** The definition of bucket boundaries must handle edge voxels correctly to ensure no voxels are missed or double-counted [S2, unofficial].
*   **Sparse Volumes:** The bucket approach is specifically designed to handle sparsity; however, if the volume is dense, the overhead of dynamic bucket management might outweigh benefits compared to fixed grids [S2, unofficial].

## Subtopics

### [Bucket Size Computation](8964-bucket-size-computation.md)
Computes the Bucket Texture storing the index of the last voxel of each bucket. It splits computation into fragment and vertex shader steps to avoid inefficient vertex shader texture fetches.

## Operating characteristics
*   **Time Complexity:** Parallel construction on GPU using compute shaders allows for near-linear scaling with voxel count, dominated by the prefix scan and neighbor finding steps [S1, unofficial].
*   **Memory Complexity:** Requires storage for voxel data, neighbor lists, and the BVH structure. Dynamic bucketing may require additional memory for cluster reallocation [S1, unofficial].
*   **Performance:** Allows for two to three times higher frame rates than no empty-space skipping during ray marching [S2, unofficial].
*   **Strengths:** Adapts to sparse volumes better than fixed-brick methods; efficient GPU parallelization [S2, unofficial].
*   **Limitations:** Complexity of implementation due to multiple compute shader stages and synchronization requirements [S1, unofficial].

## Background
Originated from work by Fernandes and Walter [FW20], as cited in [S2, unofficial]. The method is implemented in the `VoxelFragmentML` project by AlfonsoLRz [S1, unofficial]. It is part of the broader effort to optimize volume rendering for large biomedicinal datasets [S2, unofficial].

## Sources
[S1] AlfonsoLRz/VoxelFragmentML, https://github.com/AlfonsoLRz/VoxelFragmentML, fetched 2026-10-26 (unofficial)
[S2] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
