# Brick Caching with LRU Replacement

Also called: Block Memory Management Unit (MMU) with LRU Eviction, Brick Pool Manager, Dynamic Brick Cache, Volume Block Cache

## Overview
Brick Caching with LRU Replacement manages a fixed-size pool of volume data blocks, mapping brick coordinates to pool indices and evicting least-recently-used blocks when the pool is full [S2, unofficial]. It solves the problem of limited GPU memory capacity and transfer bandwidth by ensuring that expensive data transfers from slower host memory are minimized [S4, unofficial]. It sets itself apart from sibling methods by explicitly managing the cache lifecycle using a Last Recently Used (LRU) policy, often requiring CPU-side coordination due to the limitations of standard shader models regarding random memory writes [S1]. Unlike [Multi-Level Cache and Prefetching](4631-multi-level-cache-and-prefetching.md) which focuses on prefetching, or [Raymarching with LOD-based Brick Caching](5776-raymarching-with-lod-based-brick-caching.md) which integrates caching with LOD selection, this method focuses on the core eviction logic and memory pool management [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Volume dimensions, brick size, brick resolution levels, isovalue (for active brick determination), texture function (optional, for empty brick detection), file handle to volume data.
*   **Output:** A resident brick block given a brick coordinate, or `nullptr`/null reference if not resident.

**Data Structures:**
The cache is implemented as a Block Memory Management Unit (MMU) containing the following fields [S2, unofficial]:
1.  `dims`: Vector3i, total volume dimensions.
2.  `brick_dims`: Vector3i, calculated as `ceil(dims / brick_size)`.
3.  `brick_size`: Vector3i, dimensions of each brick in voxels.
4.  `entries`: Array of `Entry` structs, size `brick_dims.x * brick_dims.y * brick_dims.z`.
    *   `Entry` layout:
        *   `valid`: bool, indicates if the entry maps to a valid block.
        *   `blk_idx`: int, index into the `blocks` array.
5.  `blocks`: Array of `Block` structs, size `blk_nums`.
    *   `Block` layout:
        *   `data`: pointer to the voxel data buffer.
        *   `last_access`: timestamp or counter (used for LRU).
6.  `blk_nums`: int, total number of blocks in the pool. Calculated as `brick_dims.x * brick_dims.y * brick_dims.z * _brick_num_ratio`.
7.  `clock_pos`: int, current position in the LRU clock (used for efficient LRU approximation) [S2, unofficial].
8.  `blk_used`: int, number of currently used blocks in the pool.
9.  `data`: `uchar*`, contiguous memory buffer of size `blk_nums * (brick_size.x+1)*(brick_size.y+1)*(brick_size.z+1)` [S2, unofficial].
10. `cnt_buf`: `int*`, count buffer of size `brick_dims.x * brick_dims.y * brick_dims.z * 256`. Used to skip empty bricks based on isovalue ranges [S2, unofficial].
11. `mutex`: `pthread_mutex_t`, for thread-safe access [S2, unofficial].
12. `fin`: `ifstream`, file handle for loading bricks [S2, unofficial].

**Reference Texture (GPU Side):**
A low-resolution floating-point reference texture (e.g., $16^3$ for a $512^3$ volume with $32^3$ bricks) stores texture coordinate offsets to the brick cache [S1]. White/null references indicate bricks not in the cache [S1]. This decouples volume size from GPU resolution limits (e.g., $512^3$) and video memory limits (e.g., 256MB) [S1].

**Numbered Steps:**

1.  **Initialization:**
    *   Calculate `brick_dims` from volume `dims` and `brick_size`.
    *   Allocate `entries` array of size `brick_dims.x * brick_dims.y * brick_dims.z`. Initialize all `valid` to false.
    *   Calculate `blk_nums` using `_brick_num_ratio` (e.g., 0.5) [S2, unofficial].
    *   Allocate `blocks` array of size `blk_nums`.
    *   Allocate `data` buffer of size `blk_nums * (brick_size.x+1)*(brick_size.y+1)*(brick_size.z+1)` [S2, unofficial].
    *   Load `cnt_buf` from disk. Size is `brick_dims.x * brick_dims.y * brick_dims.z * 256` [S2, unofficial].

2.  **Get Block (`getBlock(x, y, z)`):**
    *   **Boundary Check:** If brick coordinates $(x, y, z)$ exceed `brick_dims`, return `nullptr` [S2, unofficial].
    *   **Empty Brick Check (Optional):** If a 1D transfer function `tf1d` is provided:
        *   Calculate `cnt_base = (x * brick_dims.y + y) * brick_dims.z + z`.
        *   Iterate `idx` from 0 to 255:
            *   If `cnt_buf[cnt_base + idx] != 0` AND `tf1d->sampleSimple(idx).w != 0`, break loop.
        *   If `idx == 256`, the brick is empty for the current isovalue; return `nullptr` [S2, unofficial].
    *   **Cache Hit Check:**
        *   Access `entries[x * brick_dims.y * brick_dims.z + y * brick_dims.z + z]`.
        *   If `entry.valid` is true:
            *   Mark block as used (update `last_access` or LRU clock) [S2, unofficial].
            *   Return pointer to `blocks[entry.blk_idx].data` [S2, unofficial].
    *   **Cache Miss / Eviction:**
        *   Lock `mutex` [S2, unofficial].
        *   If `blk_used < blk_nums`:
            *   Assign next free slot in `blocks` array.
        *   Else (Pool Full):
            *   Use `clock_pos` to find the first unused block (LRU-like eviction) [S2, unofficial].
            *   Mark the old entry in `entries` as invalid.
            *   Reuse the slot.
        *   Load block data from file `fin` into the assigned slot in `data` buffer [S2, unofficial].
        *   Mark `entry.valid` to true, store `blk_idx`.
        *   Increment `blk_used` if a new slot was assigned.
        *   Unlock `mutex` [S2, unofficial].
        *   Return pointer to the loaded data.

3.  **Isovalue Change Handling:**
    *   When the isovalue changes, perform a range query on `cnt_buf` to determine the active status of bricks [S1].
    *   Download active bricks into the cache [S1].
    *   Remove inactive bricks using the LRU strategy when space is needed [S1].

**Parameters:**
*   `_brick_num_ratio`: Fraction of total bricks to keep in memory pool (e.g., 0.5 for 50%) [S2, unofficial].
*   `brick_size`: Dimensions of each brick in voxels (e.g., $32^3$) [S2, unofficial].
*   `GPU Resolution Limit`: Example $512^3$ on NVIDIA GeForce 6 [S1].
*   `Memory Limit`: Example 256MB [S1].

**Edge Cases and Pitfalls:**
*   **Precision:** The `cnt_buf` uses 256 bins (8-bit) for isovalue range queries. Precision depends on the isovalue normalization [S2, unofficial].
*   **Boundaries:** Bricks are stored with 1-voxel padding on all sides to support trilinear interpolation at boundaries without accessing adjacent bricks [S1]. The stored voxel count per brick is $V_{brick} = (B_x+1)(B_y+1)(B_z+1)$ [S1].
*   **Concurrency:** The use of `pthread_mutex_t` indicates a CPU-side implementation. GPU-side implementations may require atomic operations or separate LRU buffers [S1].

## Subtopics

### [Block-based Ray Cache](2260-block-based-ray-cache.md)
The Block-based Ray Cache is a memory optimization technique for volume rendering that reduces reconstruction overhead by retaining reconstructed voxel blocks in memory for reuse by adjacent rays. It solves the problem of redundant computation in high-resolution volume rendering where data exceeds physical memory, allowing rendering times to drop from days to minutes. It sets itself apart from sibling methods by explicitly maintaining a sorted list of blocks along the ray depth to facilitate immediate lookup, rather than relying solely on generic LRU eviction without spatial locality awareness.

### [LRU Cache Eviction with Residency Update](3115-lru-cache-eviction-with-residency-update.md)
This method couples LRU brick cache eviction with hierarchical residency octree updates. When the cache lacks a free slot, it evicts the least recently accessed brick and resets bitmask entries in affected leaf and non-leaf nodes to reflect the loss of resident data. It differs from standard LRU caches by synchronizing cache state with octree metadata after each eviction.

### [Brick Cache Implementation](3222-brick-cache-implementation.md)
The Brick Cache is a single-channel 3D texture cache that stores bricks from a multi-resolution multi-volume hierarchy. It is agnostic to the number of data channels in the underlying dataset. It differs from general-purpose CPU caches by being implemented as a GPU-resident texture managed via LRU replacement using WebGPU compute passes.

### [GPU-Steered LRU Cache Management](3654-gpu-steered-lru-cache-management.md)
This method enables the GPU to steer CPU-side data loading for voxel rendering by reporting node usage, thereby eliminating the overhead of CPU-driven visibility queries and tree traversal. It builds on [Brick Caching with LRU Replacement](879-brick-caching-with-lru-replacement.md) by unifying the management of the brick pool and node pool into two LRU-controlled caches where the GPU determines refinement needs and missing data, rather than relying on occlusion queries or frustum containment checks performed on the CPU.

### [LRU Cache Management for Voxel Bricks](4571-lru-cache-management-for-voxel-bricks.md)
LRU Cache Management for Voxel Bricks is a memory management strategy that prioritizes reusing voxel brick data that has not been accessed for the longest time, thereby optimizing memory reuse while retaining useful data in the cache. It solves the problem of limited GPU memory capacity and transfer bandwidth by ensuring that expensive data transfers from slower host memory are minimized. Unlike sibling methods that may rely on hardware-specific features or different replacement policies, this approach explicitly manages the cache lifecycle using a Last Recently Used (LRU) policy, often requiring CPU-side coordination due to the limitations of standard shader models regarding random memory writes.

### [CPU Cache with Write-Back](10163-cpu-cache-with-write-back.md)
CPU DRAM serves as a warm cache between SSD storage and GPU VRAM, maintaining an LRU cache over volume blocks with per-block dirty bits. It decouples GPU residency from SSD write latency by staging evicted blocks in CPU memory before persisting them, turning frequent small updates into batched sequential appends. This method builds on [Brick Caching with LRU Replacement](879-brick-caching-with-lru-replacement.md) by adding a write-back policy that distinguishes between clean and dirty blocks, preventing unnecessary SSD writes for blocks that are still being actively reused.

### Brick Cache Updates
A variant where cache updates are driven by file server uploads. Each frame, a user-defined number of brick data is uploaded to the GPU. Each brick comprises its global brick ID and either the EMPTY value (if all zeros) or the pre-processed brick [S3, unofficial]. The global brick ID is translated to a local brick ID. If the brick's channel is not represented by the multi-channel page-table hierarchy, the brick is discarded [S3, unofficial]. If the brick is EMPTY, the corresponding page in the multi-channel page-table hierarchy is marked as EMPTY. Otherwise, the page is marked as MAPPED, and the brick's data is added to the brick cache using the LRU buffer to find free or recently unused entries [S3, unofficial]. When a brick is removed from the cache, its corresponding page is marked as UNMAPPED. Two lists of local brick IDs are passed to the residency octree: 1) IDs of bricks marked EMPTY or MAPPED, 2) IDs of bricks marked UNMAPPED [S3, unofficial].

### Two-Step Eviction and Write-Back
A variant that decouples GPU residency from SSD write latency using a two-step write-back path: VRAM → CPU → SSD [S4, unofficial]. When a block is evicted from VRAM, it is transferred via D2H and inserted into the CPU cache. Clean blocks are inserted as clean; dirty blocks as dirty [S4, unofficial]. When a dirty block is evicted from the CPU cache, it is asynchronously flushed to SSD patch segments, appending a new version and updating `Index[k]` [S4, unofficial]. On re-admission, blocks are fetched via `Index[k]` to ensure the GPU materializes the most recent version [S4, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
*   **Time:** $O(1)$ average for cache hit. $O(1)$ for eviction search (amortized using clock algorithm). $O(256)$ worst-case for empty brick check (iterating through `cnt_buf`) [S2, unofficial].
*   **Memory:** $O(blk\_nums \times brick\_volume + brick\_count \times 256 \times sizeof(int))$ [S2, unofficial].

**Measured Performance:**
*   Example configuration: $512^3$ volume, $32^3$ bricks, 256MB GPU memory limit [S1].
*   Reference texture size: $16^3$ for the $512^3$ volume [S1].

**Scaling:**
*   The overhead of brick padding is inversely proportional to the brick size [S1].
*   Using two levels of subdivision (small blocks for empty space skipping, large bricks for memory management) helps balance storage overhead and spatial coherence [S1].

**Strengths and Limitations:**
*   **Strengths:** Decouples volume size from GPU resolution and memory limits [S1]. Efficiently skips empty bricks using `cnt_buf` [S2, unofficial]. Simple LRU implementation using clock algorithm [S2, unofficial].
*   **Limitations:** Requires CPU-side coordination for LRU updates in standard shader models [S1]. Mutex contention may occur in multi-threaded CPU implementations [S2, unofficial].

**When to Use:**
*   Use when GPU memory is limited and volume data exceeds VRAM capacity [S1].
*   Use when isovalue changes frequently, requiring dynamic active brick determination [S1].

**When Not to Use:**
*   Not suitable when hardware-specific features (like hardware LRU caches) are available and sufficient.
*   Not suitable when strict real-time performance is required and CPU-GPU synchronization overhead is prohibitive.

## Background
**Origin:**
The method originates from real-time ray-casting of discrete isosurfaces, specifically addressing the need to manage large volumes on GPUs with limited memory [S1]. The concept of using a low-resolution reference texture to map to a brick cache was introduced in [KE02] (referenced in [S1]).

**History and Development:**
*   Early implementations used a fixed-size pool with LRU eviction to manage brick residency [S2, unofficial].
*   Variants have been developed to integrate with residency octrees [S1], multi-channel page-table hierarchies [S3, unofficial], and out-of-core optimization for large-scale training (e.g., 3D Gaussian Splatting) [S4, unofficial].

**Where it is used:**
*   Real-time volume rendering of large biomedical datasets [S3, unofficial].
*   Out-of-core training of large-scale 3D Gaussian Splatting primitives [S4, unofficial].
*   Web-based volume rendering systems [S3, unofficial].

## Sources
[S1] Real-Time Ray-Casting and Advanced Shading of Discrete Isosurfaces, https://cgl.ethz.ch/people/archive/siggc/publications/eg05.pdf, fetched 2026-10-26
[S2] hopexn/VolumeRenderingParallel, https://github.com/hopexn/VolumeRenderingParallel, fetched 2026-10-27 (unofficial)
[S3] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
[S4] TideGS: Scalable Training of Over One Billion 3D Gaussian Splatting Primitives via Out-of-Core Optimization, https://doi.org/10.48550/arxiv.2605.20150, fetched 2026-10-26 (unofficial)
