# Brick-based storage and caching

Also called: Bricking, Brick Caching

## Overview
This family manages volume data by partitioning dense grids into fixed-size cubic blocks (bricks) to optimize memory hierarchy usage, enabling the rendering of datasets larger than physical memory [S1, S2, unofficial]. It sets itself apart from sibling methods by explicitly organizing data into these spatial bricks to improve cache coherency and support out-of-core rendering, whereas siblings like [Evaluation Cache](874-evaluation-cache.md) focus on temporal reuse of computed values and [Tile-based Volume Ray Marching](1678-tile-based-volume-ray-marching.md) focuses on view-dependent tile ordering without necessarily partitioning the underlying storage into fixed bricks [G].

## Shared principles
All brick-based implementations share the following mechanisms:

1.  **Domain Discretization into Bricks:** The continuous domain is partitioned into a grid of fixed-size cubic blocks called bricks. The number of bricks in each dimension is calculated as:
    $$ N_x = \lceil D_x / B_x \rceil, \quad N_y = \lceil D_y / B_y \rceil, \quad N_z = \lceil D_z / B_z \rceil $$
    where $D$ are volume dimensions and $B$ are brick dimensions [S1, S2, unofficial].
2.  **Boundary Padding:** To support trilinear interpolation at brick boundaries without accessing adjacent bricks, each brick is stored with a 1-voxel padding layer on all sides. The stored voxel count per brick is:
    $$ V_{brick} = (B_x+1)(B_y+1)(B_z+1) $$
    [S1, S2, unofficial].
3.  **Linear Memory Mapping:** Bricks are stored in a linear array in xyz-order. The index of brick $(i, j, k)$ is:
    $$ Index(i, j, k) = i + j \cdot N_x + k \cdot N_x \cdot N_y $$
    [S1, unofficial].
4.  **Cache Residency:** Data is loaded into a cache pool only when needed. The cache manages the mapping between logical brick coordinates and physical memory slots, evicting blocks when capacity is exceeded [S1, S2, unofficial].

## Subtopics

### [Brick Caching with LRU Replacement](879-brick-caching-with-lru-replacement.md)
Manages a fixed-size pool of volume data blocks, mapping logical brick coordinates to physical memory slots and evicting the least-recently-used blocks when the pool is full. It solves the problem of limited GPU memory capacity and transfer bandwidth by allowing rendering of volumes larger than physical memory, reducing rendering times from days to minutes for high-resolution datasets. It sets itself apart from sibling methods by explicitly maintaining a sorted list of blocks along the ray depth to facilitate immediate lookup, rather than relying solely on generic LRU eviction without spatial locality awareness. Unlike [Multi-Level Cache and Prefetching](4631-multi-level-cache-and-prefetching.md) which uses hierarchical prefetching, this method focuses on a single flat pool with strict LRU eviction.

### [Brick-based Volume Raycasting](3341-brick-based-volume-raycasting.md)
Brick-based volume raycasting decomposes a dense volume dataset into fixed-size cubic blocks (bricks) to achieve high cache coherency on CPU architectures without the memory overhead associated with spread memory layouts. Unlike linear raycasting, which suffers from severe view-dependent performance drops due to cache misses when traversing large datasets, this method provides nearly constant rendering performance regardless of viewing direction. It builds on [Brick-based storage and caching](4138-brick-based-storage-and-caching.md) by implementing a one-level subdivision hierarchy optimized for real-time brute-force raycasting.

### [N³-tree with Brick Pool and Node Pool](3407-n-tree-with-brick-pool-and-node-pool.md)
The N³-tree with Brick Pool and Node Pool is a hierarchical data structure for voxel rendering that combines an N³-tree for spatial subdivision with a brick pool for volume data and a node pool for tree structure, allowing efficient level-of-detail (LOD) and visibility culling. It differs from sibling methods like [Bricked Volume Hierarchy](5036-bricked-volume-hierarchy.md) by storing all tree nodes in a single 3D texture (the node pool) and using a single pointer per node to access children, rather than storing eight explicit child pointers and neighbor pointers. This design reduces memory requirements by 16x compared to Gobbetti et al. [GMAG08] and improves cache coherence during traversal.

### [Multi-Level Cache and Prefetching](4631-multi-level-cache-and-prefetching.md)
This method manages the loading and residency of volumetric data (specifically Gaussian splats in Zarr archives) by combining HTTP range requests with a four-tier caching hierarchy. It solves the problem of high-latency remote data fetching (order of 100 ms) in web-based volume visualization by ensuring frequently accessed data resides in fast memory or persistent storage. It differs from sibling caches like [Brick Caching with LRU Replacement](879-brick-caching-with-lru-replacement.md) by utilizing a strict tiered structure (L2/L1/L0/S-cache) with heap-aware dynamic resizing and explicit idle-time prefetching, rather than a single LRU pool.

### [Bricked Volume Hierarchy](5036-bricked-volume-hierarchy.md)
A data organization scheme where a volume dataset is partitioned into chunks called bricks across multiple resolution levels. It solves the problem of managing multi-resolution data for efficient GPU access by decoupling the logical resolution hierarchy from the geometric subdivision structure. It sets itself apart from its siblings by strictly separating the "bricked volume hierarchy" (data chunks) from the "residency octree" (geometric boundaries), ensuring that brick size in voxels remains fixed across all resolution levels.

### [Raymarching with LOD-based Brick Caching](5776-raymarching-with-lod-based-brick-caching.md)
This method renders Compressed Segmentation Volumes (CSVs) by raymarching with one ray per pixel, decompressing bricks on-demand at the required Level of Detail (LOD) and storing them in a GPU cache. It differs from sibling methods by exploiting temporal coherency with a one-frame decompression lag, using palette-based empty-space skipping before cache allocation, and employing a contiguous base-element cache structure without superblocks. It builds on [Brick-based storage and caching](4138-brick-based-storage-and-caching.md) by adding LOD-specific decompression and a hierarchical cache pool.

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
Brick-based storage originated as a solution to the memory bandwidth and capacity limitations of early GPU architectures when rendering large volumetric datasets. The concept of partitioning data into bricks to improve cache locality was popularized in the mid-2000s for real-time volume raycasting [S1, unofficial]. The specific implementation of LRU brick caching with count buffers for isovalue-based culling was detailed in later works focusing on CPU/GPU hybrid rendering [S2, unofficial]. The use of cyclic buffers for smoke volumes was introduced in academic theses focusing on fluid simulation visualization [S1, unofficial]. Homogeneous brick optimization was integrated into frameworks like Voreen to reduce cache pressure for uniform regions [S2, unofficial].

## Sources
[S1] Real-Time Smoke Volume Rendering, http://v22015081501727525.yourvserver.net/webpage/dirkfiles/betreutearbeiten/Sowinski_MA_2012.pdf, fetched 2026-10-31 (unofficial)
[S2] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
