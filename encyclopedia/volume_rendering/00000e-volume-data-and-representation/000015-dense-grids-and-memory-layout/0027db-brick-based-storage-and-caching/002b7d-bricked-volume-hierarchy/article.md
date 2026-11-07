# Bricked Volume Hierarchy
## Overview
A data organization scheme where a volume dataset is partitioned into chunks called bricks across multiple resolution levels [S1, unofficial]. It solves the problem of managing multi-resolution data for efficient GPU access by decoupling the logical resolution hierarchy from the geometric subdivision structure [S1, unofficial]. It sets itself apart from its siblings by strictly separating the "bricked volume hierarchy" (data chunks) from the "residency octree" (geometric boundaries), ensuring that brick size in voxels remains fixed across all resolution levels [S1, unofficial].
## Shared principles
The Bricked Volume Hierarchy family shares the following mechanisms:

1.  **Multi-Resolution Brick Partitioning:** The volume is represented as a hierarchy consisting of multiple resolution levels. Each brick is a chunk of voxel data belonging to a specific resolution level [S1, unofficial].
2.  **Fixed Voxel Brick Size:** Each brick has a size in voxels (e.g., $32^3$) that is the same for all resolution levels [S1, unofficial]. This size is independent of the brick's spatial extent.
3.  **Spatial Extent Determination:** The spatial extent of a brick is determined solely by the resolution level it belongs to [S1, unofficial]. Unlike the geometric subdivision of a residency octree, the downsampling ratio between resolution levels in the bricked volume hierarchy is flexible [S1, unofficial].
4.  **GPU Cache Residency:** To access a brick's data on the GPU, it must be resident in a brick cache [S1, unofficial].

## Subtopics

### [Brick Caching with LRU Replacement](879-brick-caching-with-lru-replacement.md)
Manages the residency of bricks in the brick cache using Least Recently Used replacement. Choose when memory capacity is limited and access patterns are temporal.

### [Brick-based Volume Raycasting](3341-brick-based-volume-raycasting.md)
The rendering technique that traverses the hierarchy to sample bricks. Choose for direct volume rendering on GPUs.

### [N³-tree with Brick Pool and Node Pool](3407-n-tree-with-brick-pool-and-node-pool.md)
A specific implementation of the geometric subdivision (residency octree) that uses pools for memory management. Choose when dynamic spatial subdivision is required.

### [Multi-Level Cache and Prefetching](4631-multi-level-cache-and-prefetching.md)
Optimizes the brick cache with multiple levels and prefetching strategies. Choose for high-performance rendering with predictable access patterns.

### [Raymarching with LOD-based Brick Caching](5776-raymarching-with-lod-based-brick-caching.md)
Combines raymarching with Level-of-Detail selection based on brick distance. Choose for interactive navigation where detail varies with view distance.

## Background
The concept was introduced in the thesis "Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze" [S1, unofficial]. It distinguishes between the resolution levels of a volume and the spatial, purely geometric subdivision determined by the residency octree [S1, unofficial]. The work builds on previous research, such as that by Hadwiger et al. [HBJP12], regarding flexible downsampling ratios [S1, unofficial].

## Sources
[S1] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
