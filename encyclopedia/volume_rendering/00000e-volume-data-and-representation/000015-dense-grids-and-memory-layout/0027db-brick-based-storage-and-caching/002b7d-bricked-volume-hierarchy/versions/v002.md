# Bricked Volume Hierarchy


## Overview
A data organization scheme where a volume dataset is partitioned into chunks called bricks across multiple resolution levels [S1, unofficial]. It solves the problem of managing multi-resolution data for efficient GPU access by decoupling the logical resolution levels from the geometric subdivision of the spatial hierarchy [S1, unofficial]. It sets itself apart from sibling methods by strictly separating the "bricked volume hierarchy" (which defines data chunks and resolution) from the "residency octree" (which defines spatial subdivision and cache residency) [S1, unofficial].

## Shared principles
The Bricked Volume Hierarchy family relies on the following mechanisms and constraints, which are common to all children:

1.  **Multi-Resolution Partitioning:** The volume is represented as a hierarchy of multiple resolution levels. Each level consists of bricks, which are chunks of voxel data [S1, unofficial].
2.  **Fixed Brick Size:** Each brick has a fixed size in voxels (e.g., $32^3$) that is identical across all resolution levels [S1, unofficial]. This size is independent of the brick's spatial extent.
3.  **Spatial Extent Determination:** The spatial extent of a brick is determined solely by the resolution level it belongs to, not by the brick's fixed voxel count [S1, unofficial].
4.  **GPU Cache Requirement:** To access a brick's data on the GPU, the brick must be resident in a brick cache [S1, unofficial].

## Subtopics

### [Brick Caching with LRU Replacement](879-brick-caching-with-lru-replacement.md)
Manages the residency of bricks in the GPU cache using a Least Recently Used policy.

### [Brick-based Volume Raycasting](3341-brick-based-volume-raycasting.md)
The rendering algorithm that traverses the hierarchy to sample voxel data.

### [N³-tree with Brick Pool and Node Pool](3407-n-tree-with-brick-pool-and-node-pool.md)
A specific implementation of the spatial subdivision hierarchy (residency octree) using pooled memory structures.

### [Multi-Level Cache and Prefetching](4631-multi-level-cache-and-prefetching.md)
Optimizes data transfer by managing cache levels and predicting future brick needs.

### [Raymarching with LOD-based Brick Caching](5776-raymarching-with-lod-based-brick-caching.md)
Combines raymarching with Level-of-Detail selection to cache bricks based on viewing distance.

## Background
The concept was introduced in the thesis "Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze" (Scalable Web-Based Volume Rendering of Large Biomedical Datasets) [S1, unofficial]. The work distinguishes between the resolution levels of a volume and the spatial, purely geometric subdivision determined by the residency octree [S1, unofficial]. While the downsampling ratio between resolution levels in the bricked volume hierarchy is flexible, the spatial subdivision determined by the octree is not; at each subdivision level in the octree, each dimension is halved such that each node in the tree has eight children [S1, unofficial].

## Sources
[S1] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
