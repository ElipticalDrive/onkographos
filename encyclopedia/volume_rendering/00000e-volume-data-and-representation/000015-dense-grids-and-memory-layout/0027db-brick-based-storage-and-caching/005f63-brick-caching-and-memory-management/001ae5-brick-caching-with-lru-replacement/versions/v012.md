# Brick Caching with LRU Replacement

Also called: Block Memory Management Unit (MMU) with LRU Eviction, Brick Pool Manager, Dynamic Brick Cache, Volume Block Cache

## Overview
Brick Caching with LRU Replacement manages a fixed-size pool of volume data blocks, mapping logical brick coordinates to physical memory locations and evicting the least-recently-used blocks when the pool is full [S2, unofficial]. It solves the problem of limited GPU memory capacity and slow data transfer rates by ensuring that the most recently accessed brick data remains in the fast working set [S4, unofficial]. It distinguishes itself from standard spatial traversal by decoupling the processing order of blocks from the ray order within a frame, relying on a sorted auxiliary list structure to ensure cache hits for spatially coherent rays [S1]. It builds on [Brick Caching and Memory Management](17878-brick-caching-and-memory-management.md) by implementing a specific eviction policy (LRU) and often integrating with a reference texture for coordinate translation [S1].

## How it works
The method operates on a fixed-size pool of memory blocks. The core logic involves a mapping from logical brick coordinates $(bx, by, bz)$ to pool indices, using a Least Recently Used (LRU) policy to determine which brick to evict when the pool is full [S2, unofficial].

### Inputs and Outputs
*   **Inputs**:
    *   Volume dimensions: $D_x, D_y, D_z$ [S2, unofficial].
    *   Brick dimensions: $B_x, B_y, B_z$ [S2, unofficial].
    *   Pool capacity ratio: $\alpha$ (fraction of total bricks to keep in memory, e.g., 0.5) [S2, unofficial].
    *   Transfer function (optional): $TF$ for empty brick detection [S2, unofficial].
    *   Requested brick coordinates: $(bx, by, bz)$ [S2, unofficial].
*   **Outputs**:
    *   Pointer to brick data in the pool, or `nullptr` if the brick is empty or not resident [S2, unofficial].

### Data Structures
The implementation requires the following structures [S2, unofficial]:

1.  **Grid Dimensions**:
    *   `dims`: $(D_x, D_y, D_z)$
    *   `brick_dims`: $(\lceil D_x/B_x \rceil, \lceil D_y/B_y \rceil, \lceil D_z/B_z \rceil)$
2.  **Pool Arrays**:
    *   `entries`: Array of size $N_{bricks} = brick\_dims.x \times brick\_dims.y \times brick\_dims.z$. Each element is an `Entry` struct:
        ```python
        struct Entry:
            valid: bool
            blk_idx: int
        ```
    *   `blocks`: Array of size $N_{blocks} = \lceil N_{bricks} \times \alpha \rceil$. Each element is a `Block` struct:
        ```python
        struct Block:
            used: bool
            last_access_time: int # Or timestamp/counter
        ```
    *   `data`: Byte buffer of size $N_{blocks} \times (B_x+1)(B_y+1)(B_z+1) \times sizeof(uchar)$. Bricks are stored with $(n+1)^3$ dimensions to include neighbor samples for filtering [S1].
3.  **Metadata**:
    *   `cnt_buf`: Count buffer of size $N_{bricks} \times 256$. Used to skip empty bricks based on transfer function samples [S2, unofficial].
    *   `clock_pos`: Integer index for the clock algorithm (LRU approximation) [S2, unofficial].
    *   `blk_used`: Integer count of currently used blocks in the pool [S2, unofficial].
    *   `mutex`: Synchronization primitive for thread safety [S2, unofficial].
    *   `fin`: File handle for loading bricks from disk [S2, unofficial].

### Algorithm Steps

1.  **Initialization**:
    *   Calculate `brick_dims` from volume and brick sizes [S2, unofficial].
    *   Allocate `entries` array of size `brick_dims.x * brick_dims.y * brick_dims.z` [S2, unofficial].
    *   Calculate $N_{blocks} = \lceil brick\_dims.x \times brick\_dims.y \times brick\_dims.z \times \alpha \rceil$ [S2, unofficial].
    *   Allocate `blocks` array of size $N_{blocks}$ [S2, unofficial].
    *   Allocate `data` buffer of size $N_{blocks} \times (B_x+1)(B_y+1)(B_z+1)$ [S2, unofficial].
    *   Load `cnt_buf` from disk/file [S2, unofficial].
    *   Initialize `clock_pos = 0`, `blk_used = 0` [S2, unofficial].

2.  **Request Brick `getBlock(bx, by, bz)`**:
    *   **Boundary Check**: If $bx \ge brick\_dims.x$ or $by \ge brick\_dims.y$ or $bz \ge brick\_dims.z$, return `nullptr` [S2, unofficial].
    *   **Empty Brick Check (Optimization)**:
        *   If `tf1d` (transfer function) is provided:
            *   Calculate `cnt_base = (bx * brick_dims.y + by) * brick_dims.z + bz` [S2, unofficial].
            *   Iterate $idx$ from 0 to 255:
                *   If `cnt_buf[cnt_base + idx] != 0` AND `tf1d.sampleSimple(idx).w != 0`:
                    *   Break loop (brick is potentially non-empty) [S2, unofficial].
            *   If loop completes without break ($idx == 256$), return `nullptr` (brick is empty) [S2, unofficial].
    *   **Cache Hit Check**:
        *   Access `entry = entries[bx * brick_dims.y * brick_dims.z + by * brick_dims.z + bz]` [S2, unofficial].
        *   If `entry.valid`:
            *   Mark `blocks[entry.blk_idx].used = true` [S2, unofficial].
            *   Update `blocks[entry.blk_idx].last_access_time` [G].
            *   Return pointer to `data` offset by `entry.blk_idx * brick_volume` [S2, unofficial].
    *   **Cache Miss / Eviction**:
        *   Lock `mutex` [S2, unofficial].
        *   If `blk_used < N_{blocks}`:
            *   Find first free slot in `blocks` (where `used == false`) [G].
            *   Let `blk_idx` be the index of this slot.
        *   Else:
            *   Perform LRU eviction search starting from `clock_pos`:
                *   While `blocks[clock_pos].used == true`:
                    *   `clock_pos = (clock_pos + 1) % N_{blocks}` [S2, unofficial].
                *   Let `old_blk_idx = clock_pos` [S2, unofficial].
                *   Mark `blocks[old_blk_idx].used = false` [S2, unofficial].
                *   Invalidate corresponding entry: `entries[...] = {valid: false, blk_idx: -1}` [S2, unofficial].
                *   Let `blk_idx = old_blk_idx` [S2, unofficial].
        *   **Load Brick**:
            *   Read brick data from `fin` into `data` at offset `blk_idx * brick_volume` [S2, unofficial].
        *   **Update State**:
            *   Set `entry.valid = true`, `entry.blk_idx = blk_idx` [S2, unofficial].
            *   Set `blocks[blk_idx].used = true` [S2, unofficial].
            *   Update `blocks[blk_idx].last_access_time` [G].
            *   Increment `blk_used` if a new slot was allocated (not reused) [G].
            *   Unlock `mutex` [S2, unofficial].
        *   Return pointer to `data` at `blk_idx` [S2, unofficial].

### Parameters
*   **Brick Size**: Typically $32^3$ voxels [S1]. Larger bricks reduce storage overhead but increase granularity of empty space skipping [S1].
*   **Pool Ratio ($\alpha$)**: Fraction of total bricks to keep in memory. Example: 0.5 (50%) [S2, unofficial].
*   **GPU Resolution Limit**: Historically constrained by hardware (e.g., $512^3$ on NVIDIA GeForce 6) [S1].
*   **Memory Limit**: Example 256MB [S1].

### Edge Cases and Pitfalls
*   **Precision**: The count buffer `cnt_buf` uses 256 entries per brick, corresponding to the 256 possible transfer function sample values [S2, unofficial].
*   **Boundaries**: Brick coordinates must be checked against `brick_dims` to avoid out-of-bounds access [S2, unofficial].
*   **Concurrency**: The use of a mutex indicates this implementation is thread-safe but may suffer from contention under high load [S2, unofficial].
*   **Empty Detection**: The empty brick check relies on the transfer function's alpha channel (`w != 0`). If the TF is not provided, this optimization is skipped [S2, unofficial].

## Subtopics

### [Block-based Ray Cache](2260-block-based-ray-cache.md)
Caches reconstructed voxel blocks for reuse by adjacent rays, decoupling block processing order from ray order.

### [LRU Cache Eviction with Residency Update](3115-lru-cache-eviction-with-residency-update.md)
Couples LRU eviction with hierarchical octree residency bitmask updates.

### [Brick Cache Implementation](3222-brick-cache-implementation.md)
GPU-resident 3D texture cache managed via WebGPU compute passes, agnostic to data channels.

### [GPU-Steered LRU Cache Management](3654-gpu-steered-lru-cache-management.md)
GPU dictates brick/node pool states based on traversal history, reducing CPU overhead.

### [LRU Cache Management for Voxel Bricks](4571-lru-cache-management-for-voxel-bricks.md)
CPU-managed LRU cache for voxel bricks, bypassing shader memory write limitations.

### [CPU Cache with Write-Back](10163-cpu-cache-with-write-back.md)
Adds a write-back policy to the brick cache, staging evicted blocks in CPU memory before SSD persistence.

### Brick Cache Updates
This variant manages cache updates in a multi-channel page-table hierarchy [S3, unofficial].
1.  **Upload**: Upload a user-defined number of bricks from the file server to the GPU each frame [S3, unofficial].
2.  **Translate ID**: Translate global brick ID to local brick ID [S3, unofficial].
3.  **Discard**: If the brick's channel is not in the multi-channel page-table hierarchy, discard the brick [S3, unofficial].
4.  **Handle EMPTY**: If the brick is EMPTY (all zeros), mark the corresponding page in the hierarchy as EMPTY [S3, unofficial].
5.  **Handle MAPPED**: Otherwise, mark the page as MAPPED and add the brick's data to the brick cache [S3, unofficial].
6.  **Find Entry**: Use the LRU buffer to find free or least-recently-used cache entries [S3, unofficial].
7.  **Handle Removal**: When a brick is removed from the cache, mark its corresponding page as UNMAPPED [S3, unofficial].
8.  **Pass Lists**: Pass two lists of local brick IDs to the residency octree: 1) IDs of bricks marked EMPTY or MAPPED, 2) IDs of bricks marked UNMAPPED [S3, unofficial].

### Two-Step Eviction and Write-Back
This variant decouples GPU residency from SSD write latency using a two-step write-back path [S4, unofficial].
1.  **VRAM Eviction**: When a block is evicted from VRAM, transfer it via D2H (Device-to-Host) and insert it into the CPU cache [S4, unofficial].
2.  **Clean/Dirty Tracking**: Clean blocks are inserted as clean; dirty blocks are inserted as dirty [S4, unofficial].
3.  **CPU Eviction**: When a dirty block is evicted from the CPU cache, asynchronously flush it to SSD patch segments, append a new version, and update `Index[k]` [S4, unofficial].
4.  **Re-admission**: On re-admission, blocks are fetched via `Index[k]` to ensure the GPU materializes the most recent version [S4, unofficial].

## Operating characteristics
*   **Time Complexity**:
    *   Cache Hit: $O(1)$ average [S2, unofficial].
    *   Eviction Search: $O(1)$ amortized (using clock algorithm) [S2, unofficial].
    *   Empty Brick Check: $O(256)$ worst-case (iterating count buffer) [S2, unofficial].
*   **Memory Complexity**: $O(N_{blocks} \times V_{brick} + N_{bricks} \times 256 \times sizeof(int))$, where $V_{brick}$ is the volume of a brick including padding [S2, unofficial].
*   **Performance**: The method allows rendering volumes larger than physical GPU memory (e.g., $512^3$ on 256MB VRAM) by dynamically loading bricks [S1].
*   **Strengths**:
    *   Reduces effective memory footprint by skipping empty bricks [S1].
    *   Handles neighbor samples for filtering without complicated look-ups at boundaries [S1].
    *   Decouples volume size from GPU resolution and memory restrictions [S1].
*   **Limitations**:
    *   Overhead of $(n+1)^3$ storage is inversely proportional to brick size [S1].
    *   Mutex contention may limit scalability in multi-threaded environments [S2, unofficial].
    *   LRU approximation (clock) may not be optimal for all access patterns compared to full LRU [S2, unofficial].
*   **When to Use**: Suitable for out-of-core volume rendering where data exceeds GPU memory and access patterns exhibit spatial locality [S1][S4, unofficial].
*   **When Not to Use**: When memory is sufficient to hold the entire volume, or when access patterns are random and LRU provides no benefit [S1].

## Background
The method originates from real-time ray-casting techniques for discrete isosurfaces [S1]. It was developed to address the limitations of GPU memory capacity and resolution restrictions (e.g., NVIDIA GeForce 6) [S1]. The approach adapts the concept of adaptive texture maps [KE02] to volume data, using a low-resolution reference texture to map volume coordinates to a single brick cache texture resident in GPU memory [S1]. The implementation details in [S2, unofficial] reflect a C++/Qt-based parallel volume rendering system, while [S3, unofficial] and [S4, unofficial] extend the concept to web-based and out-of-core training contexts respectively.

## Sources
[S1] Real-Time Ray-Casting and Advanced Shading of Discrete Isosurfaces, https://cgl.ethz.ch/people/archive/siggc/publications/eg05.pdf, fetched 2026-10-26
[S2] hopexn/VolumeRenderingParallel, https://github.com/hopexn/VolumeRenderingParallel, fetched 2026-10-27 (unofficial)
[S3] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
[S4] TideGS: Scalable Training of Over One Billion 3D Gaussian Splatting Primitives via Out-of-Core Optimization, https://doi.org/10.48550/arxiv.2605.20150, fetched 2026-10-26 (unofficial)
