# Brick Cache Implementation
Also called: Brick Cache, Brick Cache Layout
## Overview
The Brick Cache is a single-channel 3D texture cache that stores bricks from a multi-resolution multi-volume hierarchy [S1, unofficial]. It is agnostic to the number of data channels in the underlying dataset [S1, unofficial]. It differs from general-purpose CPU caches by being implemented as a GPU-resident texture managed via LRU replacement using WebGPU compute passes [S1, unofficial].
## How it works
### Inputs and Outputs
**Inputs:**
*   `Csize`: Cache dimensions $(Csize_x, Csize_y, Csize_z)$, chosen by the user at initialization [S1, unofficial][S2].
*   `Bsize`: Brick dimensions $(Bsize_x, Bsize_y, Bsize_z)$ [S1, unofficial][S2].
*   `m`: Number of data channels [S2].
*   `k`: Number of resolution levels [S2].
*   `Usage Buffer`: A 3D texture in `r32uint` format storing the current global timestamp for each brick-cache entry [S1, unofficial].

**Outputs:**
*   `Brick Cache`: A single-channel 3D texture in `r8unorm` format containing the brick data [S1, unofficial][S2].
*   `LRU Buffer`: A storage buffer in GPU memory containing an array of linear indices into the brick cache, sorted by most recent access time [S1, unofficial].

### Data Structures
1.  **Brick Cache**: Implemented as a single-channel 3D texture in `r8unorm` format [S1, unofficial][S2]. Each entry corresponds to a single brick in the multi-resolution multi-volume hierarchy [S1, unofficial]. The cache size $Csize$ is rounded up to be a multiple of the brick size $Bsize$ [S1, unofficial].
2.  **Usage Buffer**: A 3D texture in `r32uint` format with one voxel per brick-cache entry [S1, unofficial]. Its size $Usize$ is defined as:
    $$
    Usize = \left( \frac{Csize_x}{Bsize_x}, \frac{Csize_y}{Bsize_y}, \frac{Csize_z}{Bsize_z} \right)
    $$
    [S1, unofficial]
3.  **LRU Buffer**: A storage buffer in GPU memory containing an array of linear indices into the brick cache [S1, unofficial]. The array is sorted by the most recent access time of the corresponding brick-cache entry [S1, unofficial].

### Algorithm: LRU Buffer Update
The LRU buffer is updated after each frame using WebGPU compute passes [S1, unofficial].
1.  **Process Usage Buffer**: Process the usage buffer to produce a mask of brick-cache entries accessed at the current timestamp [S1, unofficial].
2.  **Compute Exclusive Scan**: Use a sequence of compute passes to compute the exclusive scan of this mask [S1, unofficial].
3.  **Move Indices**: Use the result of the scan to move all indices in the LRU buffer referencing brick-cache entries that have been used at the current timestamp to the front of the LRU buffer while preserving the order of the remaining elements [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Assumptions:
# - usage_buffer: list of ints (r32uint), size Usize
# - lru_buffer: list of ints (u32), size Csize
# - current_timestamp: int (current frame number)
# Brick cache is implicitly updated via separate upload passes [S1, unofficial].

def update_lru_buffer(usage_buffer, lru_buffer, current_timestamp):
    # Step 1: Produce mask of accessed entries
    # The usage buffer stores the timestamp of the last access per entry.
    mask = [1 if val == current_timestamp else 0 for val in usage_buffer]
    
    # Step 2: Compute exclusive scan of the mask
    # scan_result[i] holds the count of used entries before index i.
    scan_result = [0] * len(mask)
    current_sum = 0
    for i in range(len(mask)):
        scan_result[i] = current_sum
        current_sum += mask[i]
        
    # Step 3: Move indices referencing used entries to the front
    # Used entries (mask[cache_idx]==1) are placed at indices given by scan_result.
    # Unused entries (mask[cache_idx]==0) are placed after all used entries, preserving order.
    new_lru_buffer = [0] * len(lru_buffer)
    dest_unused = sum(mask)
    for i in range(len(lru_buffer)):
        cache_idx = lru_buffer[i]
        if mask[cache_idx] == 1:
            new_lru_buffer[scan_result[cache_idx]] = lru_buffer[i]
        else:
            new_lru_buffer[dest_unused] = lru_buffer[i]
            dest_unused += 1
    return new_lru_buffer
```
[G] The sequential Python logic mirrors the parallel WebGPU compute passes described in the source. The exclusive scan determines destination indices for used entries, while unused entries are appended sequentially to maintain stability [S1, unofficial].

### Parameters
*   $Csize$: Cache size, chosen by the user at initialization [S1, unofficial][S2].
*   $Bsize$: Brick size [S1, unofficial][S2].
*   $m$: Number of channels [S2].
*   $k$: Number of resolution levels [S2].

### Edge Cases and Pitfalls
*   **Cache Size Alignment**: $Csize$ must be rounded up to be a multiple of $Bsize$ [S1, unofficial].
*   **Precision**: The usage buffer uses `r32uint` for timestamps, which must be sufficient to avoid overflow over the lifetime of the cache [S1, unofficial].
*   **Boundary Conditions**: The usage buffer size $Usize$ is derived from $Csize/Bsize$ [S1, unofficial]. If $Csize$ is not a multiple of $Bsize$, the division may not be integer, but the source states $Csize$ is rounded up [S1, unofficial].
## Subtopics

### Block-based Ray Cache
Stores ray segments in blocks rather than individual bricks [2260-block-based-ray-cache.md] [G]. Chosen when ray coherence is high and block-level access patterns dominate [G].

### LRU Cache Eviction with Residency Update
Updates residency metadata during eviction rather than maintaining a strict LRU list [3115-lru-cache-eviction-with-residency-update.md] [G]. Chosen when metadata overhead must be minimized [G].

### GPU-Steered LRU Cache Management
Uses GPU-computed access statistics to drive eviction decisions [3654-gpu-steered-lru-cache-management.md] [G]. Chosen when CPU-GPU synchronization latency is a bottleneck [G].

### LRU Cache Management for Voxel Bricks
Manages LRU state specifically for voxel-level brick granularity [4571-lru-cache-management-for-voxel-bricks.md] [G]. Chosen when fine-grained access tracking is required [G].

## Operating characteristics
*   **Time Complexity**: The LRU update requires processing the usage buffer and computing an exclusive scan, which is $O(Csize)$ per frame [S1, unofficial].
*   **Memory Complexity**: The cache uses $Csize$ entries in `r8unorm` format [S1, unofficial]. The usage buffer uses $Usize$ entries in `r32uint` format [S1, unofficial]. The LRU buffer uses $Csize$ entries in `u32` format [S1, unofficial].
*   **Measured Performance**: Not given in the sources [S1, unofficial].
*   **Scaling**: Not given in the sources [S1, unofficial].
*   **Strengths**: GPU-resident, agnostic to data channels, efficient LRU updates via compute shaders [S1, unofficial].
*   **Limitations**: Single-channel texture format requires channel multiplexing or external metadata for native multi-channel storage [S1, unofficial].
*   **When to use**: Use when rendering large biomedicine datasets on WebGPU with strict memory constraints [S1, unofficial].
*   **When not to use**: Avoid when native multi-channel texture storage is required without additional mapping overhead [S1, unofficial].
## Background
The Brick Cache implementation is part of a scalable web-based volume rendering system for large biomedicine datasets [S1, unofficial]. It was developed to manage memory efficiently on the GPU using WebGPU [S1, unofficial]. The approach uses a residency octree for managing multi-channel data [S2].
## Sources
[S1] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
[S2] Residency Octree: A Hybrid Approach for Scalable Web ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC10840607/, fetched 2026-10-29
