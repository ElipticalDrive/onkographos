# Region Clustered BVH Volume Rendering
Also called: GPU BVH ESS, OptiX DVR

## Overview
Region Clustered BVH Volume Rendering is a GPU-based Empty Space Skipping (ESS) technique that accelerates direct volume rendering by constructing a Bounding Volume Hierarchy (BVH) from clustered active regions of a volume, which is then traced via NVIDIA OptiX to skip empty space [S1]. Unlike traditional GPU DVR methods that rely on octrees or rasterization-based occupancy geometry (such as Sparseleap), this method leverages BVHs to represent sparse or thin strands of data with fewer nodes, avoiding the need for rasterization when the camera moves [S1].

## How it works
The method operates in two phases: an offline CPU-based construction phase that prepares the BVH from volume data, and an online GPU-based traversal phase using OptiX.

### Inputs and Outputs
*   **Input:** A volumetric dataset divided into fixed-size regions (bricks); a transfer function (TF) defining opacity thresholds.
*   **Output:** A BVH structure containing Axis-Aligned Bounding Boxes (AABBs) for clustered active regions, ready for ray tracing.

### Data Structures
1.  **Region Flags Array:** A 3D array (or bit-string) where each element corresponds to a region in the volume. The value is `1` if the region is active (contains opacity > 0 according to the TF) and `0` otherwise [S1].
2.  **3D Summed Area Table (3DSAT):** A precomputed 3D array derived from the Region Flags Array. It allows $O(1)$ queries (8 lookups) to determine the number of active regions within any rectangular sub-volume [S1].
3.  **Clustered AABBs:** A list of bounding boxes representing the spatial extent of each identified cluster of active regions.

### Algorithm Steps

1.  **Region Partitioning:** Divide the volume into fixed-size regions (bricks). For each region, store the minimum and maximum voxel values [S1].
2.  **Activity Testing:** When the transfer function updates, test all regions in parallel to determine if they are active. A region is active if its min/max values indicate the presence of opacity > 0. Store the result as a 3D bit-string `ActiveFlags` [S1].
3.  **3DSAT Construction:** Compute the 3D Summed Area Table from `ActiveFlags`. For any sub-volume defined by corners $(x_1, y_1, z_1)$ and $(x_2, y_2, z_2)$, the number of active regions $N$ is calculated using 8 lookups into the 3DSAT [S1].
4.  **Clustering:** Identify contiguous groups of active regions to form larger cube-shaped blocks (e.g., $2\times2\times2$, $3\times3\times3$, $4\times4\times4$) to reduce the total node count in the BVH [S1].
    *   Maintain a `ClusteredFlags` bit-string to track regions already assigned to a cluster.
    *   Sweep through the volume in descending order of cluster size (e.g., checking for $64^3$, then $63^3$, etc.).
    *   For each candidate cluster size $S$, iterate through all possible positions. If a candidate block of size $S$ consists entirely of active regions (verified via 3DSAT) and no region in the block is marked in `ClusteredFlags`, mark the block as a cluster and update `ClusteredFlags` [S1].
5.  **AABB Generation:** Calculate the spatial bounds (AABBs) for each identified cluster.
6.  **BVH Construction and Traversal:** Pass the list of AABBs to NVIDIA OptiX. OptiX constructs the BVH and initiates ray tracing for the volume rendering pipeline [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_region_clustered_bvh(volume_data, transfer_function, region_size):
    """
    Constructs a BVH for empty space skipping using region clustering.
    """
    # 1. Determine active regions
    # volume_data is a 3D array of voxels
    # transfer_function is a callable or lookup table
    active_flags = compute_active_flags(volume_data, transfer_function, region_size)
    
    # 2. Build 3D Summed Area Table for O(1) region counting
    sat = build_3d_summed_area_table(active_flags)
    
    # 3. Cluster active regions
    # Max cluster size to check (e.g., 64)
    max_cluster_dim = 64
    clustered_flags = create_zero_bit_string(active_flags.shape)
    clusters = []
    
    # Sweep in descending order of cluster size
    for size in range(max_cluster_dim, 1, -1):
        # Iterate over all possible top-left corners for a cube of 'size'
        for x in range(0, active_flags.shape[0] - size + 1):
            for y in range(0, active_flags.shape[1] - size + 1):
                for z in range(0, active_flags.shape[2] - size + 1):
                    # Check if this block is fully active using 3DSAT
                    # 8 lookups into sat
                    count = query_3dsat(sat, x, y, z, x + size - 1, y + size - 1, z + size - 1)
                    
                    expected_count = size * size * size
                    
                    if count == expected_count:
                        # Check if any region in this block is already clustered
                        if not is_any_region_clustered(clustered_flags, x, y, z, size):
                            # Mark this block as a cluster
                            mark_regions_as_clustered(clustered_flags, x, y, z, size)
                            
                            # Calculate AABB for this cluster
                            aabb = compute_aabb_from_region_coords(x, y, z, size, region_size)
                            clusters.append(aabb)
                            
    # 4. Build BVH from clusters
    bvh = optix_build_bvh(clusters)
    return bvh

def compute_active_flags(volume, tf, region_size):
    """
    Returns a 3D bit-string where 1 indicates an active region.
    """
    flags = zeros_like(volume, dtype=bool)
    # Parallel test of all regions
    for each region r in volume:
        min_val, max_val = get_region_min_max(volume, r)
        if tf_has_opacity(min_val, max_val):
            flags[r] = True
    return flags

def query_3dsat(sat, x1, y1, z1, x2, y2, z2):
    """
    Returns the sum of active flags in the box defined by corners.
    Uses 8 lookups into the 3D SAT.
    """
    # Standard 3D SAT inclusion-exclusion principle
    return (
        sat[x2, y2, z2]
        - sat[x1-1, y2, z2] - sat[x2, y1-1, z2] - sat[x2, y2, z1-1]
        + sat[x1-1, y1-1, z2] + sat[x1-1, y2, z1-1] + sat[x2, y1-1, z1-1]
        - sat[x1-1, y1-1, z1-1]
    )
```

### Parameters and Edge Cases
*   **Cluster Shapes:** The method prefers cube-shaped clusters ($2\times2\times2$, $3\times3\times3$, etc.) [S1]. Future work suggests investigating non-cubed shapes (e.g., $2\times2\times8$) [S1].
*   **Precision:** The 3DSAT query requires exact integer arithmetic to ensure `count == expected_count`. Floating-point errors in TF evaluation should be avoided [S1].
*   **Degenerate Input:** If the volume is entirely empty, the BVH will be empty, and no rays will be traced [S1]. If the volume is entirely active, the clustering may result in a single large block or a few large blocks, depending on the sweep order [S1].

## Operating characteristics
*   **Time Complexity:** The clustering step involves a sweep over the volume dimensions. For a volume of size $N$ and max cluster size $K$, the complexity is roughly $O(N \cdot K^3)$ in the worst case if not optimized, but the 3DSAT allows fast validation [S1]. The BVH build time is noted as not a hindering factor [S1].
*   **Memory Complexity:** Requires storage for the volume data, the active flags bit-string, the 3DSAT (same dimensions as active flags), and the BVH nodes [S1].
*   **Performance:** Tested on an Intel Xeon E5-1620 v2 CPU and NVIDIA RTX2080 GPU [S1]. The method shows performance benefits over non-clustered BVHs and avoids the rasterization overhead of Sparseleap [S1].
*   **Strengths:** Efficiently handles sparse data and thin strands with fewer BVH nodes than octrees; avoids rasterization overhead on camera move [S1].
*   **Limitations:** Build time on CPU; clustering heuristic is limited to fully active cubes in the basic implementation [S1].

## Background
*   **Origin:** The method was presented in the context of re-evaluating BVHs for GPU-based ESS for Direct Volume Rendering, leveraging NVIDIA OptiX and RTX RTCore hardware [S1].
*   **History:** BVHs were traditionally used on CPUs for full-resolution DVR (Knoll et al. 2011) but were not standard on GPUs due to build times and traversal logic [S1]. This method adapts them for GPU ESS using hardware-accelerated ray tracing [S1].
*   **Comparison:** It differs from Sparseleap (Hadwiger et al. 2018), which uses octrees to generate occupancy geometry that must be rasterized on camera move [S1]. It also differs from traditional GPU DVR octrees, which can become deep and expensive for sparse data [S1].

## Sources
[S1] An Analysis of Region Clustered BVH Volume Rendering ..., https://www.highperformancegraphics.org/wp-content/uploads/2019/session1/region_clustered_bvh_volume_rendering.pdf, fetched 2026-10-28
