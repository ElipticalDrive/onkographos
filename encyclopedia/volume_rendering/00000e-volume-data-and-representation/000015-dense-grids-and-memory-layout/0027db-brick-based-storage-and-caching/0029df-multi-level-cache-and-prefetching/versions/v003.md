# Multi-Level Cache and Prefetching

Also called: HTTP Range Request Caching, Luxar Viewer Cache Architecture

## Overview
Multi-Level Cache and Prefetching is a memory management strategy for volumetric and splat-based data that utilizes a four-tier caching hierarchy combined with proactive data fetching to hide network latency. It solves the problem of slow interactive performance when accessing large, remote datasets by ensuring that frequently accessed or spatially adjacent data is available in fast memory before it is needed. It differs from sibling methods like [Brick Caching with LRU Replacement](879-brick-caching-with-lru-replacement.md) by explicitly separating compressed and decompressed storage into distinct tiers (L1/L0) and integrating an S-cache for slice-level acceleration, while its prefetching logic is driven by access patterns and idle time rather than simple spatial locality alone.

## How it works
The method manages data flow from remote storage (via HTTP Range Requests) through a four-level cache hierarchy to the rendering engine.

**Inputs and Outputs:**
*   **Input:** Remote dataset chunks (e.g., Zarr archives), user interaction events (orbit, slider playback).
*   **Output:** Decoded geometry/splat data ready for rendering; prefetched data stored in cache tiers.

**Data Structures and Layout:**
The cache consists of four levels with specific memory constraints and purposes [S1, unofficial]:
1.  **L2 (Disk Cache):** Stored in the browser’s Origin Private File System (OPFS). Fixed size of 2 GB. Survives browser restarts. Stores compressed chunks.
2.  **L1 (Compressed Memory Cache):** Stores compressed in-memory chunks. Ceiling of 100 MB.
3.  **L0 (Decompressed Memory Cache):** Stores decompressed in-memory chunks. Ceiling of 200 MB.
4.  **S-cache (Slice Cache):** Stores fully decoded slice ladders. Allows revisited slices to skip the chunk decoding path. Default size 128 MB, grows up to 2 GB.

**Numbered Steps:**
1.  **Request Initiation:** The renderer requests data for a specific spatial region or slice.
2.  **Cache Lookup:** The system checks the S-cache first. If the slice is present, it is returned immediately. If not, it checks L0, then L1, then L2.
3.  **Fetch and Decode:**
    *   If data is in L2, it is read from OPFS.
    *   If data is remote, it is fetched via HTTP Range Requests.
    *   Data is decompressed using Blosc/zstd codecs (running in WebAssembly) [S1, unofficial].
4.  **Cache Insertion:**
    *   Compressed data is stored in L1.
    *   Decompressed data is stored in L0.
    *   If the data constitutes a full slice, it is also stored in the S-cache.
5.  **Heap-Aware Adjustment:**
    *   If system memory (heap) is tight, L0 and L1 shrink from their configured ceilings (200 MB and 100 MB respectively) to free memory [S1, unofficial].
    *   The S-cache grows with available heap space, up to its maximum of 2 GB [S1, unofficial].
6.  **Prefetching:**
    *   **Immediate Prefetch:** After an on-disk hit or a remote fetch, a prefetcher requests the chunk on either side in the data ordering [S1, unofficial].
    *   **Idle Prefetch:** During user orbit or slider playback, an idle-time prefetcher warms the adjacent slice to hide remote-fetch latency (order of 100 ms) [S1, unofficial].

**Parameters:**
*   L2 Size: 2 GB (Fixed) [S1, unofficial].
*   L1 Ceiling: 100 MB [S1, unofficial].
*   L0 Ceiling: 200 MB [S1, unofficial].
*   S-cache Default: 128 MB [S1, unofficial].
*   S-cache Max: 2 GB [S1, unofficial].
*   Remote Fetch Latency: ~100 ms [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Memory Pressure:** Under high memory pressure, L0 and L1 eviction may occur more frequently than expected due to the shrinking ceilings [S1, unofficial].
*   **Latency Hiding:** The effectiveness of idle prefetching depends on the user's interaction speed; if interaction is too rapid, the prefetched adjacent slice may not be used before the next request [S1, unofficial].

## Subtopics

### [VxlSlabColumn Priority Cache](4742-vxlslabcolumn-priority-cache.md)
A priority-based cache for voxel data that manages dirty/clean states to ensure spatial neighbors are available for surface operations.

### [Rolling 3-Row Slab Cache](4747-rolling-3-row-slab-cache.md)
A sliding window cache for voxel columns that optimizes for locality during column-wise edits.

## Operating characteristics
*   **Time Complexity:** Lookup is $O(1)$ for cache hits. Remote fetch latency is approximately 100 ms, which is hidden by the idle-time prefetcher during user interaction [S1, unofficial].
*   **Memory Complexity:** Total cache size ranges from 128 MB (S-cache default) to 4.3 GB (L2 2 GB + L1 100 MB + L0 200 MB + S-cache 2 GB) [S1, unofficial].
*   **Strengths:** Hides network latency effectively; separates compressed and decompressed storage to optimize memory usage; persistent disk cache reduces repeated network requests.
*   **Limitations:** Complex memory management logic; performance depends on heap availability; WebAssembly decoding overhead for Blosc/zstd.
*   **When to Use:** Interactive web-based visualization of large, remote volumetric or splat datasets where network latency is a bottleneck.
*   **When Not to Use:** Local rendering with fast storage where cache overhead outweighs benefits; scenarios with strict memory constraints below 128 MB.

## Background
The method was introduced in the Luxar viewer for interactive web visualization of multidimensional scientific data, specifically Gaussian splatting for microscopy [S1, unofficial]. It leverages modern web technologies like OPFS and WebAssembly to manage large datasets in the browser environment.

## Sources
[S1] Luxar: Gaussian splatting for microscopy and scalable interactive web visualisation of multidimensional scientific data, https://doi.org/10.64898/2026.09.22.753676, fetched 2026-10-26 (unofficial)
