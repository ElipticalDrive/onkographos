# Brick-based storage and caching
Also called: bricking, brick caching, brick-based rendering

## Overview
Brick-based storage and caching decomposes a dense volume dataset into fixed-size cubic blocks (bricks) to improve memory locality, enable out-of-core rendering of datasets larger than physical memory, and support multi-resolution hierarchies [S1, unofficial][S2, unofficial]. It sets itself apart from flat linear layouts by eliminating view-dependent performance drops through cache-coherent access patterns and from pure spatial indexing structures by decoupling the logical data partitioning from the geometric traversal hierarchy [S1, unofficial][S5036, unofficial].

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
Manages a fixed-size pool of volume data blocks, mapping logical brick coordinates to physical memory slots and evicting the least-recently-used blocks when the pool is full. It solves the problem of limited GPU memory capacity and transfer bandwidth by allowing rendering of volumes larger than physical memory, reducing rendering times from days to minutes for high-resolution datasets. It sets itself apart from sibling methods by explicitly maintaining a sorted list of blocks along the ray depth to facilitate immediate lookup, rather than relying solely on generic LRU eviction without spatial locality awareness. Unlike [Multi-Level Cache and Prefetching](4631-multi-level-cache-and-prefetching.md) which uses hierarchical prefetching, this method focuses on a single flat pool with strict LRU eviction.

### [Brick-based Volume Raycasting](3341-brick-based-volume-raycasting.md)
Decomposes a dense volume dataset into fixed-size cubic blocks (bricks) to achieve high cache coherency on CPU architectures without the memory overhead associated with spread memory layouts. Unlike linear raycasting, which suffers from severe view-dependent performance drops due to cache misses when traversing large datasets, this method provides nearly constant rendering performance regardless of viewing direction. It builds on [Brick-based storage and caching](4138-brick-based-storage-and-caching.md) by implementing a one-level subdivision hierarchy optimized for real-time brute-force raycasting.

### [N³-tree with Brick Pool and Node Pool](3407-n-tree-with-brick-pool-and-node-pool.md)
The N³-tree with Brick Pool and Node Pool is a hierarchical data structure for voxel rendering that combines an N³-tree for spatial subdivision with a brick pool for volume data and a node pool for tree structure, allowing efficient level-of-detail (LOD) and visibility culling. It differs from sibling methods like [Bricked Volume Hierarchy](5036-bricked-volume-hierarchy.md) by storing all tree nodes in a single 3D texture (the node pool) and using a single pointer per node to access children, rather than storing eight explicit child pointers and neighbor pointers. This design reduces memory requirements by 16x compared to Gobbetti et al. [GMAG08] and improves cache coherence during traversal.

### [Multi-Level Cache and Prefetching](4631-multi-level-cache-and-prefetching.md)
This method manages the loading and residency of volumetric data (specifically Gaussian splats in Zarr archives) by combining HTTP range requests with a four-tier caching hierarchy. It solves the problem of high-latency remote data fetching (order of 100 ms) in web-based volume visualization by ensuring frequently accessed data resides in fast memory or persistent storage. It differs from sibling caches like [Brick Caching with LRU Replacement](879-brick-caching-with-lru-replacement.md) by utilizing a strict tiered structure (L2/L1/L0/S-cache) with heap-aware dynamic resizing and explicit idle-time prefetching, rather than a single LRU pool.

### [Bricked Volume Hierarchy](5036-bricked-volume-hierarchy.md)
A data organization scheme where a volume dataset is partitioned into chunks called bricks across multiple resolution levels. It solves the problem of managing multi-resolution data for efficient GPU access by decoupling the logical resolution hierarchy from the geometric subdivision structure. It sets itself apart from its siblings by strictly separating the "bricked volume hierarchy" (data chunks) from the "residency octree" (geometric boundaries), ensuring that brick size in voxels remains fixed across all resolution levels.

### [Raymarching with LOD-based Brick Caching](5776-raymarching-with-lod-based-brick-caching.md)
This method renders Compressed Segmentation Volumes (CSVs) by raymarching with one ray per pixel, decompressing bricks on-demand at the required Level of Detail (LOD) and storing them in a GPU cache. It differs from sibling methods by exploiting temporal coherency with a one-frame decompression lag, using palette-based empty-space skipping before cache allocation, and employing a contiguous base-element cache structure without superblocks. It builds on [Brick-based storage and caching](4138-brick-based-storage-and-caching.md) by adding LOD-specific decompression and a hierarchical cache pool.

### Cyclic Buffer for Smoke Volume
A cyclic buffer structure used to store layers of smoke volume data, where the first layer (Index 0) is duplicated to the end to enable smooth transitions at the seeding step [S1, unofficial]. Each layer has an age relative to the time it was seeded. The oldest layer is overwritten next. When a layer at index 4 is overwritten, it becomes the youngest, and the seeding indicator moves one step forward. To enable smooth transitions at the seeding step, the volumes of cubic cells between two layers are multiplied by a fade-in/out modifier based on the age of the cell [S1, unofficial].

### Homogeneous Brick Optimization
Nodes corresponding to homogeneous bricks (all voxels contain the same value up to a user-defined threshold) store the average value in the brick instead of a pointer to the brick's location in the brick cache [S2, unofficial]. For multi-channel data, a node can only be marked homogeneous if its corresponding bricks in all channels are homogeneous [S2, unofficial].

## Background
Brick-based rendering originated in the early 2000s as a solution to the memory bandwidth and cache coherence limitations of direct volume rendering on large datasets. Key early works include the "Bricked Volume Hierarchy" by Gobbetti et al. [GMAG08], which introduced the concept of decoupling data bricks from geometric traversal, and subsequent optimizations for CPU and GPU architectures. The technique has been widely adopted in medical imaging (e.g., Voreen [DLJL22]) and scientific visualization to handle terabyte-scale datasets. Recent developments integrate brick-based caching with web-based rendering (e.g., Viv [MGP+22]) and compressed data formats like Zarr and CSVs to support real-time interactive exploration over networks.

## Sources
[S1] Real-Time Smoke Volume Rendering, http://v22015081501727525.yourvserver.net/webpage/dirkfiles/betreutearbeiten/Sowinski_MA_2012.pdf, fetched 2026-10-31 (unofficial)
[S2] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
