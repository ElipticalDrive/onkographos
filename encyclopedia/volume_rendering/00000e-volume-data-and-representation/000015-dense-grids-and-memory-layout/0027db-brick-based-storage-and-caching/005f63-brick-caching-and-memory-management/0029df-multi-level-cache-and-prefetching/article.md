# Multi-Level Cache and Prefetching

Also called: HTTP Range Request Caching, Luxar Viewer Cache Architecture

## Overview
Multi-Level Cache and Prefetching is a memory management strategy for interactive volume rendering that decouples data retrieval from the rendering loop using a hierarchical cache structure and background prefetching [S1, unofficial]. It solves the problem of high-latency remote data fetching (order of 100 ms) by ensuring that decompressed geometry and compressed chunks are available in local memory before they are needed [S1, unofficial]. It differs from sibling methods like [Brick Caching with LRU Replacement](879-brick-caching-with-lru-replacement.md) by explicitly managing four distinct tiers of storage (disk, compressed memory, decompressed memory, and slice ladders) with heap-aware dynamic sizing, rather than a single fixed pool [S1, unofficial]. It also differs from [Raymarching with LOD-based Brick Caching](5776-raymarching-with-lod-based-brick-caching.md) by focusing on chunk-level and slice-level locality for splat-based rendering rather than voxel block LODs [S1, unofficial].

## How it works
The method operates on a four-level cache hierarchy that stores data fetched via HTTP range requests from Zarr archives or similar chunked storage formats [S1, unofficial]. The system assumes a background prefetcher is active to hide I/O latency.

**Inputs and Outputs:**
*   **Input:** A sequence of view rays or slice planes requested by the user (orbit, slider playback).
*   **Output:** Rendered pixels or updated slice views, with cache state updated to reflect accessed data.

**Data Structures:**
The cache consists of four tiers, each with specific storage characteristics and size constraints:
1.  **L2 (Disk Cache):** Stored in the browser’s Origin Private File System (OPFS). Fixed size of 2 GB. Stores compressed chunks. Survives browser restarts.
2.  **L1 (Compressed Memory):** Stores compressed chunks in RAM. Ceiling of 100 MB.
3.  **L0 (Decompressed Memory):** Stores decompressed chunks in RAM. Ceiling of 200 MB.
4.  **S-cache (Slice Cache):** Stores fully decoded slice ladders. Default size 128 MB, grows up to 2 GB. Allows revisited slices to skip the chunk path entirely.

**Heap Awareness:**
The in-memory tiers (L0, L1, S-cache) are heap-aware. When the available heap memory is tight, L0 and L1 shrink from their configured ceilings. The S-cache grows with the available heap from its 128 MB default up to 2 GB [S1, unofficial].

**Numbered Steps:**
1.  **Request:** The renderer requests a data chunk or slice.
2.  **L0 Check:** Check if the decompressed chunk is in L0. If yes, use it.
3.  **L1 Check:** If not in L0, check if the compressed chunk is in L1. If yes, decompress it (using WASM-accelerated Blosc/zstd codecs) and place in L0. Evict from L1 if necessary.
4.  **L2 Check:** If not in L1, check if the compressed chunk is in L2 (OPFS). If yes, load into L1, then decompress to L0.
5.  **Remote Fetch:** If not in L2, issue an HTTP range request to fetch the chunk. Upon arrival, store in L2, then L1, then L0.
6.  **S-cache Optimization:** If the request is for a full slice, check the S-cache. If the fully decoded slice ladder is present, use it directly, bypassing chunk-level fetching.
7.  **Prefetching:**
    *   **Immediate:** After an on-disk hit or remote fetch, the prefetcher requests the chunk on either side in the data ordering [S1, unofficial].
    *   **Idle:** An idle-time prefetcher warms the adjacent slice to hide remote-fetch latency (order of 100 ms) during user orbit or slider playback [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def get_data(request):
    # request contains chunk_id or slice_id
    
    # 1. Check L0 (Decompressed)
    if request in L0:
        return L0[request]
    
    # 2. Check L1 (Compressed)
    if request in L1:
        decompressed = decompress(L1[request])
        L0[request] = decompressed
        manage_heap_pressure() # May shrink L0/L1 ceilings
        return decompressed
        
    # 3. Check L2 (Disk/OPFS)
    if request in L2:
        compressed = load_from_opfs(request)
        L1[request] = compressed
        return get_data(request) # Retry to L1 -> L0
        
    # 4. Remote Fetch
    compressed = fetch_remote(request)
    L2[request] = compressed
    L1[request] = compressed
    decompressed = decompress(compressed)
    L0[request] = decompressed
    
    # 5. Prefetch neighbors
    prefetch_neighbors(request)
    
    return decompressed

def prefetch_neighbors(request):
    # Request chunk on either side in ordering
    left_chunk = get_neighbor_chunk(request, direction=-1)
    right_chunk = get_neighbor_chunk(request, direction=1)
    async_fetch(left_chunk)
    async_fetch(right_chunk)

def idle_prefetch():
    # Warm adjacent slice during orbit/slider
    current_slice = get_current_slice()
    next_slice = get_adjacent_slice(current_slice, direction=1)
    async_fetch_slice(next_slice)
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
This variant manages the multi-resolution page directory, page table cache, and block cache using LRU/MRU schemes, where updates are initiated by cache misses reported by the ray-caster [S2]. On a cache miss, the CPU first checks if the required block is resident in the CPU block cache (a larger version in CPU memory of the GPU block cache) [S2]. If the block is in the CPU block cache, it is downloaded into the GPU block cache, and the page directory and page table cache are updated accordingly [S2]. If the block is not in the CPU block cache, a request for the 3D volume data block is issued asynchronously to the volume data construction stage. The download of block data and page table updates are deferred until the constructed block arrives [S2].

## Operating characteristics
*   **Time Complexity:** Cache hit time is O(1) for memory tiers (L0/L1). Disk hit (L2) is O(1) for access but involves I/O latency. Remote fetch is O(Latency), where latency is ~100 ms [S1, unofficial]. Prefetching overlaps I/O with computation, effectively hiding latency during interactive navigation.
*   **Memory Complexity:** Total memory usage is bounded by the sum of cache ceilings (L0 + L1 + S-cache) plus OPFS usage (L2). Max in-memory usage is approximately 400 MB (L0+L1) + 2 GB (S-cache) = 2.4 GB, plus 2 GB on disk [S1, unofficial].
*   **Measured Performance:** The architecture is designed to hide remote-fetch latency (order of 100 ms) during user orbit or slider playback [S1, unofficial].
*   **Strengths:** High responsiveness for interactive web-based visualization; efficient use of browser storage (OPFS); adaptive to memory constraints.
*   **Limitations:** Dependent on HTTP range request support; performance degrades if heap memory is severely constrained; complex cache coherence management between tiers.
*   **When to Use:** Interactive web visualization of large, chunked scientific datasets (e.g., microscopy, Gaussian splats).
*   **When Not to Use:** Offline batch rendering where data is already in memory; environments without HTTP range request support; systems with extremely low memory where even the minimum cache footprint is prohibitive.

## Background
The Multi-Level Cache and Prefetching architecture was introduced in the Luxar viewer for interactive web visualization of multidimensional scientific data, specifically Gaussian splatting for microscopy [S1, unofficial]. It leverages browser-native storage (OPFS) and WebAssembly for decompression to enable scalable interactive viewing of large datasets in the browser [S1, unofficial]. The approach was developed to address the challenges of loading and rendering massive volumetric datasets over the web with low latency [S1, unofficial].

## Sources
[S1] Luxar: Gaussian splatting for microscopy and scalable interactive web visualisation of multidimensional scientific data, https://doi.org/10.64898/2026.09.22.753676, fetched 2026-10-26 (unofficial)
[S2] PDF Interactive Volume Exploration of Petascale Microscopy Data Streams ..., https://vccvisualization.org/publications/2012_hadwiger_petascale.pdf, fetched 2026-10-28
