# Multi-Level Cache and Prefetching

Also called: HTTP Range Request Caching, Luxar Viewer Cache Architecture

## Overview
This method manages the loading and residency of volumetric data (specifically Gaussian splats in Zarr archives) by combining HTTP range requests with a four-tier caching hierarchy. It solves the problem of high-latency remote data fetching (order of 100 ms) in web-based volume visualization by ensuring frequently accessed data resides in fast memory or persistent storage. It differs from sibling caches like [Brick Caching with LRU Replacement](879-brick-caching-with-lru-replacement.md) by utilizing a strict tiered structure (L2/L1/L0/S-cache) with heap-aware dynamic resizing and explicit idle-time prefetching, rather than a single LRU pool.

## How it works
**Inputs:**
- A Zarr archive containing chunked data (e.g., `.luxar.zarr.zip`).
- HTTP range request capability from the client.
- System memory (heap) status.

**Outputs:**
- Decoded voxel/splat data ready for rendering.
- Prefetch requests for adjacent data.

**Data Structures:**
The cache consists of four distinct tiers. The memory limits for the in-memory tiers are dynamic based on heap availability.

1.  **L2 (Persistent Storage):**
    -   Location: Browser Origin Private File System (OPFS).
    -   Size: Fixed 2 GB.
    -   Content: Compressed chunks (Blosc/zstd).
    -   Persistence: Survives browser restarts.

2.  **L1 (Compressed In-Memory):**
    -   Location: RAM.
    -   Size: Dynamic ceiling, default 100 MB.
    -   Content: Compressed chunks.
    -   Behavior: Shrinks from ceiling when heap is tight.

3.  **L0 (Decompressed In-Memory):**
    -   Location: RAM.
    -   Size: Dynamic ceiling, default 200 MB.
    -   Content: Decompressed chunks.
    -   Behavior: Shrinks from ceiling when heap is tight.

4.  **S-cache (Slice Ladder):**
    -   Location: RAM.
    -   Size: Dynamic, default 128 MB, grows up to 2 GB.
    -   Content: Fully decoded slice ladders.
    -   Behavior: Grows with heap; allows revisited slices to skip the chunk path entirely.

**Algorithm Steps:**

1.  **Data Encoding and Indexing:**
    Data is stored in Zarr archives. Encoding uses semantic types (8 types) with a width-aware Blosc-zstd level 9 policy:
    -   Byte shuffle for multi-byte integer codes.
    -   No shuffle for single-byte codes and floating-point arrays [S1, unofficial].

2.  **Fetch Request:**
    When data is required, the system issues an HTTP range request to fetch the specific chunk from the archive (or the uncompressed `.luxar.zarr.zip` member) [S1, unofficial].

3.  **Cache Population (L2 -> L1 -> L0):**
    -   If the chunk is not in L2, fetch it from the remote server.
    -   Store the compressed chunk in L2 (OPFS).
    -   Store the compressed chunk in L1 (if space permits).
    -   Decompress the chunk (using WASM Blosc/zstd codec) and store the result in L0 (if space permits).
    -   If the chunk is part of a slice ladder, decode the full slice and store it in the S-cache [S1, unofficial].

4.  **Cache Eviction and Resizing:**
    -   Monitor system heap usage.
    -   If heap is tight, reduce the effective ceilings of L0 and L1 below their configured maximums [S1, unofficial].
    -   If heap is abundant, allow the S-cache to grow from its 128 MB default towards 2 GB [S1, unofficial].

5.  **Prefetching:**
    -   **On-Disk/Remote Hit:** Immediately after a chunk is fetched or found on disk, request the chunk on either side in the linear ordering [S1, unofficial].
    -   **Idle-Time:** During user orbit or slider playback, an idle-time prefetcher warms the adjacent slice to hide remote-fetch latency [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class MultiLevelCache:
    def __init__(self):
        self.l2_opfs = OPFSStorage(max_size_gb=2)
        self.l1_compressed = MemoryBuffer(ceiling_mb=100)
        self.l0_decompressed = MemoryBuffer(ceiling_mb=200)
        self.s_cache_slice_ladders = MemoryBuffer(default_mb=128, max_mb=2048)
        self.heap_monitor = HeapMonitor()

    def get_data(self, chunk_id):
        # Check L0
        if chunk_id in self.l0_decompressed:
            return self.l0_decompressed.get(chunk_id)
        
        # Check L1
        if chunk_id in self.l1_compressed:
            decompressed = decompress_wasm(self.l1_compressed.get(chunk_id))
            self.l0_decompressed.put(chunk_id, decompressed)
            self.l2_opfs.put(chunk_id, self.l1_compressed.get(chunk_id))
            return decompressed

        # Check L2 (OPFS)
        if chunk_id in self.l2_opfs:
            compressed = self.l2_opfs.get(chunk_id)
            self.l1_compressed.put(chunk_id, compressed)
            decompressed = decompress_wasm(compressed)
            self.l0_decompressed.put(chunk_id, decompressed)
            return decompressed

        # Remote Fetch
        compressed = http_range_request(chunk_id)
        self.l2_opfs.put(chunk_id, compressed)
        self.l1_compressed.put(chunk_id, compressed)
        decompressed = decompress_wasm(compressed)
        self.l0_decompressed.put(chunk_id, decompressed)
        
        # Trigger Prefetch
        self.prefetch_adjacent(chunk_id)
        
        return decompressed

    def prefetch_adjacent(self, current_chunk_id):
        # Request chunk on either side in ordering
        left_id = get_neighbor_id(current_chunk_id, direction=-1)
        right_id = get_neighbor_id(current_chunk_id, direction=1)
        # Asynchronous fetch
        async_fetch(left_id)
        async_fetch(right_id)

    def update_heap_status(self, current_heap_usage):
        # L0 and L1 shrink from ceilings when heap is tight
        if current_heap_usage > THRESHOLD_TIGHT:
            self.l0_decompressed.set_ceiling(self.l0_decompressed.get_ceiling() * 0.8)
            self.l1_compressed.set_ceiling(self.l1_compressed.get_ceiling() * 0.8)
        else:
            # S-cache grows with heap
            self.s_cache_slice_ladders.set_max_mb(min(2048, current_heap_usage * SCALE_FACTOR))

    def idle_prefetch(self, current_slice_id):
        # Warm adjacent slice during orbit/slider
        adjacent_slice = get_adjacent_slice(current_slice_id)
        async_fetch_slice(adjacent_slice)
```

## Subtopics

### [VxlSlabColumn Priority Cache](4742-vxlslabcolumn-priority-cache.md)
Manages slab columns via priority levels (Edit, Neighbor, Support) and dirty/clean states.

### [Rolling 3-Row Slab Cache](4747-rolling-3-row-slab-cache.md)
Maintains a fixed-size sliding window of Y-rows for incremental updates.

## Operating characteristics
- **Latency Hiding:** The primary benefit is hiding remote-fetch latency, which is on the order of 100 ms [S1, unofficial].
- **Memory Efficiency:** The heap-aware design allows the cache to adapt to device constraints, clamping rendering budgets on mobile devices [S1, unofficial].
- **Complexity:** The four-level hierarchy adds management complexity compared to single-tier caches but provides granular control over compressed vs. decompressed data residency.
- **Scalability:** Supports n-dimensional datasets by displaying three spatial dimensions and navigating others via sliders, with the cache handling the underlying chunk resolution [S1, unofficial].

## Background
This architecture originates from the Luxar viewer, designed for interactive web visualization of multidimensional scientific data (specifically Gaussian splatting for microscopy) [S1, unofficial]. It leverages modern web standards like OPFS and WebAssembly for decoding to enable scalable visualization in the browser.

## Sources
[S1] Luxar: Gaussian splatting for microscopy and scalable interactive web visualisation of multidimensional scientific data, https://doi.org/10.64898/2026.09.22.753676, fetched 2026-10-26 (unofficial)
