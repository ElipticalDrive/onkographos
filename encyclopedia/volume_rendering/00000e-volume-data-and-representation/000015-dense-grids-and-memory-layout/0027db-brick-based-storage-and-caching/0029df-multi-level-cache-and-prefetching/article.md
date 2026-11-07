# Multi-Level Cache and Prefetching

Also called: HTTP Range Request Caching, Luxar Viewer Cache Architecture

## Overview
Multi-Level Cache and Prefetching is a memory management strategy for web-based volume visualization that bridges the gap between persistent storage and high-speed rendering by utilizing a four-tier cache hierarchy and aggressive prefetching [S1, unofficial]. It solves the problem of high-latency remote data fetching (order of 100 ms) in browser environments by ensuring that frequently accessed data slices and chunks are available in faster memory tiers, thereby hiding I/O latency during user interactions such as orbiting or slider playback [S1, unofficial]. It differs from sibling methods like [Brick Caching with LRU Replacement](879-brick-caching-with-lru-replacement.md) by explicitly separating compressed and decompressed states across distinct memory ceilings and leveraging the browser's Origin Private File System (OPFS) for persistent storage, rather than relying solely on volatile GPU or CPU caches [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A Zarr archive of volumetric data (e.g., Gaussian splats) stored as chunk files, accessible via HTTP range requests.
*   **Output:** Fully decoded slice ladders and decompressed chunks resident in memory for immediate rendering.

**Data Structures:**
The system employs a four-level cache hierarchy. The structure is defined by the following tiers, ordered from slowest/persistent to fastest/volatile:

1.  **L2 (Persistent Storage):** The browser’s Origin Private File System (OPFS).
    *   **Capacity:** Fixed at 2 GB.
    *   **Content:** Compressed chunks (Blosc/zstd encoded).
    *   **Persistence:** Survives browser restarts.
2.  **L1 (Compressed In-Memory):**
    *   **Capacity:** Ceiling of 100 MB.
    *   **Content:** Compressed chunks.
    *   **Behavior:** Heap-aware; shrinks when memory is tight.
3.  **L0 (Decompressed In-Memory):**
    *   **Capacity:** Ceiling of 200 MB.
    *   **Content:** Decompressed chunks.
    *   **Behavior:** Heap-aware; shrinks when memory is tight.
4.  **S-cache (Slice Cache):**
    *   **Capacity:** Default 128 MB, grows up to 2 GB based on heap availability.
    *   **Content:** Fully decoded slice ladders.
    *   **Behavior:** Allows revisited slices to skip the chunk path entirely.

**Numbered Steps:**
1.  **Request Initiation:** The viewer requests data for a specific spatial region or slice.
2.  **Cache Lookup:** The system checks the S-cache first. If the fully decoded slice ladder is present, it is returned immediately [S1, unofficial].
3.  **Chunk Fetching:** If not in S-cache, the system checks L0 (decompressed chunks) and L1 (compressed chunks).
4.  **Remote Fetch:** If the data is not in L0 or L1, an HTTP range request is issued to fetch the compressed chunk from the Zarr archive [S1, unofficial].
5.  **Decompression:** The fetched compressed chunk is decompressed using a WebAssembly (WASM) codec (Blosc/zstd) [S1, unofficial].
6.  **L0 Insertion:** The decompressed chunk is placed into the L0 cache. If L0 exceeds its 200 MB ceiling, older entries are evicted [S1, unofficial].
7.  **L1 Insertion:** The compressed chunk is placed into the L1 cache. If L1 exceeds its 100 MB ceiling, older entries are evicted [S1, unofficial].
8.  **L2 Persistence:** The compressed chunk is written to the L2 OPFS storage if not already present [S1, unofficial].
9.  **S-cache Population:** If the chunk contributes to a full slice, the decoded slice ladder is added to the S-cache. If the S-cache exceeds its dynamic limit (up to 2 GB), older slices are evicted [S1, unofficial].
10. **Prefetching:**
    *   **Immediate Prefetch:** After an on-disk hit or a remote fetch, a prefetcher requests the chunk on either side in the data ordering [S1, unofficial].
    *   **Idle Prefetch:** An idle-time prefetcher warms the adjacent slice to hide remote-fetch latency (order of 100 ms) during user orbit or slider playback [S1, unofficial].

**Formulas:**
*   **S-cache Growth:** The S-cache capacity $C_{S}$ is dynamic:
    $$ C_{S} = \min(2 \text{ GB}, \max(128 \text{ MB}, f(\text{Heap Size}))) $$
    where $f(\text{Heap Size})$ is a monotonic function of available heap memory [S1, unofficial].

**Parameters:**
*   **L2 Capacity:** 2 GB (Fixed) [S1, unofficial].
*   **L1 Ceiling:** 100 MB [S1, unofficial].
*   **L0 Ceiling:** 200 MB [S1, unofficial].
*   **S-cache Default:** 128 MB [S1, unofficial].
*   **S-cache Max:** 2 GB [S1, unofficial].
*   **Remote Fetch Latency:** Order of 100 ms [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Heap Tightness:** When system memory is constrained, L0 and L1 automatically shrink below their configured ceilings to prevent out-of-memory errors [S1, unofficial].
*   **Dimensionality:** The system supports up to 16 dimensions using WASM-accelerated kernels; beyond 16 dimensions, it falls back to a slower TypeScript implementation [S1, unofficial].

## Subtopics

### [VxlSlabColumn Priority Cache](4742-vxlslabcolumn-priority-cache.md)
A memory management mechanism that loads and recompiles slab columns based on priority levels (`Edit`, `Neighbor`, `Support`) to ensure spatial neighbors are available for surface operations. It differs by explicitly managing dirty/clean states and triggering recompilation only when necessary, rather than relying on spatial sliding windows.

### [Rolling 3-Row Slab Cache](4747-rolling-3-row-slab-cache.md)
A memory structure storing voxel column data across a rolling window of three Y-rows to support incremental updates and neighbor-dependent recompilation in dense voxel grids. It sets itself apart by maintaining a fixed-size sliding window of Y-indices, optimizing for locality during column-wise edits.

## Operating characteristics
*   **Time Complexity:** Lookup in L0/L1/S-cache is $O(1)$ assuming hash-based indexing. Remote fetch is $O(L)$ where $L$ is network latency (approx. 100 ms) [S1, unofficial]. Prefetching adds overhead but hides this latency during interactive periods.
*   **Memory Complexity:** Total memory usage is bounded by the sum of L0, L1, and S-cache ceilings, plus the fixed L2 storage. Maximum volatile memory usage is approximately 4.128 GB (200 MB + 100 MB + 2 GB) under worst-case heap expansion [S1, unofficial].
*   **Measured Performance:** The architecture is designed to hide remote-fetch latency (order of 100 ms) during user orbit or slider playback, ensuring smooth interaction [S1, unofficial].
*   **Strengths:** Persistent caching via OPFS reduces repeated network requests; heap-aware tiers prevent browser crashes on low-memory devices; idle prefetching masks I/O latency.
*   **Limitations:** High memory footprint (up to ~4 GB); dependency on browser support for OPFS and WebAssembly; fallback to TypeScript for >16 dimensions reduces performance.
*   **When to Use:** Ideal for web-based visualization of large, multidimensional scientific datasets (e.g., microscopy) where network latency is significant and persistent caching is beneficial.
*   **When Not to Use:** Unsuitable for environments without OPFS support or where memory is strictly constrained below 128 MB.

## Background
The Multi-Level Cache and Prefetching architecture was introduced in the Luxar viewer, a tool for Gaussian splatting for microscopy and scalable interactive web visualization of multidimensional scientific data [S1, unofficial]. It was developed to address the challenges of loading and rendering large Zarr archives in web browsers, leveraging HTTP range requests and browser-native storage APIs to provide a responsive user experience [S1, unofficial].

## Sources
[S1] Luxar: Gaussian splatting for microscopy and scalable interactive web visualisation of multidimensional scientific data, https://doi.org/10.64898/2026.09.22.753676, fetched 2026-10-26 (unofficial)
