# Variance-Adaptive Multi-Resolution Grid

Also called: Adaptive Resolution Merging, Multi-resolution grid, Single-resolution grid, Supereight2, TSDF Variance Downsampling, Voxel Merging via Variance Threshold

## Overview
The Variance-Adaptive Multi-Resolution Grid is a sparse voxel representation method that dynamically adjusts voxel resolution based on the temporal variance of Truncated Signed Distance Field (TSDF) values within voxel blocks [S1]. It solves the problem of memory inefficiency in large-scale 3D reconstruction by allocating high-resolution voxels only in geometrically complex regions (high variance) and merging them into coarser representations in flat regions (low variance) [S1]. Unlike its siblings, it uses a single flat hash table to index blocks of varying resolutions, avoiding recursive data structures to facilitate efficient GPU parallel execution [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A sequence of observations (RGB-D images or LiDAR point clouds) with known camera poses.
*   **Output:** A multi-resolution sparse voxel grid stored in a hash table, where each entry points to a voxel block at a specific resolution level.

**Data Structures:**
The grid is managed via a flat hash table. Each entry in the hash table corresponds to a 3D block coordinate $(x, y, z)$.
*   **Hash Function:** The block coordinates are hashed using three prime numbers to map to hash table buckets [S1]:
    *   $p_1 = 73856093$
    *   $p_2 = 19349669$
    *   $p_3 = 83492791$
*   **Hash Table Entry:** Contains a pointer to a block in a heap specific to that resolution level [S1].
*   **Variance State:** For each voxel, the method maintains the running mean $D_i$ and the accumulated sum of squared deviations $S_{2,i}$ to compute variance incrementally [S1].

**Parameters:**
*   **Variance Threshold:** A predefined threshold to decide merging. Not given in the sources [S1].
*   **Voxel Size:** 0.01 m for dense RGB-D input; 0.20 m for sparse LiDAR input [S1].
*   **Hash Table Configuration:** Bucket size of 10 hash entries; eventual linked list size of 7 elements [S1].
*   **Quadtree Subdivision (for rendering):** Threshold contrast of 0.1; minimum pixel size of 1 [S1].

**Algorithm Steps:**

1.  **Initialization:** Initialize all voxel blocks to the finest resolution [S1].
2.  **Variance Aggregation:**
    *   For each voxel $i$ receiving a new observation $d_k$, update the running mean $D_{i,k}$ and the sum of squared deviations $S_{2,i,k}$ using Welford's algorithm [S1]:
        $$ S_{2,ik} = S_{2,ik-1} + (d_k - D_{i,k-1})(d_k - D_{i,k}) $$
        $$ \sigma_i^2 = \frac{S_{2,i,k}}{k} $$
    *   Aggregate the voxel-wise variances $\sigma_i^2$ at the block level to compute the mean variance for each block [S1].
3.  **Merging Decision:**
    *   Compare the block's mean variance against the predefined threshold [S1].
    *   If the mean variance is **below** the threshold, reallocate the block at a coarser resolution (merge voxels) [S1].
    *   If the mean variance is **above** the threshold, keep the block at the current resolution [S1].
4.  **Re-integration:** Re-integrate the merged block into the grid at the coarser resolution [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
P1 = 73856093
P2 = 19349669
P3 = 83492791
VARIANCE_THRESHOLD = Not given in the sources # [S1]

class VoxelBlock:
    def __init__(self, resolution):
        self.resolution = resolution
        # Initialize variance tracking for all voxels in the block
        self.means = [0.0] * num_voxels
        self.sum_sq_devs = [0.0] * num_voxels
        self.counts = [0] * num_voxels

def hash_block_coords(x, y, z):
    # Simple linear congruential hash using the specified primes
    # Note: Exact hash table indexing logic (bucketing) is implementation-specific
    # but relies on these primes for coordinate mapping [S1]
    h = x * P1 + y * P2 + z * P3
    return h % HASH_TABLE_SIZE

def update_variance(voxel_index, new_value, block):
    k = block.counts[voxel_index]
    old_mean = block.means[voxel_index]
    
    # Welford's online algorithm update
    k += 1
    delta = new_value - old_mean
    new_mean = old_mean + delta / k
    delta2 = new_value - new_mean
    block.sum_sq_devs[voxel_index] += delta * delta2
    block.means[voxel_index] = new_mean
    block.counts[voxel_index] = k

def compute_block_variance(block):
    total_variance = 0.0
    num_voxels = len(block.counts)
    for i in range(num_voxels):
        if block.counts[i] > 0:
            # Variance is sum of squared deviations divided by count
            # Note: Using k as denominator as per Eq. 6 [S1]
            var = block.sum_sq_devs[i] / block.counts[i]
            total_variance += var
    return total_variance / num_voxels

def process_frame(observations, hash_table):
    for obs in observations:
        voxel_coords = project_to_voxel_grid(obs)
        for v in voxel_coords:
            block_key = hash_block_coords(v.x, v.y, v.z)
            
            # Get or create block at finest resolution initially
            if block_key not in hash_table:
                hash_table[block_key] = VoxelBlock(resolution="finest")
            
            block = hash_table[block_key]
            voxel_index = get_local_voxel_index(v, block.resolution)
            
            # Update variance incrementally
            update_variance(voxel_index, obs.depth_value, block)

    # Post-processing: Check for merging
    for block_key, block in hash_table.items():
        mean_var = compute_block_variance(block)
        
        if mean_var < VARIANCE_THRESHOLD:
            # Merge to coarser resolution
            coarser_block = merge_block_to_coarser(block)
            hash_table[block_key] = coarser_block
```

**Edge Cases and Pitfalls:**
*   **Precision:** The use of Welford's algorithm ensures numerical stability for incremental variance computation, avoiding the need to store full observation history [S1].
*   **Boundaries:** The method assumes ground-truth fixed poses for evaluation [S1].
*   **Degenerate Input:** The method is sensor-agnostic and does not require sensor-specific noise parameters [S1].

## Subtopics

### [Multi-Grid Merging](7905-multi-grid-merging.md)
Merges multiple independent grids rather than adjusting resolution within a single grid structure.

### [Neuralangelo Hashgrid Configuration](8523-neuralangelo-hashgrid-configuration.md)
Configures hash grids specifically for neural surface reconstruction, focusing on frequency encoding rather than variance-based merging.

### [Hierarchical Voxel Block Hashing](12402-hierarchical-voxel-block-hashing.md)
Uses a hierarchical tree structure for block indexing, contrasting with the flat hash table approach.

### [Multi-Level Morton Code Data Structure](18700-multi-level-morton-code-data-structure.md)
Uses Morton codes for spatial indexing across multiple resolutions, differing from the prime-based coordinate hashing.

### [Instant-NGP Grid Hierarchy](18729-instant-ngp-grid-hierarchy.md)
Employs a fixed multi-resolution grid hierarchy for neural rendering, not dynamically merged based on variance.

### [Bin-Hash Index Construction](20205-bin-hash-index-construction.md)
Constructs indices using binning strategies, differing from the direct prime-based hashing of block coordinates.

## Operating characteristics
*   **Time Complexity:** The method is implemented in a fully parallel manner on the GPU, using intra-block reductions to accumulate variance terms [S1].
*   **Memory Complexity:** Achieves memory savings of 2.0× to 7.5× for RGB-D data and 2.3× to 2.9× for LiDAR data compared to baselines (except VDBFusion) [S1].
*   **Performance:** Processing speed is up to 13× faster than VDBFusion and Voxblox [S1].
*   **Strengths:** Sensor-agnostic, numerically stable variance computation, efficient GPU parallelization due to flat hash table structure [S1].
*   **Limitations:** Requires a predefined variance threshold (value not specified) [S1]. Single-resolution variant yields speedups but multi-resolution is needed for large-scale/sparse scenes to preserve quality [S1].
*   **When to use:** Large-scale 3D reconstruction with RGB-D or LiDAR inputs where memory efficiency and speed are critical [S1].
*   **When not to use:** When ground-truth poses are unavailable or when sensor-specific noise modeling is preferred over variance-based heuristics [S1].

## Background
The method was introduced in the paper "Resolution Where It Counts: Hash-based GPU-Accelerated 3D ..." [S1]. It is used for 3D reconstruction on the ScanNet (RGB-D) and NC (LiDAR) datasets with ground-truth fixed poses [S1]. The approach builds on the concept of sparse voxel grids but introduces variance-driven allocation to optimize memory usage without compromising geometric accuracy [S1].

## Sources
[S1] Resolution Where It Counts: Hash-based GPU-Accelerated 3D ..., https://arxiv.org/abs/2511.21459, fetched 2026-10-29
