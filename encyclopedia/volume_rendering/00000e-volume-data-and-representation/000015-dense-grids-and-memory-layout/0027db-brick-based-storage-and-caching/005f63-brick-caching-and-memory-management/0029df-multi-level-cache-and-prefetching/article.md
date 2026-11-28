# Multi-Level Cache and Prefetching

Also called: HTTP Range Request Caching, Luxar Viewer Cache Architecture

## Overview
Multi-Level Cache and Prefetching is a memory management strategy for interactive volume rendering that decouples data retrieval from the rendering loop using a hierarchical cache structure and background prefetching [S1, unofficial]. It solves the problem of high-latency remote data fetching (order of 100 ms) by ensuring that decompressed geometry and compressed chunks are available in local memory before they are needed [S1, unofficial]. It differs from sibling methods like [Brick Caching with LRU Replacement](879-brick-caching-with-lru-replacement.md) by explicitly managing four distinct tiers of storage (disk, compressed memory, decompressed memory, and slice ladders) with heap-aware dynamic sizing, rather than a single fixed pool [S1, unofficial]. It also differs from [Raymarching with LOD-based Brick Caching](5776-raymarching-with-lod-based-brick-caching.md) by focusing on chunk-level and slice-level locality for splat-based rendering rather than voxel block LODs [S1, unofficial].

## How it works
The method operates on a four-level cache hierarchy that stores data fetched via HTTP range requests from Zarr archives or similar chunked storage formats [S1, unofficial]. The system employs a background idle-time prefetcher to hide remote-fetch latency [S1, unofficial].

**Inputs and Outputs:**
*   **Input:** A sequence of view rays or slice planes requested by the user (orbit, slider playback) [S1, unofficial].
*   **Output:** Rendered pixels or updated slice views, with cache state updated to reflect accessed data [S1, unofficial].

**Data Structures:**
The cache consists of four tiers [S1, unofficial]: L2 (Disk Cache) is stored in the browser’s Origin Private File System (OPFS), has a fixed size of 2 GB, stores compressed chunks, and survives browser restarts [S1, unofficial]. L1 (Compressed Memory) stores compressed chunks in RAM with a 100 MB ceiling [S1, unofficial]. L0 (Decompressed Memory) stores decompressed chunks in RAM with a 200 MB ceiling [S1, unofficial]. S-cache (Slice Cache) stores fully decoded slice ladders, has a 128 MB default size that grows up to 2 GB, and allows revisited slices to skip the chunk path entirely [S1, unofficial].

**Heap Awareness:**
The in-memory tiers (L0, L1, S-cache) are heap-aware [S1, unofficial]. When the available heap memory is tight, L0 and L1 shrink from their configured ceilings, while the S-cache grows with the heap from its 128 MB default up to 2 GB [S1, unofficial].

**Numbered Steps:**
1.  **Request:** The renderer requests a data chunk or slice [S1, unofficial].
2.  **L0 Check:** Check if the decompressed chunk is in L0; if yes, use it [S1, unofficial].
3.  **L1 Check:** If not in L0, check if the compressed chunk is in L1; if yes, decompress it using WASM-accelerated Blosc/zstd codecs and place in L0, evicting from L1 if necessary [S1, unofficial].
4.  **L2 Check:** If not in L1, check if the compressed chunk is in L2 (OPFS); if yes, load into L1, then decompress to L0 [S1, unofficial].
5.  **Remote Fetch:** If not in L2, issue an HTTP range request to fetch the chunk; upon arrival, store in L2, then L1, then L0 [S1, unofficial].
6.  **S-cache Optimization:** If the request is for a full slice, check the S-cache; if the fully decoded slice ladder is present, use it directly, bypassing chunk-level fetching [S1, unofficial].
7.  **Prefetching:** Immediate prefetching requests the chunk on either side in the ordering after an on-disk hit or remote fetch [S1, unofficial]. Idle-time prefetching warms the adjacent slice to hide remote-fetch latency (order of 100 ms) during user orbit or slider playback [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Cache tiers: L0 (decompressed), L1 (compressed), L2 (disk/OPFS), S_CACHE (slice ladders)
L0 = {}  # max 200 MB
L1 = {}  # max 100 MB
L2 = {}  # fixed 2 GB
S_CACHE = {} # max 2 GB, default 128 MB

def decompress_chunk(compressed_bytes):
    # Uses WASM-accelerated Blosc/zstd codecs [S1, unofficial]
    # Exact codec parameters not given in sources [G]
    return decompress_blosc_zstd(compressed_bytes)

def load_from_opfs(chunk_id):
    # Reads compressed chunk from browser's Origin Private File System [S1, unofficial]
    return read_file_from_opfs(chunk_id)

def fetch_remote(chunk_id):
    # Issues HTTP range request to fetch chunk [S1, unofficial]
    return http_get_range_request(chunk_id)

def async_fetch(chunk_id):
    # Background thread fetches chunk without blocking renderer [S1, unofficial]
    threading.Thread(target=fetch_and_cache, args=(chunk_id,)).start()

def get_data(request):
    chunk_id = request.chunk_id
    slice_id = request.slice_id

    # 6. S-cache Optimization: If request is for a full slice, check S-cache
    if slice_id is not None and slice_id in S_CACHE:
        return S_CACHE[slice_id]

    # 1. Check L0 (Decompressed)
    if chunk_id in L0:
        return L0[chunk_id]

    # 2. Check L1 (Compressed)
    if chunk_id in L1:
        decompressed = decompress_chunk(L1[chunk_id])
        L0[chunk_id] = decompressed
        manage_heap_pressure()
        return decompressed

    # 3. Check L2 (Disk/OPFS)
    if chunk_id in L2:
        compressed = load_from_opfs(chunk_id)
        L1[chunk_id] = compressed
        return get_data(request) # Retry to L1 -> L0

    # 4. Remote Fetch
    compressed = fetch_remote(chunk_id)
    L2[chunk_id] = compressed
    L1[chunk_id] = compressed
    decompressed = decompress_chunk(compressed)
    L0[chunk_id] = decompressed

    # 7. Prefetching: Immediate
    prefetch_neighbors(chunk_id)

    return decompressed

def prefetch_neighbors(chunk_id):
    # After on-disk hit or remote fetch, requests chunk on either side in ordering [S1, unofficial]
    left_id = get_neighbor_chunk_id(chunk_id, direction=-1)
    right_id = get_neighbor_chunk_id(chunk_id, direction=1)
    async_fetch(left_id)
    async_fetch(right_id)

def idle_prefetch():
    # Warms adjacent slice during orbit/slider playback to hide ~100 ms latency [S1, unofficial]
    current_slice = get_current_slice_id()
    next_slice = get_adjacent_slice_id(current_slice, direction=1)
    async_fetch(next_slice)

def manage_heap_pressure():
    # Shrinks L0 and L1 ceilings when heap is tight, grows S-cache up to 2 GB [S1, unofficial]
    # Exact eviction policy not given in sources [G]
    pass
```

**Parameters:**
*   **L2 Size:** 2 GB (Fixed) [S1, unofficial].
*   **L1 Ceiling:** 100 MB [S1, unofficial].
*   **L0 Ceiling:** 200 MB [S1, unofficial].
*   **S-cache Default:** 128 MB [S1, unofficial].
*   **S-cache Max:** 2 GB [S1, unofficial].
*   **Remote Fetch Latency:** Order of 100 ms [S1, unofficial].

**Edge Cases:**
*   **Heap Pressure:** If heap memory is insufficient, L0 and L1 ceilings shrink dynamically, potentially causing more frequent recompression or re-fetching [S1, unofficial].
*   **Mobile Memory:** Rendering budgets are clamped to mobile memory limits, which may affect cache sizes or prefetch aggressiveness [S1, unofficial].

## Subtopics

### [VxlSlabColumn Priority Cache](4742-vxlslabcolumn-priority-cache.md)
The VxlSlabColumn Priority Cache is a memory management mechanism for voxel volume data that loads and recompiles slab columns based on priority levels to ensure spatial neighbors are available for surface operations. It differs from sibling caches like the Rolling 3-Row Slab Cache by explicitly managing dirty/clean states via a priority enum (`Edit`, `Neighbor`, `Support`) and triggering recompilation only when necessary, rather than relying solely on spatial sliding windows.

### [Rolling 3-Row Slab Cache](4747-rolling-3-row-slab-cache.md)
The Rolling 3-Row Slab Cache is a memory structure that stores voxel column data across a rolling window of three Y-rows to support incremental updates and neighbor-dependent recompilation in dense voxel grids. It sets itself apart from sibling caches like the [VxlSlabColumn Priority Cache](4742-vxlslabcolumn-priority-cache.md) by maintaining a fixed-size sliding window of Y-indices rather than priority-based or full-grid storage, optimizing for locality during column-wise edits. It builds on the principles of [Multi-Level Cache and Prefetching](4631-multi-level-cache-and-prefetching.md) by restricting active memory to a narrow spatial band.

### [Dataset Prefetching](14225-dataset-prefetching.md)
Dataset Prefetching is a technique that decouples data retrieval from the rendering loop by running a background thread to prefetch batches into a queue. It solves the problem of CPU-bound data loading stalling the GPU by ensuring that volume data is available in memory before it is needed for rendering. Unlike sibling methods that focus on spatial caching hierarchies (e.g., [VxlSlabColumn Priority Cache](4742-vxlslabcolumn-priority-cache.md) or [Rolling 3-Row Slab Cache](4747-rolling-3-row-slab-cache.md)), this method focuses on temporal overlap of I/O and computation.

### Page Table and Cache Management
This variant manages the multi-resolution page directory, page table cache, and block cache using LRU/MRU schemes, where updates are initiated by cache misses reported by the ray-caster [S2]. On a cache miss, the CPU first checks if the required block is resident in the CPU block cache (a larger version in CPU memory of the GPU block cache) [S2]. If the block is in the CPU block cache, it is downloaded into the GPU block cache, and the page directory and page table cache are updated accordingly [S2]. If the block is not in the CPU block cache, a request for the 3D volume data block is issued asynchronously to the volume data construction stage [S2]. The download of block data and page table updates are deferred until the constructed block arrives [S2].

## Operating characteristics
*   **Time Complexity:** Cache hit time is O(1) for memory tiers (L0/L1) [S1, unofficial]. Disk hit (L2) is O(1) for access but involves I/O latency [S1, unofficial]. Remote fetch is O(Latency), where latency is ~100 ms [S1, unofficial]. Prefetching overlaps I/O with computation, effectively hiding latency during interactive navigation [S1, unofficial].
*   **Memory Complexity:** Total memory usage is bounded by the sum of cache ceilings (L0 + L1 + S-cache) plus OPFS usage (L2) [S1, unofficial]. Max in-memory usage is approximately 400 MB (L0+L1) + 2 GB (S-cache) = 2.4 GB, plus 2 GB on disk [S1, unofficial].
*   **Measured Performance:** The architecture is designed to hide remote-fetch latency (order of 100 ms) during user orbit or slider playback [S1, unofficial].
*   **Strengths:** High responsiveness for interactive web-based visualization; efficient use of browser storage (OPFS); adaptive to memory constraints [S1, unofficial].
*   **Limitations:** Dependent on HTTP range request support; performance degrades if heap memory is severely constrained; complex cache coherence management between tiers [S1, unofficial].
*   **When to Use:** Interactive web visualization of large, chunked scientific datasets (e.g., microscopy, Gaussian splats) [S1, unofficial].
*   **When Not to Use:** Offline batch rendering where data is already in memory; environments without HTTP range request support; systems with extremely low memory where even the minimum cache footprint is prohibitive [S1, unofficial].

## Background
The Multi-Level Cache and Prefetching architecture was introduced in the Luxar viewer for interactive web visualization of multidimensional scientific data, specifically Gaussian splatting for microscopy [S1, unofficial]. It leverages browser-native storage (OPFS) and WebAssembly for decompression to enable scalable interactive viewing of large datasets in the browser [S1, unofficial]. The approach was developed to address the challenges of loading and rendering massive volumetric datasets over the web with low latency [S1, unofficial].

## Sources
[S1] Luxar: Gaussian splatting for microscopy and scalable interactive web visualisation of multidimensional scientific data, https://doi.org/10.64898/2026.09.22.753676, fetched 2026-10-26 (unofficial)
[S2] PDF Interactive Volume Exploration of Petascale Microscopy Data Streams ..., https://vccvisualization.org/publications/2012_hadwiger_petascale.pdf, fetched 2026-10-28
