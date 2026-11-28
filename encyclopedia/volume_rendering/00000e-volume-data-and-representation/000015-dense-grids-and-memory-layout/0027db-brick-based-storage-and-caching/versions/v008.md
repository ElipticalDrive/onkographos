# Brick-based storage and caching
Also called: Bricking, Brick Caching, Brick-based Volume Raycasting, N³-tree with Brick Pool and Node Pool, Multi-Level Cache and Prefetching, Bricked Volume Hierarchy, Raymarching with LOD-based Brick Caching, Cyclic Buffer for Smoke Volume, Homogeneous Brick Optimization

## Overview
Brick-based storage and caching decomposes large volumetric datasets into fixed-size, power-of-two sub-volumes called "bricks" to manage memory capacity and transfer bandwidth efficiently [S1, unofficial]. It solves the problem of rendering datasets larger than available GPU memory by streaming only the necessary bricks into a resident cache, often using eviction policies like LRU or hierarchical prefetching [S1, unofficial]. It sets itself apart from sibling methods by explicitly decoupling the logical resolution levels from the geometric subdivision of space (in hierarchical variants) or by focusing on the static pool management and eviction logic itself (in caching variants), unlike [Multi-Level Cache and Prefetching](4631-multi-level-cache-and-prefetching.md) which optimizes for temporal locality across multiple tiers, or [Raymarching with LOD-based Brick Caching](5776-raymarching-with-lod-based-brick-caching.md) which integrates caching with level-of-detail selection [S1, unofficial].

## Shared principles
The family shares the following mechanisms:

1.  **Multi-Resolution Brick Partitioning:** The volume is represented as a hierarchy consisting of multiple resolution levels. Each brick is a chunk of voxel data belonging to a specific resolution level [S1, unofficial].
2.  **Fixed Voxel Brick Size:** Each brick has a size in voxels (e.g., $32^3$) that is the same for all resolution levels [S1, unofficial]. This size is independent of the brick's spatial extent.
3.  **Spatial Extent Determination:** The spatial extent of a brick is determined solely by the resolution level it belongs to [S1, unofficial]. Unlike the geometric subdivision of a residency octree, the downsampling ratio between resolution levels in the bricked volume hierarchy is flexible [S1, unofficial].
4.  **GPU Cache Residency:** To access a brick's data on the GPU, it must be resident in a brick cache [S1, unofficial].
5.  **Boundary Padding:** Bricks are typically stored with 1-voxel padding on all sides to support trilinear interpolation at boundaries without accessing adjacent bricks [S1, unofficial][S2, unofficial]. The stored voxel count per brick is:
    $$ V_{brick} = (B_x+1)(B_y+1)(B_z+1) $$
    where $B_x, B_y, B_z$ are the logical brick dimensions [S1, unofficial].

## Subtopics

### [Brick Caching with LRU Replacement](879-brick-caching-with-lru-replacement.md)
Manages a fixed-size pool of volume data blocks, mapping brick coordinates to pool indices and evicting least-recently-used blocks when the pool is full. It solves the problem of limited GPU memory capacity and transfer bandwidth by ensuring that expensive data transfers from slower host memory are minimized. It sets itself apart from sibling methods by explicitly managing the cache lifecycle using a Last Recently Used (LRU) policy, often requiring CPU-side coordination due to the limitations of standard shader models regarding random memory writes. Unlike [Multi-Level Cache and Prefetching](4631-multi-level-cache-and-prefetching.md) which optimizes for temporal locality across multiple tiers, or [Raymarching with LOD-based Brick Caching](5776-raymarching-with-lod-based-brick-caching.md) which integrates caching with level-of-detail selection, this method focuses on the static pool management and eviction logic itself.

### [Brick-based Volume Raycasting](3341-brick-based-volume-raycasting.md)
Decomposes volume data into fixed-size, power-of-two bricks to achieve high CPU cache coherency without the memory overhead of spread memory layouts. Unlike linear raycasting, which suffers from view-dependent performance drops due to cache misses, this method provides nearly constant rendering performance regardless of viewing direction. It builds on [Brick-based storage and caching](4138-brick-based-storage-and-caching.md) by defining the specific ray-marching algorithm and addressing scheme required to traverse this one-level subdivision hierarchy efficiently.

### [N³-tree with Brick Pool and Node Pool](3407-n-tree-with-brick-pool-and-node-pool.md)
The N³-tree with Brick Pool and Node Pool is a hierarchical data structure for voxel rendering that combines an N³-tree for spatial subdivision with a brick pool for volume data and a node pool for tree structure, allowing efficient Level-of-Detail (LOD) and visibility culling. It differs from sibling methods like [Brick Caching with LRU Replacement](879-brick-caching-with-lru-replacement.md) by integrating the tree traversal logic directly into the GPU shader using a compact, cache-coherent node representation, and from [Bricked Volume Hierarchy](5036-bricked-volume-hierarchy.md) by using a single pointer per node to reduce memory overhead and improve cache coherence.

### [Multi-Level Cache and Prefetching](4631-multi-level-cache-and-prefetching.md)
Multi-Level Cache and Prefetching is a memory management strategy for web-based volume visualization that bridges the gap between persistent storage and high-speed rendering by utilizing a four-tier cache hierarchy and aggressive prefetching. It solves the problem of high-latency remote data fetching (order of 100 ms) in browser environments by ensuring that frequently accessed data slices and chunks are available in faster memory tiers, thereby hiding I/O latency during user interactions such as orbiting or slider playback. It differs from sibling methods like [Brick Caching with LRU Replacement](879-brick-caching-with-lru-replacement.md) by explicitly separating compressed and decompressed states across distinct memory ceilings and leveraging the browser's Origin Private File System (OPFS) for persistent storage, rather than relying solely on volatile GPU or CPU caches.

### [Bricked Volume Hierarchy](5036-bricked-volume-hierarchy.md)
The Bricked Volume Hierarchy is a data organization scheme where a volume dataset is partitioned into chunks called bricks across multiple resolution levels. It solves the problem of managing multi-resolution volume data by decoupling the logical resolution levels from the geometric subdivision of the space. What sets it apart from its siblings, such as the [N³-tree with Brick Pool and Node Pool](3407-n-tree-with-brick-pool-and-node-pool.md), is that the downsampling ratio between resolution levels is flexible, whereas the geometric subdivision in residency octrees is fixed.

### [Raymarching with LOD-based Brick Caching](5776-raymarching-with-lod-based-brick-caching.md)
Raymarching with LOD-based Brick Caching is a visualization method for Compressed Segmentation Volumes (CSVs) that renders volumes by decompressing bricks on-demand at the required Level of Detail (LOD) and storing them in a GPU cache. It solves the problem of rendering extremely large segmentation volumes by exploiting temporal coherency and detail separation, allowing coarse LOD data to remain on the GPU while fine LOD data is streamed from CPU memory. It differs from its siblings by using a specific cache structure of contiguous base elements rather than superblocks/columns, and by testing palettes for empty-space skipping before requesting cache locations.

### Cyclic Buffer for Smoke Volume
#### definition
- [S1, unofficial] A cyclic buffer structure used to store layers of smoke volume data, where the first layer (Index 0) is duplicated to the end to enable smooth transitions at the seeding step.
- [S1, unofficial] A buffer where the first layer is duplicated to the end to depict cyclic property, allowing overwriting of the oldest layer.

#### step
- [S1, unofficial] Each layer has an age relative to the time it was seeded. The oldest layer is overwritten next. When a layer at index 4 is overwritten, it becomes the youngest, and the seeding indicator moves one step forward.
- [S1, unofficial] To enable smooth transitions at the seeding step, the volumes of cubic cells between two layers are multiplied by a fade-in/out modifier based on the age of the cell.
- [S1, unofficial] Overwrite the oldest layer (indicated by an arrow) with new data, moving the seeding indicator one step forward. The overwritten layer becomes the youngest.
- [S1, unofficial] Multiply volumes of cubic cells between two layers by a fade-in/out modifier depending on cell age to enable smooth transitions at the seeding step.

### Homogeneous Brick Optimization
#### step
- [S2, unofficial] Nodes corresponding to homogeneous bricks (all voxels contain the same value up to a user-defined threshold) store the average value in the brick instead of a pointer to the brick's location in the brick cache.

#### pitfall
- [S2, unofficial] For multi-channel data, a node can only be marked homogeneous if its corresponding bricks in all channels are homogeneous.

## Background
Brick-based storage originated as a solution to the memory bandwidth and capacity limitations of early GPU architectures, particularly when rendering large volumetric datasets that exceeded the available VRAM [S1, unofficial]. The concept of decomposing volumes into fixed-size bricks allows for efficient caching and streaming, reducing the overhead associated with linear memory layouts which suffer from view-dependent cache misses [S1, unofficial]. The specific implementation of LRU caching for bricks was detailed in early works on real-time volume rendering, such as those by Brix et al., which introduced the use of count buffers for empty-space skipping and LRU replacement policies [S2, unofficial]. The N³-tree variant was developed to combine the benefits of hierarchical subdivision with the efficiency of brick-based caching, allowing for dynamic LOD selection and efficient traversal on the GPU [S1, unofficial]. More recent developments, such as Multi-Level Cache and Prefetching, have adapted these principles for web-based environments, utilizing browser-specific storage APIs like OPFS to handle high-latency network transfers [S1, unofficial]. The Cyclic Buffer variant was introduced specifically for smoke volume rendering to handle temporal data streams efficiently [S1, unofficial].

## Sources
[S1] Real-Time Smoke Volume Rendering, http://v22015081501727525.yourvserver.net/webpage/dirkfiles/betreutearbeiten/Sowinski_MA_2012.pdf, fetched 2026-10-31 (unofficial)
[S2] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
