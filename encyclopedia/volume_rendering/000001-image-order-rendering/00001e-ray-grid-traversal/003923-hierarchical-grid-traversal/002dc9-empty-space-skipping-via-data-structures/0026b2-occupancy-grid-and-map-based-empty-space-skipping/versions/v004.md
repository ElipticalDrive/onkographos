# Occupancy Grid and Map-Based Empty Space Skipping

Also called: Occupancy Grid Acceleration, Block-Based Skipping, Grid-Based Empty Space Skipping

## Overview
This family of methods accelerates volumetric ray casting by using a secondary, lower-resolution grid (occupancy map) to identify inactive regions of the volume, allowing rays to skip sampling points in empty space [S1][S2, unofficial]. It differs from hierarchical methods like [Space Leaping](5863-space-leaping.md) by using a flat, regular grid structure, and from [Distance Map Acceleration](972-distance-map-acceleration.md) by relying on binary occupancy flags rather than distance-to-nearest-occupied-voxel metrics [S1][S2, unofficial].

## Shared principles
All methods in this family share the core mechanism of using a secondary data structure to represent the spatial distribution of active volume data, allowing rays to skip over inactive regions without sampling the primary volume texture.

1.  **Acceleration Structure:** A data structure (e.g., occupancy grid, octree, BVH, hash map) is constructed to represent the volume. Each element in this structure corresponds to a region of the volume and stores metadata indicating whether that region contains "active" data (e.g., density above a threshold, non-zero opacity) [S1][S2, unofficial].
2.  **Active Region Definition:** A region is defined as active if it contributes to the final image. Common criteria include:
    *   Maximum voxel value exceeding an isovalue threshold $\tau$ [S1].
    *   Presence of non-zero opacity after transfer function application [S1][S2, unofficial].
    *   Density values above a specific threshold $\tau_1$ [S1].
3.  **Ray Query and Skip:** During rendering, for each ray, the algorithm queries the acceleration structure to find the entry and exit points of active regions.
    *   If a ray segment lies entirely within an inactive region indicated by the structure, the ray is advanced to the next active region boundary without sampling the volume [S1][S2, unofficial].
    *   If a ray segment intersects an active region, standard ray marching or integration is performed within that segment [S1][S2, unofficial].
4.  **Coordinate Mapping:** Efficient mapping between ray positions in world/voxel space and indices in the acceleration structure is required. For regular grids, this is often a simple division by block size: $u_{\text{occ}} = u_{\text{vol}} / B$ [S1].

## Subtopics

### [Distance Map Acceleration](972-distance-map-acceleration.md)
Uses a precomputed Chebyshev distance map derived from an occupancy map to skip sampling points in empty regions, enabling larger, more informed skip steps based on the distance to the nearest occupied region.

### [Two-stage Ray Marching with Empty-Space Skipping](2985-two-stage-ray-marching-with-empty-space-skipping.md)
Decouples the search for active data from density sampling using a strict two-phase loop: Stage 1 performs coarse occupancy lookups to skip inactive blocks, while Stage 2 performs fine-grained density sampling only within confirmed active regions.

### [Hierarchical Sampling with Occupancy Grid](3035-hierarchical-sampling-with-occupancy-grid.md)
Combines empty-space skipping via an occupancy grid with importance-based hierarchical sampling to accelerate Neural Radiance Field (NeRF) training and inference, decoupling coarse sampling (guided by spatial occupancy) from fine sampling (guided by rendered weights).

### [Empty Space Filtering (Training-Time Bounding Box)](6752-empty-space-filtering-training-time-bounding-box.md)
A preprocessing and runtime optimization for dynamic neural volume rendering that constructs a coarse geometry prior (bounding box) from alpha values across training time steps to filter out sampled points in empty regions.

## Background
The concept of using occupancy grids for empty space skipping in volume rendering was introduced by Levoy in 1990 [S1]. It is widely used in real-time volumetric rendering, isosurface extraction, and more recently in the training and inference of Neural Radiance Fields (NeRFs) to reduce computational load in empty space [S1, unofficial].

## Sources
[S1] Levoy, M. "Display of images from three-dimensional medical scans." SIGGRAPH, 1990.
[S2] Unofficial implementation notes and community discussions regarding occupancy grid acceleration in volume rendering and NeRFs. (unofficial)
