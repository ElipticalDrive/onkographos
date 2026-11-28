# Brick Caching and Memory Management


## Overview
Brick Caching and Memory Management organizes volume data into fixed-size blocks (bricks) and manages their transfer between slow storage and fast rendering memory. It solves limited capacity and high latency by maintaining a working set of frequently accessed bricks and evicting unused ones. It sets itself apart from sibling methods like [Bricking](914-bricking.md) (which defines the grid structure) and [Raymarching with LOD-based Brick Caching](5776-raymarching-with-lod-based-brick-caching.md) (which optimizes traversal) by focusing specifically on the dynamic allocation, eviction policies, and hierarchical storage tiers of the brick data itself.

## Shared principles

## Subtopics

### [Brick Caching with LRU Replacement](879-brick-caching-with-lru-replacement.md)
Manages a fixed-size pool of volume data blocks, mapping logical brick coordinates to physical memory locations and evicting the least-recently-used blocks when the pool is full. It solves the problem of limited GPU memory capacity and slow data transfer rates by ensuring that the most recently accessed brick data remains in the fast working set. It distinguishes itself from standard spatial traversal by decoupling the processing order of blocks from the ray order within a frame, relying on a sorted auxiliary list structure to ensure cache hits for spatially coherent rays. Choose it for GPU raycasting with spatially coherent access patterns.

### [Multi-Level Cache and Prefetching](4631-multi-level-cache-and-prefetching.md)
Decouples data retrieval from the rendering loop using a hierarchical cache structure and background prefetching. It solves the problem of high-latency remote data fetching (order of 100 ms) by ensuring that decompressed geometry and compressed chunks are available in local memory before they are needed. It differs from siblings by explicitly managing four distinct tiers of storage (disk, compressed memory, decompressed memory, and slice ladders) with heap-aware dynamic sizing, rather than a single fixed pool. Choose it for web-based or remote visualization where network latency is high.

### [Raymarching with LOD-based Brick Caching](5776-raymarching-with-lod-based-brick-caching.md)
Renders Compressed Segmentation Volumes (CSVs) by raymarching with one ray per pixel, decompressing bricks on-demand at the required Level of Detail (LOD) and storing them in a GPU cache. It differs from its siblings by exploiting temporal coherency with a one-frame decompression lag, using palette-based empty-space skipping before cache allocation, and employing a contiguous base-element cache structure that omits the superblock/column concepts found in Shading Atlas Streaming (SAS). Choose it for interactive segmentation volume rendering with varying detail requirements.

### Brick Cache Configuration
A shared cache between brick map geometric primitives and brick map textures, as they use the same underlying data structure. The brick cache size can be changed using the `Option "limits" "brickmemory"` command in the RIB file, or specified in the `rendermn.ini` file using the syntax `/prman/brickmemory`. The `brickmemory` parameter specifies the cache size in kilobytes, with a default of 10 MB per thread (e.g., 10240 KB). The RIB Option overrides the `rendermn.ini` setting if both are specified.

### Bricking (Large Volume Handling)
Cuts arbitrarily large volumes into $32^3$ bricks which are rendered separately. The resulting brick images are composited together in image space in the correct order. Optionally, volume traversal can be restricted to a cuboid subvolume to render a subset of a larger volume. Hardware implementations historically supported 128 MB of volume storage (a $32^3$ volume of 8-bit voxels), with driver software handling automatic swapping of volume data to and from host memory for larger datasets.

## Background
Not given in the sources.

## Sources
[S1] The Brick Map Geometric Primitive, https://smartcg.com/tech/cg/refs/man/RMS_1.0/rman/prman_technical_rendering/AppNotes/brickmapgprim.html, fetched 2026-10-28
[S2] Real-Time Volume Rendering, https://cfcs.pku.edu.cn/baoquan/docs/20180622110335231616.pdf, fetched 2026-10-30 (unofficial)
