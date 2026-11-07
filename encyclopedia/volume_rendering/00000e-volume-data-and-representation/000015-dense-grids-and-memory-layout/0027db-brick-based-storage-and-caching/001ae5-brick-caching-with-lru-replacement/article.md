# Brick Caching with LRU Replacement

Also called: Block Memory Management Unit (MMU) with LRU Eviction, Brick Pool Manager, Dynamic Brick Cache, Volume Block Cache

## Overview
Brick Caching with LRU Replacement manages a fixed-size pool of volume data blocks, mapping logical brick coordinates to physical memory slots and evicting the least-recently-used blocks when the pool is full [S2, unofficial]. It solves the problem of limited GPU memory capacity and transfer bandwidth by allowing rendering of volumes larger than physical memory, reducing rendering times from days to minutes for high-resolution datasets [S1][S3, unofficial]. It sets itself apart from sibling methods by explicitly maintaining a sorted list of blocks along the ray depth to facilitate immediate lookup, rather than relying solely on generic LRU eviction without spatial locality awareness [S1]. Unlike [Multi-Level Cache and Prefetching](4631-multi-level-cache-and-prefetching.md) which uses hierarchical prefetching, this method focuses on a single flat pool with strict LRU eviction [S2, unofficial].

## How it works
**Inputs:**
- Volume dimensions $D_x, D_y, D_z$ [S2, unofficial].
- Brick dimensions $B_x, B_y, B_z$ (typically $32^3$ voxels) [S1][S2, unofficial].
- Cache capacity ratio $\alpha$ (e.g., 0.5 for 50% of total bricks) [S2, unofficial].
- Transfer function $TF$ (optional, for active brick determination) [S1].
- Count buffer $cnt\_buf$ precomputed from disk, size $N_x \cdot N_y \cdot N_z \cdot 256$ [S2, unofficial].

**Outputs:**
- A pointer to the voxel data of the requested brick $(i, j, k)$, or `nullptr` if the brick is not resident or empty [S2, unofficial].

**Data Structures:**
The cache is implemented as a Block Memory Management Unit (MMU) containing the following fields [S2, unofficial]:

1.  **dims**: Vector3i, original volume dimensions.
2.  **brick_dims**: Vector3i, number of bricks per dimension:
    $$ N_x = \lceil D_x / B_x \rceil, \quad N_y = \lceil D_y / B_y \rceil, \quad N_z = \lceil D_z / B_z \rceil $$
3.  **brick_size**: Vector3i, dimensions of each brick in voxels $(B_x, B_y, B_z)$ [S2, unofficial].
4.  **entries**: Array of `Entry` structures, size $N_x \cdot N_y \cdot N_z$. Each `Entry` contains:
    -   `valid`: bool, indicates if the mapping is active.
    -   `blk_idx`: int, index into the `blocks` array.
5.  **blocks**: Array of `Block` structures, size $blk\_nums$. Each `Block` contains:
    -   `brick_coords`: Vector3i, logical coordinates $(i, j, k)$.
    -   `last_access`: int, timestamp or counter for LRU ordering.
    -   `data_ptr`: pointer to the voxel data in the `data` pool.
6.  **blk_nums**: int, total number of blocks in the pool, calculated as:
    $$ blk\_nums = N_x \cdot N_y \cdot N_z \cdot \alpha $$
7.  **clock_pos**: int, current position in the LRU clock for eviction search [S2, unofficial].
8.  **blk_used**: int, number of currently used blocks in the pool.
9.  **data**: uchar*, linear memory pool of size $blk\_nums \times (B_x+1)(B_y+1)(B_z+1)$ [S2, unofficial].
10. **cnt_buf**: int*, buffer of size $N_x \cdot N_y \cdot N_z \cdot 256$ storing voxel counts for each isovalue level (0-255) for every brick [S2, unofficial].
11. **mutex**: pthread_mutex_t, for thread-safe access [S2, unofficial].
12. **fin**: ifstream, file handle for loading bricks from disk [S2, unofficial].

**Algorithm Steps:**

1.  **Initialization:**
    -   Calculate $N_x, N_y, N_z$ from volume and brick dimensions.
    -   Allocate `entries` array of size $N_x \cdot N_y \cdot N_z$, initializing all `valid` to false.
    -   Calculate $blk\_nums$ using ratio $\alpha$.
    -   Allocate `blocks` array of size $blk\_nums$.
    -   Allocate `data` buffer of size $blk\_nums \times (B_x+1)(B_y+1)(B_z+1)$.
    -   Load `cnt_buf` from disk.
    -   Initialize `clock_pos` to 0, `blk_used` to 0 [S2, unofficial].

2.  **Get Block Request `getBlock(x, y, z)`:**
    -   **Boundary Check:** If $x \ge N_x$ or $y \ge N_y$ or $z \ge N_z$, return `nullptr` [S2, unofficial].
    -   **Empty Brick Check (Optimization):**
        -   If a transfer function $TF$ is provided, iterate index $idx$ from 0 to 255.
        -   Check if $cnt\_buf[base + idx] \neq 0$ AND $TF.sampleSimple(idx).w \neq 0$.
        -   If no such $idx$ is found (loop completes with $idx=256$), return `nullptr` (brick is effectively empty for this transfer function) [S2, unofficial].
    -   **Cache Hit Check:**
        -   Calculate linear index $L = x + y \cdot N_x + z \cdot N_x \cdot N_y$.
        -   If `entries[L].valid` is true:
            -   Retrieve `blk_idx = entries[L].blk_idx`.
            -   Update `blocks[blk_idx].last_access` to current timestamp.
            -   Return `blocks[blk_idx].data_ptr` [S2, unofficial].
    -   **Cache Miss / Eviction:**
        -   Lock `mutex` [S2, unofficial].
        -   If `blk_used < blk_nums`:
            -   Find the first free slot in `blocks` (where `brick_coords` is invalid/unused).
            -   Let this slot be `new_blk_idx`.
        -   Else (pool full):
            -   Perform LRU eviction search starting from `clock_pos`.
            -   Iterate through `blocks` circularly. If `blocks[i].last_access` is older than a threshold or simply the minimum found so far, mark it for eviction.
            -   *Note: The source describes a "clock" mechanism where `clock_pos` finds the first unused block or evicts the least recently used.*
            -   Let `old_blk_idx` be the evicted block.
            -   Invalidate `entries` mapping for `blocks[old_blk_idx].brick_coords`.
            -   Set `new_blk_idx = old_blk_idx` [S2, unofficial].
        -   **Load Brick:**
            -   Read brick data from file `fin` at logical coordinates $(x, y, z)$.
            -   Store data into `data` pool at offset `new_blk_idx * brick_volume`.
            -   Update `blocks[new_blk_idx]`:
                -   `brick_coords = (x, y, z)`
                -   `last_access = current_timestamp`
                -   `data_ptr = &data[new_blk_idx * brick_volume]`
            -   Update `entries[L]`:
                -   `valid = true`
                -   `blk_idx = new_blk_idx`
            -   Increment `blk_used` if a new slot was allocated.
            -   Update `clock_pos` to `(new_blk_idx + 1) % blk_nums` [S2, unofficial].
        -   Unlock `mutex` [S2, unofficial].
        -   Return `blocks[new_blk_idx].data_ptr` [S2, unofficial].

3.  **Isovalue Change Handling:**
    -   When the isovalue changes, perform a range query on `cnt_buf` for all bricks to determine active status [S1].
    -   Download active bricks into the cache.
    -   Remove inactive bricks using the LRU strategy when space is needed [S1].

**Parameters:**
-   **brick_size**: Typically $32^3$ voxels [S1][S2, unofficial].
-   **_brick_num_ratio ($\alpha$)**: Fraction of total bricks to keep in memory, e.g., 0.5 [S2, unofficial].
-   **GPU Resolution Limit**: Example $512^3$ on NVIDIA GeForce 6 [S1].
-   **Memory Limit**: Example 256MB [S1].

**Edge Cases and Pitfalls:**
-   **Precision:** The count buffer uses 256 levels (uchar/int mapping). If the volume data has higher precision, binning or sampling is required to populate `cnt_buf` [S2, unofficial].
-   **Boundaries:** Bricks are stored with 1-voxel padding $(B_x+1)(B_y+1)(B_z+1)$ to support trilinear interpolation at boundaries without accessing adjacent bricks [S1][S2, unofficial].
-   **Concurrency:** The use of `pthread_mutex_t` indicates this implementation is designed for multi-threaded CPU access. On GPU, atomic operations or a different synchronization primitive would be required [S2, G, unofficial].

## Subtopics

### [Block-based Ray Cache](2260-block-based-ray-cache.md)
The Block-based Ray Cache is a memory optimization technique that reduces reconstruction overhead by retaining reconstructed voxel blocks in memory for reuse by adjacent rays. It solves the problem of redundant computation in high-resolution volume rendering where data exceeds physical memory, allowing rendering times to drop from days to minutes. It sets itself apart from sibling methods by explicitly maintaining a sorted list of blocks along the ray depth to facilitate immediate lookup, rather than relying solely on generic LRU eviction without spatial locality awareness.

### [LRU Cache Eviction with Residency Update](3115-lru-cache-eviction-with-residency-update.md)
This method couples LRU brick cache eviction with hierarchical residency octree updates. When the cache lacks a free slot, it evicts the least recently accessed brick and resets bitmask entries in affected leaf and non-leaf nodes to reflect the loss of resident data. It differs from standard LRU caches by synchronizing cache state with octree metadata after each eviction.

### [Brick Cache Implementation](3222-brick-cache-implementation.md)
The Brick Cache is a single-channel 3D texture cache that stores bricks from a multi-resolution multi-volume hierarchy. It is agnostic to the number of data channels in the underlying dataset. It differs from general-purpose CPU caches by being implemented as a GPU-resident texture managed via LRU replacement using WebGPU compute passes.

### [GPU-Steered LRU Cache Management](3654-gpu-steered-lru-cache-management.md)
This method enables the GPU to steer CPU-side data loading for voxel rendering by reporting node usage, thereby eliminating the overhead of CPU-driven visibility queries and tree traversal. It differs from sibling methods by unifying the management of the brick pool and node pool into two LRU-controlled caches where the GPU determines refinement needs and missing data, rather than relying on occlusion queries or frustum containment checks performed on the CPU.

### [LRU Cache Management for Voxel Bricks](4571-lru-cache-management-for-voxel-bricks.md)
LRU Cache Management for Voxel Bricks is a memory management strategy that prioritizes reusing voxel brick data that has not been accessed for the longest time, thereby optimizing memory reuse while retaining useful data in the cache. It solves the problem of limited GPU memory capacity and transfer bandwidth by ensuring that expensive data transfers from slower host memory are minimized. Unlike sibling methods that may rely on hardware-specific features or different replacement policies, this approach explicitly manages the cache lifecycle using a Last Recently Used (LRU) policy, often requiring CPU-side coordination due to the limitations of standard shader models regarding random memory writes.

### Brick Cache Updates
A variant involving dynamic updates from a file server [S3, unofficial].
-   **Upload:** A user-defined number of brick data is uploaded to the GPU each frame. Each brick comprises its global brick ID and either the EMPTY value (if all zeros) or the pre-processed brick [S3, unofficial].
-   **Translate ID:** The brick's global brick ID is translated to a local brick ID [S3, unofficial].
-   **Discard:** If the brick's channel is not represented by the multi-channel page-table hierarchy, the brick is discarded [S3, unofficial].
-   **Handle EMPTY:** If the brick is EMPTY, the corresponding page in the multi-channel page-table hierarchy is marked as EMPTY [S3, unofficial].
-   **Handle MAPPED:** Otherwise, the corresponding page is marked as MAPPED, and the brick's data is added to the brick cache [S3, unofficial].
-   **Find Entry:** The LRU buffer is used to find free cache entries or entries not accessed recently [S3, unofficial].
-   **Handle Removal:** When a brick is removed from the cache, its corresponding page is marked as UNMAPPED [S3, unofficial].
-   **Pass Lists:** Two lists of local brick IDs are passed to the residency octree: 1) IDs of bricks marked EMPTY or MAPPED, 2) IDs of bricks marked UNMAPPED [S3, unofficial].

## Operating characteristics
-   **Time Complexity:** $O(1)$ average for cache hit. $O(1)$ for eviction search (amortized using clock algorithm). $O(256)$ worst-case for empty brick check (iterating the count buffer) [S2, unofficial].
-   **Memory Complexity:** $O(blk\_nums \times brick\_volume + brick\_count \times 256 \times sizeof(int))$ [S2, unofficial]. The overhead is inversely proportional to brick size due to padding [S1].
-   **Measured Performance:** Example resolution $512^3$ on NVIDIA GeForce 6 with 256MB memory limit [S1]. Rendering times can drop from days to minutes for high-resolution volumes [S1][S3, unofficial].
-   **Strengths:** Decouples volume size from GPU memory restrictions; efficient space skipping via count buffer; simple implementation [S1][S2, unofficial].
-   **Limitations:** Single-level cache may suffer from thrashing if working set exceeds pool size; mutex contention on CPU; 256-level count buffer may not capture all transfer function nuances [S1][S2, unofficial].
-   **When to use:** When volume size exceeds available VRAM/CPU cache and a single-level cache is sufficient.
-   **When not to use:** When hierarchical LOD is required or when GPU-steered updates are needed for better performance [S1][S3, unofficial].

## Background
Originating from real-time ray-casting research for discrete isosurfaces, this method was described in work by Sigg and Hadwiger (2005) [S1]. It builds on the idea of adaptive texture maps [KE02] to manage memory for large volumes. The technique is used in web-based volume rendering of large biomedicine datasets [S3, unofficial] and in parallel volume rendering implementations [S2, unofficial].

## Sources
[S1] Real-Time Ray-Casting and Advanced Shading of Discrete Isosurfaces, https://cgl.ethz.ch/people/archive/siggc/publications/eg05.pdf, fetched 2026-10-26
[S2] hopexn/VolumeRenderingParallel, https://github.com/hopexn/VolumeRenderingParallel, fetched 2026-10-27 (unofficial)
[S3] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
