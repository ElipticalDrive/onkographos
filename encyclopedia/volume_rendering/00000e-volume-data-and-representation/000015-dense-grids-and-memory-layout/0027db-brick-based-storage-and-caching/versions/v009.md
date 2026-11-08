# Brick-based storage and caching

Also called: Bricking, Brick Caching, Multi-Resolution Brick Partitioning

## Overview
Brick-based storage and caching partitions a volumetric dataset into fixed-size sub-volumes (bricks) to manage memory constraints and optimize data access patterns for rendering [S1, S2, S3, unofficial]. It solves the problem of limited GPU texture memory and bandwidth by allowing the system to stream only the necessary bricks into a resident cache, often combined with level-of-detail (LOD) strategies to reduce sampling costs [S1, S2, S3, S4, unofficial]. It sets itself apart from sibling methods by explicitly decoupling the logical resolution levels from the geometric subdivision of space, using fixed-size voxel bricks that are managed via a cache hierarchy, whereas [Tile-based Volume Ray Marching](1678-tile-based-volume-ray-marching.md) operates on 2D screen tiles and [Chunk-based Voxel Data Structure](1818-chunk-based-voxel-data-structure.md) typically implies a different structural hierarchy or memory layout optimization [S2, S4, unofficial].

## Shared principles
The following mechanisms and terms are common to all child methods in this family:

1.  **Multi-Resolution Brick Partitioning:** The volume is represented as a hierarchy consisting of multiple resolution levels. Each brick is a chunk of voxel data belonging to a specific resolution level [S1, unofficial].
2.  **Fixed Voxel Brick Size:** Each brick has a size in voxels (e.g., $32^3$) that is the same for all resolution levels [S1, unofficial]. This size is independent of the brick's spatial extent.
3.  **Spatial Extent Determination:** The spatial extent of a brick is determined solely by the resolution level it belongs to [S1, unofficial]. Unlike the geometric subdivision of a residency octree, the downsampling ratio between resolution levels in the bricked volume hierarchy is flexible [S1, unofficial].
4.  **GPU Cache Residency:** To access a brick's data on the GPU, it must be resident in a brick cache [S1, unofficial].
5.  **Boundary Padding:** Bricks are typically stored with 1-voxel padding on all sides to support trilinear interpolation at boundaries without accessing adjacent bricks [S1, unofficial][S2, unofficial]. The stored voxel count per brick is:
    $$ V_{brick} = (B_x+1)(B_y+1)(B_z+1) $$
    where $B_x, B_y, B_z$ are the logical brick dimensions [S1, unofficial].

## Subtopics

### [Brick Caching with LRU Replacement](879-brick-caching-with-lru-replacement.md)
Manages a fixed-size pool of volume data blocks, mapping brick coordinates to pool indices and evicting least-recently-used blocks when the pool is full. It solves the problem of limited GPU memory capacity and transfer bandwidth by ensuring that expensive data transfers from slower host memory are minimized. It sets itself apart from sibling methods by explicitly managing the cache lifecycle using a Last Recently Used (LRU) policy, often requiring CPU-side coordination due to the limitations of standard shader models regarding random memory writes. Unlike [Multi-Level Cache and Prefetching](4631-multi-level-cache-and-prefetching.md) which focuses on prefetching, or [Raymarching with LOD-based Brick Caching](5776-raymarching-with-lod-based-brick-caching.md) which integrates caching with LOD selection, this method focuses on the core eviction logic and memory pool management.

### [Bricking](914-bricking.md)
Bricking is a memory management technique for volume rendering that partitions a volumetric dataset larger than the available GPU memory (or texture unit limits) into smaller, fixed-size sub-volumes called "bricks". Each brick is loaded into GPU memory, rendered independently, and the resulting images are composited to form the final image. It differs from [Chunk-based Voxel Data Structure](1818-chunk-based-voxel-data-structure.md) by typically using uniform, fixed-size blocks optimized for texture sampling, and from [Tile-based Volume Ray Marching](1678-tile-based-volume-ray-marching.md) by operating on 3D spatial subdivisions rather than 2D screen tiles.

### [Level-of-Detail Volume Rendering via 3D Textures](1706-level-of-detail-volume-rendering-via-3d-textures.md)
This method provides an adaptive level-of-detail (LOD) selection algorithm for multi-resolution volume rendering using 3D texture mapping. It renders volume data in a region-of-interest at high resolution while progressively lowering the resolution for areas further away, enabling interactive exploration of large-scale datasets. Unlike approaches that rely on user-specified error tolerances or spherical shells, this technique guarantees consistent interpolation between different resolution levels and explicitly fixes rendering artifacts (such as light/dark bands) at brick boundaries using polygon clipping and opacity correction.

### [Brick-based Volume Raycasting](3341-brick-based-volume-raycasting.md)
Brick-based volume raycasting decomposes volume data into fixed-size, power-of-two bricks to achieve high CPU cache coherency without the memory overhead of spread memory layouts. Unlike linear raycasting, which suffers from view-dependent performance drops due to cache misses, this method provides nearly constant rendering performance regardless of viewing direction. It builds on [Brick-based storage and caching](4138-brick-based-storage-and-caching.md) by defining the specific ray-marching algorithm and addressing scheme required to traverse this one-level subdivision hierarchy efficiently.

### [N³-tree with Brick Pool and Node Pool](3407-n-tree-with-brick-pool-and-node-pool.md)
The N³-tree with Brick Pool and Node Pool is a hierarchical data structure for voxel rendering that combines an N³-tree for spatial subdivision with a brick pool for volume data and a node pool for tree structure, allowing efficient Level-of-Detail (LOD) and visibility culling. It differs from sibling methods like [Brick Caching with LRU Replacement](879-brick-caching-with-lru-replacement.md) by integrating the tree traversal logic directly into the GPU shader using a compact, cache-coherent node representation, and from [Bricked Volume Hierarchy](5036-bricked-volume-hierarchy.md) by using a single pointer per node to reduce memory overhead and improve cache coherence.

### [Multi-Level Cache and Prefetching](4631-multi-level-cache-and-prefetching.md)
Multi-Level Cache and Prefetching is a memory management strategy for web-based volume visualization that bridges the gap between persistent storage and high-speed rendering by utilizing a four-tier cache hierarchy and aggressive prefetching. It solves the problem of high-latency remote data fetching (order of 100 ms) in browser environments by ensuring that frequently accessed data slices and chunks are available in faster memory tiers, thereby hiding I/O latency during user interactions such as orbiting or slider playback. It differs from sibling methods like [Brick Caching with LRU Replacement](879-brick-caching-with-lru-replacement.md) by explicitly separating compressed and decompressed states across distinct memory ceilings and leveraging the browser's Origin Private File System (OPFS) for persistent storage, rather than relying solely on volatile GPU or CPU caches.

### [Bricked Volume Hierarchy](5036-bricked-volume-hierarchy.md)
The Bricked Volume Hierarchy is a data organization scheme where a volume dataset is partitioned into chunks called bricks across multiple resolution levels. It solves the problem of managing multi-resolution volume data by decoupling the logical resolution levels from the geometric subdivision of the space. What sets it apart from its siblings, such as the [N³-tree with Brick Pool and Node Pool](3407-n-tree-with-brick-pool-and-node-pool.md), is that the downsampling ratio between resolution levels is flexible, whereas the geometric subdivision in residency octrees is fixed.

### [Raymarching with LOD-based Brick Caching](5776-raymarching-with-lod-based-brick-caching.md)
Raymarching with LOD-based Brick Caching is a visualization method for Compressed Segmentation Volumes (CSVs) that renders volumes by decompressing bricks on-demand at the required Level of Detail (LOD) and storing them in a GPU cache. It solves the problem of rendering extremely large segmentation volumes by exploiting temporal coherency and detail separation, allowing coarse LOD data to remain on the GPU while fine LOD data is streamed from CPU memory. It differs from its siblings by using a specific cache structure of contiguous base elements rather than superblocks/columns, and by testing palettes for empty-space skipping before requesting cache locations.

#### Cyclic Buffer for Smoke Volume
- **Definition:** A cyclic buffer structure used to store layers of smoke volume data, where the first layer (Index 0) is duplicated to the end to enable smooth transitions at the seeding step [S1, unofficial]. A buffer where the first layer is duplicated to the end to depict cyclic property, allowing overwriting of the oldest layer [S1, unofficial].
- **Steps:** Each layer has an age relative to the time it was seeded. The oldest layer is overwritten next. When a layer at index 4 is overwritten, it becomes the youngest, and the seeding indicator moves one step forward [S1, unofficial]. To enable smooth transitions at the seeding step, the volumes of cubic cells between two layers are multiplied by a fade-in/out modifier based on the age of the cell [S1, unofficial]. Overwrite the oldest layer (indicated by an arrow) with new data, moving the seeding indicator one step forward. The overwritten layer becomes the youngest [S1, unofficial]. Multiply volumes of cubic cells between two layers by a fade-in/out modifier depending on cell age to enable smooth transitions at the seeding step [S1, unofficial].

#### Homogeneous Brick Optimization
- **Steps:** Nodes corresponding to homogeneous bricks (all voxels contain the same value up to a user-defined threshold) store the average value in the brick instead of a pointer to the brick's location in the brick cache [S2, unofficial].
- **Pitfalls:** For multi-channel data, a node can only be marked homogeneous if its corresponding bricks in all channels are homogeneous [S2, unofficial].

## Background
Bricking was introduced to overcome hardware limitations regarding texture memory size and resolution limits (e.g., $512^3$ textures or 256MB video memory on older GPUs like the NVIDIA GeForce 6) [S1, unofficial]. Early implementations used an oldest-out-first swapping strategy for supervolumes that did not fit in GPU memory [S2, unofficial]. The technique evolved to include multi-resolution hierarchies and LOD-based caching to handle increasingly large datasets in interactive applications [S1, S2, S3, S4, unofficial]. It is widely used in scientific visualization, medical imaging, and real-time rendering engines where data exceeds available VRAM [S1, S2, S3, S4, unofficial].

## Sources
[S1] Real-Time Smoke Volume Rendering, http://v22015081501727525.yourvserver.net/webpage/dirkfiles/betreutearbeiten/Sowinski_MA_2012.pdf, fetched 2026-10-31 (unofficial)
[S2] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
