# Brick Caching with LRU Replacement

Also called: Block Memory Management Unit (MMU) with LRU Eviction, Brick Pool Manager, Dynamic Brick Cache, Volume Block Cache

## Overview
Brick Caching with LRU Replacement manages a fixed-size pool of volume data blocks, mapping brick coordinates to pool indices and evicting least-recently-used blocks when the pool is full [S2, unofficial]. It solves the problem of limited GPU memory capacity and transfer bandwidth by ensuring that expensive data transfers from slower host memory are minimized [S1]. Unlike general-purpose CPU caches, this approach explicitly manages the cache lifecycle using a Last Recently Used (LRU) policy, often requiring CPU-side coordination due to the limitations of standard shader models regarding random memory writes [S1]. It differs from [Brick-based Volume Raycasting](3341-brick-based-volume-raycasting.md) by decoupling the volume size from GPU resolution restrictions through a re-arranged brick structure and a resident cache texture [S1].

## How it works
**Inputs and Outputs:**
- **Input:** Volume dimensions, brick size, brick pool ratio, file handle to volume data, optional transfer function (for active brick determination).
- **Output:** A block pointer (or `nullptr` if empty/not resident) for a given 3D brick coordinate $(x, y, z)$.

**Data Structures:**
The cache is managed by a Block Memory Management Unit (MMU) containing the following fields [S2, unofficial]:
1.  `dims`: Vector3i, total volume dimensions.
2.  `brick_dims`: Vector3i, number of bricks in each dimension, calculated as $\lceil \text{dims} / \text{brick\_size} \rceil$.
3.  `brick_size`: Vector3i, dimensions of each brick in voxels.
4.  `entries`: Array of `Entry` structures, size `brick_dims.x * brick_dims.y * brick_dims.z`.
    -   `Entry` layout: `{ bool valid; int blk_idx; }`.
    -   `valid`: Indicates if the entry maps to a resident block.
    -   `blk_idx`: Index into the `blocks` array.
5.  `blocks`: Array of `Block` structures, size `blk_nums`.
    -   `blk_nums`: Total number of blocks in the pool, calculated as `brick_dims.x * brick_dims.y * brick_dims.z * _brick_num_ratio`.
6.  `data`: `uchar*`, contiguous memory pool of size `blk_nums * (brick_size.x+1) * (brick_size.y+1) * (brick_size.z+1)`. Bricks are stored with size $(n+1)^3$ to include neighbor samples for filtering [S1].
7.  `cnt_buf`: `int*`, count buffer of size `brick_dims.x * brick_dims.y * brick_dims.z * 256`. Used to skip empty bricks. Loaded from disk [S2, unofficial].
8.  `clock_pos`: `int`, position in the LRU clock for eviction [S2, unofficial].
9.  `blk_used`: `int`, number of currently used blocks in the pool [S2, unofficial].
10. `mutex`: `pthread_mutex_t`, for thread-safe access [S2, unofficial].
11. `fin`: `ifstream`, file handle for loading bricks [S2, unofficial].

**Parameters:**
-   `_brick_num_ratio`: Fraction of total bricks to keep in memory (e.g., 0.5 for 50%) [S2, unofficial].
-   `brick_size`: Dimensions of each brick in voxels. Small blocks fit the isosurface tightly for empty space skipping, while larger bricks avoid excessive storage overhead [S1].
-   GPU Resolution Limit: Example 512³ on NVIDIA GeForce 6 [S1].
-   Memory Limit: Example 256MB [S1].

**Algorithm Steps:**

1.  **Initialization:**
    -   Calculate `brick_dims` from volume `dims` and `brick_size`.
    -   Allocate `entries` array of size `brick_dims.x * brick_dims.y * brick_dims.z`.
    -   Calculate `blk_nums` using `_brick_num_ratio`.
    -   Allocate `blocks` array of size `blk_nums`.
    -   Allocate `data` buffer of size `blk_nums * (brick_size.x+1) * (brick_size.y+1) * (brick_size.z+1)`.
    -   Load `cnt_buf` of size `brick_dims.x * brick_dims.y * brick_dims.z * 256` from disk [S2, unofficial].
    -   Initialize `clock_pos = 0`, `blk_used = 0`, and all `entries` as invalid [S2, unofficial].

2.  **Get Block (x, y, z):**
    -   **Boundary Check:** If $x \ge \text{brick\_dims}.x$ or $y \ge \text{brick\_dims}.y$ or $z \ge \text{brick\_dims}.z$, return `nullptr` [S2, unofficial].
    -   **Empty Brick Check:** If a transfer function `tf1d` is provided, check `cnt_buf`. Let `cnt_base = (x * brick_dims.y + y) * brick_dims.z + z`. Iterate `idx` from 0 to 255:
        -   If `cnt_buf[cnt_base + idx] != 0` and `tf1d->sampleSimple(idx).w != 0`, break loop.
        -   If `idx == 256`, return `nullptr` (brick is empty) [S2, unofficial].
    -   **Cache Hit:** Check `entries` at index $(x, y, z)$. If `valid` is true:
        -   Mark block as used (update LRU timestamp/position) [S2, unofficial].
        -   Return pointer to block data at `blocks[entries[idx].blk_idx].data` [S2, unofficial].
    -   **Cache Miss / Eviction:**
        -   Lock `mutex` [S2, unofficial].
        -   If `blk_used < blk_nums`:
            -   Assign next free slot (find first invalid entry in `blocks` or maintain a free list).
        -   Else (pool full):
            -   Use `clock_pos` to find the first unused block (LRU-like eviction) [S2, unofficial].
            -   Mark the old entry in `entries` as invalid.
            -   Update `clock_pos` to the next position [S2, unofficial].
        -   Load block data from file into the assigned slot in `data` [S2, unofficial].
        -   Mark `entries` at $(x, y, z)$ as valid, store `blk_idx`.
        -   Increment `blk_used` if a new slot was assigned.
        -   Unlock `mutex` [S2, unofficial].
        -   Return pointer to the loaded block data [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class BrickCache:
    def __init__(self, dims, brick_size, brick_num_ratio, file_path):
        self.brick_size = brick_size
        self.brick_dims = Vector3i(
            ceil(dims.x / brick_size.x),
            ceil(dims.y / brick_size.y),
            ceil(dims.z / brick_size.z)
        )
        self.total_bricks = self.brick_dims.x * self.brick_dims.y * self.brick_dims.z
        self.blk_nums = int(self.total_bricks * brick_num_ratio)
        
        # Data structures
        self.entries = [Entry(valid=False, blk_idx=-1) for _ in range(self.total_bricks)]
        self.blocks = [Block() for _ in range(self.blk_nums)]
        self.data = bytearray(self.blk_nums * (brick_size.x+1) * (brick_size.y+1) * (brick_size.z+1))
        self.cnt_buf = load_cnt_buf_from_disk(self.brick_dims) # Size: total_bricks * 256
        self.blk_used = 0
        self.clock_pos = 0
        self.file = open(file_path, 'rb')
        self.mutex = Mutex()

    def get_block(self, x, y, z, tf1d=None):
        # Boundary check
        if x >= self.brick_dims.x or y >= self.brick_dims.y or z >= self.brick_dims.z:
            return None

        # Empty brick check
        if tf1d is not None:
            cnt_base = (x * self.brick_dims.y + y) * self.brick_dims.z + z
            is_empty = True
            for idx in range(256):
                if self.cnt_buf[cnt_base + idx] != 0 and tf1d.sample_simple(idx).w != 0:
                    is_empty = False
                    break
            if is_empty:
                return None

        # Calculate entry index
        entry_idx = (x * self.brick_dims.y + y) * self.brick_dims.z + z
        entry = self.entries[entry_idx]

        # Cache hit
        if entry.valid:
            self.update_lru(entry.blk_idx)
            return self.get_block_data(entry.blk_idx)

        # Cache miss / Eviction
        self.mutex.lock()
        try:
            if self.blk_used < self.blk_nums:
                # Find free slot
                blk_idx = self.find_free_slot()
                self.blk_used += 1
            else:
                # LRU Eviction
                blk_idx = self.clock_pos
                old_entry_idx = self.blocks[blk_idx].entry_idx # Assuming Block tracks its entry
                if old_entry_idx != -1:
                    self.entries[old_entry_idx].valid = False
                self.clock_pos = (self.clock_pos + 1) % self.blk_nums
            
            # Load block
            self.load_block_from_file(blk_idx, x, y, z)
            
            # Update entry
            self.entries[entry_idx].valid = True
            self.entries[entry_idx].blk_idx = blk_idx
            self.blocks[blk_idx].entry_idx = entry_idx
        finally:
            self.mutex.unlock()

        return self.get_block_data(blk_idx)

    def update_lru(self, blk_idx):
        # Update timestamp or move to front of LRU list
        pass

    def find_free_slot(self):
        # Find first block with valid=False or unused
        pass

    def load_block_from_file(self, blk_idx, x, y, z):
        # Read from self.file into self.data at offset corresponding to blk_idx
        pass

    def get_block_data(self, blk_idx):
        # Return pointer/offset to data for blk_idx
        pass
```

**Edge Cases and Pitfalls:**
-   **Precision:** Brick coordinates must be correctly mapped to entry indices to avoid out-of-bounds access [S2, unofficial].
-   **Boundaries:** Bricks at the volume boundary may have incomplete neighbor samples if the volume size is not a multiple of brick size. The $(n+1)^3$ storage helps with filtering but requires careful boundary handling during reconstruction [S1].
-   **Concurrency:** The mutex ensures thread safety but may become a bottleneck. The LRU clock position must be updated atomically or protected by the mutex [S2, unofficial].
-   **Empty Bricks:** The `cnt_buf` check is critical for performance. If not used, the cache may fill with empty bricks, wasting memory [S2, unofficial].

## Subtopics

### [Brick Cache Implementation](3222-brick-cache-implementation.md)
The Brick Cache is a single-channel 3D texture cache that stores bricks from a multi-resolution multi-volume hierarchy. It is agnostic to the number of data channels. It differs from general-purpose CPU caches by being implemented as a GPU-resident texture managed via LRU replacement using WebGPU compute passes. It uses a low-resolution floating-point reference texture (e.g., 16³ for a 512³ volume with 32³ bricks) to store texture coordinate offsets to the brick cache, allowing the fragment shader to transform volume coordinates into brick cache texture coordinates. White/null references indicate bricks not in the cache.

### [LRU Cache Management for Voxel Bricks](4571-lru-cache-management-for-voxel-bricks.md)
LRU Cache Management for Voxel Bricks is a memory management strategy that prioritizes reusing voxel brick data that has not been accessed for the longest time, thereby optimizing memory reuse while retaining useful data in the cache. It solves the problem of limited GPU memory capacity and transfer bandwidth by ensuring that expensive data transfers from slower host memory are minimized. Unlike sibling methods that may rely on hardware-specific features or different replacement policies, this approach explicitly manages the cache lifecycle using a Last Recently Used (LRU) policy, often requiring CPU-side coordination due to the limitations of standard shader models regarding random memory writes.

### Block-based Ray Cache
After a block is reconstructed, it is buffered in cache memory instead of being discarded [S3, unofficial]. To access a voxel, the cache is searched using a buffered-block list; if the voxel is within a cached block, it is retrieved immediately, otherwise reconstruction is invoked [S3, unofficial]. The replacement policy substitutes the oldest block with the new block when the cache is full [S3, unofficial]. The cache size requirement in 3D for image resolution $N \times N$ and block size $s \times s \times s$ is $2 \times s \times N \times N$ (two layers of sub-volumes) [S3, unofficial]. For $N=256$, cache size is 0.5 MB (S=16) or 1 MB (S=32); for $N=1024$, it is 2 MB (S=16) or 4 MB (S=32) [S3, unofficial]. Cache size grows at rate $2^2$ while original data grows at $2^3$, making caching efficient for high-resolution data [S3, unofficial].

### LRU Cache Eviction with Residency Update
If no cache slot is available, the least recently accessed brick is evicted using an LRU scheme [S4]. After eviction, all leaf nodes whose spatial extent overlaps the evicted brick's spatial extent are checked to see if they are still partially mapped (i.e., if any other brick in the same resolution level is resident) [S4]. If no other brick in that resolution level is resident, the corresponding bit in the node's bitmask is reset [S4]. Non-leaf nodes are updated by setting their bitmasks to the bitwise OR of their child nodes' bitmasks [S4].

### GPU-Steered LRU Cache Management
A method where the GPU steers data loading by indicating which nodes were helpful, using spatial and temporal coherence to update an LRU cache on the CPU without significant overhead [S5, unofficial]. Nodes and bricks are subdivided or fused in an LRU manner, with timestamps reset upon usage [S5, unofficial]. The CPU updates timestamps of pool elements in its mirrored data structure based on GPU feedback and triggers loading of needed data [S5, unofficial]. Three MRTs are used to store node usage information, allowing 12 nodes per pixel (3 x 4) [S5, unofficial].

### Brick Cache Updates
A user-defined number of brick data is uploaded from the file server to the GPU each frame [S6, unofficial]. Each brick comprises its global brick ID and either the EMPTY value (if all zeros) or the pre-processed brick [S6, unofficial]. The global brick ID is translated to a local brick ID [S6, unofficial]. If the brick's channel is not represented by the multi-channel page-table hierarchy, the brick is discarded [S6, unofficial]. If EMPTY, the corresponding page is marked EMPTY; otherwise, it is marked MAPPED and added to the brick cache using the LRU buffer to find free or unused entries [S6, unofficial]. When a brick is removed, its page is marked UNMAPPED [S6, unofficial]. Two lists of local brick IDs (EMPTY/MAPPED and UNMAPPED) are passed to the residency octree [S6, unofficial].

## Operating characteristics
-   **Time Complexity:** O(1) average for cache hit, O(1) for eviction search (amortized), O(256) worst-case for empty brick check [S2, unofficial].
-   **Memory Complexity:** O(blk_nums × brick_volume + brick_count × 256 × sizeof(int)) [S2, unofficial].
-   **Performance:** The method allows rendering of volumes larger than GPU memory (e.g., 512³ on 256MB VRAM) by streaming bricks [S1].
-   **Strengths:** Efficient memory usage for large volumes; effective empty space skipping via `cnt_buf`; decouples volume size from GPU constraints [S1][S2, unofficial].
-   **Limitations:** Requires CPU-side coordination for LRU updates in some implementations; mutex contention may limit parallelism; overhead of $(n+1)^3$ storage for bricks [S1][S2, unofficial].
-   **When to use:** When rendering large volumes that exceed GPU memory, especially with dynamic isovalue changes requiring active brick determination [S1].
-   **When not to use:** For small volumes that fit entirely in GPU memory, where a simple resident texture is more efficient [S1].

## Background
The concept of brick-based caching with LRU replacement originated in the context of real-time ray-casting of discrete isosurfaces, where decoupling volume size from GPU resolution restrictions was necessary [S1]. The specific implementation details, including the MMU structure and `cnt_buf` optimization, are documented in open-source projects and academic works on parallel volume rendering [S2, S5, S6, unofficial]. The integration with residency octrees and GPU-steered management further evolved the technique to handle multi-resolution hierarchies and web-based rendering scenarios [S4][S5, S6, unofficial].

## Sources
[S1] Real-Time Ray-Casting and Advanced Shading of Discrete Isosurfaces, https://cgl.ethz.ch/people/archive/siggc/publications/eg05.pdf, fetched 2026-10-26
[S2] hopexn/VolumeRenderingParallel, https://github.com/hopexn/VolumeRenderingParallel, fetched 2026-10-27 (unofficial)
[S3] Direct rendering of multiresolution decomposed volume data., https://doi.org/10.82465/4870, fetched 2026-10-26 (unofficial)
[S4] Residency Octree: A Hybrid Approach for Scalable Web ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC10840607/, fetched 2026-10-29
[S5] Ray-Guided Streaming for Efficient and Detailed Voxel Rendering, https://www-sop.inria.fr/reves/Basilic/2009/CNLE09/CNLE09.pdf, fetched 2026-10-26 (unofficial)
[S6] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
