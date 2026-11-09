# Bucket-based LBVH construction

Also called: BVH Building, Bvh constructor, Fernandes and Walter LBVH construction

## Overview
Bucket-based LBVH construction builds a Bounding Volume Hierarchy (BVH) for voxel data by grouping voxels into dynamic buckets rather than a fixed uniform grid, addressing the inefficiency of static brick sizes in sparse volumes [S2, unofficial]. It sets itself apart from fixed-brick approaches (like Zellmann et al.'s LBVH) by adapting bucket boundaries to local voxel density, reducing empty-space traversal overhead [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of active voxels defined on a 3D integer grid $(i, j, k)$ with resolution $(N_x, N_y, N_z)$ [G].
*   **Output:** A BVH structure (nodes with bounding volumes and children) and a "Bucket Texture" storing the index of the last voxel of each bucket [S1, S2, unofficial].

**Data Structures:**
*   **Voxel List:** Array of active voxel coordinates.
*   **Bucket Texture:** 1D array where index $b$ stores the global index of the last voxel belonging to bucket $b$ [S2, unofficial].
*   **SSBOs (Shader Storage Buffer Objects):** Used for volatile GPU data during compute shader execution [S1, unofficial].

**Parameters:**
*   `BVH_BUILDING_RADIUS`: A static constant used in neighbor finding, set to 100 [S1, unofficial].
*   Bucket Size: Computed dynamically; see [Bucket Size Computation](8964-bucket-size-computation.md).

**Steps:**
1.  **Group Voxels into Buckets:** Instead of a fixed grid, voxels are assigned to buckets based on spatial locality. The boundaries are determined by the [Bucket Size Computation](8964-bucket-size-computation.md) method, which identifies the last voxel index for each bucket [S2, unofficial].
2.  **Compute Bucket Texture:** Identify bucket boundaries. This is split into:
    *   *Fragment Shader Step:* Identify boundary voxels.
    *   *Vertex Shader Step:* Construct the Bucket Texture to avoid expensive texture fetches in vertex shaders for boundary comparison [S2, unofficial].
3.  **Neighbor Finding:** Use a compute shader (`FIND_BEST_NEIGHBOR`) to find neighboring voxels/clusters within `BVH_BUILDING_RADIUS` [S1, unofficial].
4.  **Cluster Merging:** Use `CLUSTER_MERGING` to merge adjacent clusters based on neighbor information [S1, unofficial].
5.  **Reallocation:** Use `REALLOCATE_CLUSTERS` to adjust cluster assignments [S1, unofficial].
6.  **Prefix Scan for Hierarchy:**
    *   Use `REDUCE_PREFIX_SCAN` and `DOWN_SWEEP_PREFIX_SCAN` to compute prefix sums for hierarchical aggregation [S1, unofficial].
    *   Use `RESET_LAST_POSITION_PREFIX_SCAN` to manage position tracking during scans [S1, unofficial].
7.  **Finalize:** Use `END_LOOP_COMPUTATIONS` to finalize the BVH structure [S1, unofficial].
8.  **Aggregate Data:** Call `aggregateSSBOData(volatileGPUData)` to consolidate results from SSBOs [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for Bucket-based LBVH Construction
# Assumes compute shader stages are invoked in order

def build_bucket_lbvhi(voxels, resolution):
    # 1. Compute Bucket Boundaries
    # Uses Bucket Size Computation logic (see child article)
    bucket_boundaries = compute_bucket_boundaries(voxels)
    
    # 2. Construct Bucket Texture
    # Fragment shader identifies boundaries, vertex shader builds texture
    bucket_texture = construct_bucket_texture(bucket_boundaries)
    
    # 3. Initialize BVH Data Structures in SSBOs
    ssbo_data = initialize_ssbo(voxels)
    
    # 4. Compute Shaders Execution
    # Step 4a: Find Neighbors
    ssbo_data = find_best_neighbor(ssbo_data, radius=100) # BVH_BUILDING_RADIUS
    
    # Step 4b: Merge Clusters
    ssbo_data = cluster_merging(ssbo_data)
    
    # Step 4c: Reallocate Clusters
    ssbo_data = reallocate_clusters(ssbo_data)
    
    # Step 4d: Prefix Scans for Hierarchy
    ssbo_data = reduce_prefix_scan(ssbo_data)
    ssbo_data = down_sweep_prefix_scan(ssbo_data)
    ssbo_data = reset_last_position_prefix_scan(ssbo_data)
    
    # Step 4e: Finalize
    end_loop_computations(ssbo_data)
    
    # 5. Aggregate Results
    bvh = aggregate_ssbo_data(ssbo_data)
    
    return bvh, bucket_texture
```

**Edge Cases:**
*   **Sparse Volumes:** Fixed grids waste memory; bucketing adapts to density [S2, unofficial].
*   **Precision:** GPU compute shaders require careful handling of atomic operations in neighbor finding [G].
*   **Boundary Conditions:** Bucket boundaries must be correctly identified to avoid missing neighbors [S2, unofficial].

## Subtopics

### [Bucket Size Computation](8964-bucket-size-computation.md)
Computes the Bucket Texture storing the last voxel index of each bucket. It sets itself apart by splitting computation into fragment and vertex shader steps to avoid inefficient vertex shader texture fetches. Choose it when optimizing GPU-based bucket boundary identification for sparse volumes.

## Operating characteristics
*   **Time Complexity:** Parallel compute shader execution allows near-linear scaling with voxel count, dependent on bucket count and hierarchy depth [S1, unofficial].
*   **Memory Complexity:** Proportional to the number of voxels and buckets; Bucket Texture size equals number of buckets [S2, unofficial].
*   **Performance:** Allows 2-3x higher frame rates than no empty-space skipping [S2, unofficial]. Outperforms fixed-brick LBVH in sparse scenes [S2, unofficial].
*   **Strengths:** Adapts to local density; efficient GPU implementation via compute shaders [S1, S2, unofficial].
*   **Limitations:** Requires complex shader pipeline; bucket size computation adds overhead [S2, unofficial].
*   **When to Use:** Sparse volumetric data on GPU; when fixed-brick LBVH is inefficient [S2, unofficial].
*   **When Not to Use:** Dense volumes where fixed grids are efficient; CPU-only implementations [S2, unofficial].

## Background
Originated from Fernandes and Walter [FW20], who proposed grouping voxels into buckets for LBVH construction [S2, unofficial]. Implemented in projects like VoxelFragmentML [S1, unofficial]. Used in scalable web-based volume rendering of large biomedicinal datasets [S2, unofficial].

## Sources
[S1] AlfonsoLRz/VoxelFragmentML, https://github.com/AlfonsoLRz/VoxelFragmentML, fetched 2026-10-26 (unofficial)
[S2] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
