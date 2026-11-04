# Brick Cache Implementation
Also called: Brick Cache, Brick Cache Layout
## Overview
The Brick Cache is a single-channel 3D texture cache that stores bricks from a multi-resolution multi-volume hierarchy [S1, unofficial]. It is agnostic to the number of data channels [S1, unofficial]. It differs from general-purpose CPU caches by being implemented as a GPU-resident texture managed via LRU replacement using WebGPU compute passes [S1, unofficial].
## How it works
### Inputs and Outputs
**Inputs:**
*   `Csize`: Cache dimensions $(Csize_x, Csize_y, Csize_z)$, chosen by the user at initialization [S1, unofficial][S2].
*   `Bsize`: Brick dimensions $(Bsize_x, Bsize_y, Bsize_z)$ [S1, unofficial][S2].
*   `m`: Number of channels [S2].
*   `k`: Number of resolution levels [S2].
*   `Usage Buffer`: A 3D texture in `r32uint` format indicating which cache entries were accessed in the current frame [S1, unofficial].

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
2.  **Compute Exclusive Scan**: Use a sequence of compute passes to compute the exclusive scan of the mask [S1, unofficial].
3.  **Move Indices**: Use the result of the scan to move all indices in the LRU buffer referencing brick-cache entries that have been used at the current timestamp to the front of the LRU buffer, while preserving the order of the remaining elements [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Assumptions:
# - usage_buffer: 3D texture (r32uint), size Usize
# - lru_buffer: storage buffer (u32), size Csize
# - current_timestamp: u32 (current frame number)
# - brick_cache: 3D texture (r8unorm), size Csize

def update_lru_buffer():
    # Step 1: Produce mask of accessed entries
    # The usage buffer is written to during rendering when a brick is accessed.
    # We need to identify which entries in the usage buffer correspond to the current timestamp.
    # Note: The source describes processing the usage buffer to produce a mask.
    # We assume the usage buffer stores timestamps or a boolean flag per entry.
    # Based on "produce a mask of brick-cache entries accessed at the current timestamp":
    
    mask = compute_access_mask(usage_buffer, current_timestamp)
    
    # Step 2: Compute exclusive scan of the mask
    # The mask is likely a boolean or integer array indicating access.
    # Exclusive scan produces indices for reordering.
    scan_result = exclusive_scan(mask)
    
    # Step 3: Move indices in LRU buffer
    # The LRU buffer contains linear indices into the brick cache.
    # We reorder these indices based on the scan result.
    new_lru_buffer = reorder_lru_indices(lru_buffer, scan_result)
    
    return new_lru_buffer

def compute_access_mask(usage_buffer, timestamp):
    # Returns a mask where 1 indicates access at current timestamp, 0 otherwise.
    # The usage buffer has one voxel per brick-cache entry.
    mask = []
    for i in range(len(usage_buffer)):
        if usage_buffer[i] == timestamp:
            mask.append(1)
        else:
            mask.append(0)
    return mask

def exclusive_scan(mask):
    # Computes exclusive prefix sum.
    # scan_result[i] = sum(mask[0]...mask[i-1])
    scan_result = [0] * len(mask)
    current_sum = 0
    for i in range(len(mask)):
        scan_result[i] = current_sum
        current_sum += mask[i]
    return scan_result

def reorder_lru_indices(lru_buffer, scan_result):
    # Moves indices referencing used entries to the front.
    # Preserves order of remaining elements.
    # This implies a stable partition based on the mask.
    # The scan result gives the destination index for each element.
    new_lru_buffer = [0] * len(lru_buffer)
    for i in range(len(lru_buffer)):
        if mask[i] == 1:
            # Used entries go to the front, ordered by their original position
            dest_index = scan_result[i]
            new_lru_buffer[dest_index] = lru_buffer[i]
        else:
            # Unused entries go to the back
            # The source says "preserving the order of remaining elements"
            # The scan result for unused elements (mask[i]==0) is the count of used elements before i
            # But we need to place them after all used elements.
            # Let's assume the scan result for unused elements is not directly the destination.
            # Actually, the exclusive scan of the mask gives the number of used elements before index i.
            # So used elements are placed at indices 0, 1, 2...
            # Unused elements need to be placed after the last used element.
            pass
            
    # Refined logic based on "move all indices... to the front... preserving order of remaining"
    # This is a stable partition.
    used_indices = []
    unused_indices = []
    for i in range(len(lru_buffer)):
        if mask[i] == 1:
            used_indices.append(lru_buffer[i])
        else:
            unused_indices.append(lru_buffer[i])
            
    new_lru_buffer = used_indices + unused_indices
    return new_lru_buffer
```
[G] The pseudocode above implements the stable partition described in the source. The source mentions an "exclusive scan" which is typically used for parallel prefix sums to determine destination indices in a parallel implementation. The sequential logic here achieves the same result: used elements are moved to the front, preserving their relative order, and unused elements follow, preserving their relative order.

### Parameters
*   $Csize$: Cache size, chosen by the user at initialization [S1, unofficial][S2].
*   $Bsize$: Brick size [S1, unofficial][S2].
*   $m$: Number of channels [S2].
*   $k$: Number of resolution levels [S2].

### Edge Cases and Pitfalls
*   **Cache Size Alignment**: $Csize$ must be rounded up to be a multiple of $Bsize$ [S1, unofficial].
*   **Precision**: The usage buffer uses `r32uint` for timestamps, which must be sufficient to avoid overflow over the lifetime of the cache [S1, unofficial].
*   **Boundary Conditions**: The usage buffer size $Usize$ is derived from $Csize/Bsize$. If $Csize$ is not a multiple of $Bsize$, the division may not be integer, but the source states $Csize$ is rounded up [S1, unofficial].

## Subtopics

### [LRU Cache Management for Voxel Bricks](4571-lru-cache-management-for-voxel-bricks.md)
General LRU management strategies for voxel bricks.

### [Brick Caching with LRU Replacement](879-brick-caching-with-lru-replacement.md)
The parent method, providing the high-level LRU replacement policy.

## Operating characteristics
*   **Time Complexity**: The LRU update requires processing the usage buffer and computing an exclusive scan, which is $O(Csize)$ per frame [S1, unofficial].
*   **Memory Complexity**: The cache uses $Csize$ entries in `r8unorm` format [S1, unofficial]. The usage buffer uses $Usize$ entries in `r32uint` format [S1, unofficial]. The LRU buffer uses $Csize$ entries in `u32` format [S1, unofficial].
*   **Strengths**: GPU-resident, agnostic to data channels, efficient LRU updates via compute shaders [S1, unofficial].
*   **Limitations**: Single-channel texture limits direct multi-channel storage without additional mapping [S1, unofficial].

## Background
The Brick Cache implementation is part of a scalable web-based volume rendering system for large biomedicinal datasets [S1, unofficial]. It was developed to manage memory efficiently on the GPU using WebGPU [S1, unofficial]. The approach uses a residency octree for managing multi-channel data [S2].

## Sources
[S1] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
[S2] Residency Octree: A Hybrid Approach for Scalable Web ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC10840607/, fetched 2026-10-29
