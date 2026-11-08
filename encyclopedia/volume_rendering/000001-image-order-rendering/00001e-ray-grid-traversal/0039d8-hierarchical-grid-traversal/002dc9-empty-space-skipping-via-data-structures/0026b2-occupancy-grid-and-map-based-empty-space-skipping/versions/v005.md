# Occupancy Grid and Map-Based Empty Space Skipping


## Overview
This family of methods accelerates volumetric ray casting by using a secondary, lower-resolution grid (occupancy map) to identify inactive regions of the volume, allowing rays to skip sampling points in empty space [S1, unofficial]. It differs from hierarchical methods by using a flat, regular grid structure, and from distance-based methods by relying on binary occupancy flags rather than distance-to-nearest-occupied-voxel metrics [G].

## Shared principles
All methods in this family share the core mechanism of using a secondary data structure to represent the spatial distribution of active volume data, allowing rays to skip over inactive regions without sampling the primary volume texture [G].

1. **Acceleration Structure:** A regular grid (occupancy map) is constructed to represent the volume, where each element corresponds to a block of volume voxels and stores a binary flag indicating occupancy [G].
2. **Active Region Definition:** A block is defined as active if it contributes to the final image, typically determined by density or opacity thresholds [G].
3. **Ray Query and Skip:** During rendering, the algorithm queries the grid to find entry and exit points of active regions. Rays advance past inactive blocks without sampling, and perform standard integration within active segments [G].
4. **Coordinate Mapping:** Efficient mapping between ray positions and grid indices is required, typically via division by block size: $u_{\text{occ}} = u_{\text{vol}} / B$ [G].

## Subtopics

### [Distance Map Acceleration](972-distance-map-acceleration.md)
Uses a precomputed Chebyshev distance map derived from an occupancy map to skip sampling points in empty regions, enabling larger, more informed skip steps based on the distance to the nearest occupied region.

### [Two-stage Ray Marching with Empty-Space Skipping](2985-two-stage-ray-marching-with-empty-space-skipping.md)
Decouples the search for active data from density sampling using a strict two-phase loop: Stage 1 performs coarse occupancy lookups to skip inactive blocks, while Stage 2 performs fine-grained density sampling only within confirmed active regions.

### [Hierarchical Sampling with Occupancy Grid](3035-hierarchical-sampling-with-occupancy-grid.md)
Combines empty-space skipping via an occupancy grid with importance-based hierarchical sampling to accelerate Neural Radiance Field (NeRF) training and inference, decoupling coarse sampling (guided by spatial occupancy) from fine sampling (guided by rendered weights).

### [Empty Space Filtering (Training-Time Bounding Box)](6752-empty-space-filtering-training-time-bounding-box.md)
A preprocessing and runtime optimization for dynamic neural volume rendering that constructs a coarse geometry prior (bounding box) from alpha values across training time steps to filter out sampled points in empty regions.

### [MERF Real-Time Rendering](10175-merf-real-time-rendering.md)
A real-time rendering implementation as a Javascript 3D (three.js) web application, orchestrated by a single GLSL fragment shader, using a multi-resolution hierarchy of occupancy grids for efficient ray marching.

### Ray Marching with Occupancy Grid (Instant-NGP style)
An acceleration strategy that maintains an occupancy grid marking voxels as empty or non-empty to prevent sampling in empty spaces, updated periodically based on SDF values [S1, unofficial]. It maintains an occupancy grid that marks each voxel grid as empty or non-empty to guide the marching process, periodically updates the occupancy grid based on the SDF value predicted by the model, and updates the occupancy grid using the calculated density value in the same manner as Instant-NGP [S1, unofficial]. It samples 90% of rays on foreground pixels and 10% of rays on background pixels to achieve faster convergence [S1, unofficial].

## Background
The method is widely used in real-time volumetric rendering and, more recently, in the training and inference of Neural Radiance Fields (NeRFs) to reduce computational load in empty space [S1, unofficial]. NeuS2 adopts a ray marching acceleration strategy used in Instant-NGP, maintaining an occupancy grid that roughly marks each voxel grid as empty or non-empty to guide the marching process and prevent sampling in empty spaces [S1, unofficial]. The occupancy grid is periodically updated based on the SDF value predicted by the model, using a logistic density distribution to calculate density values for each grid [S1, unofficial].

## Sources
[S1] NeuS2: Fast Learning of Neural Implicit Surfaces for Multi-view Reconstruction, https://doi.org/10.1109/iccv51070.2023.00305, fetched 2026-10-26 (unofficial)
