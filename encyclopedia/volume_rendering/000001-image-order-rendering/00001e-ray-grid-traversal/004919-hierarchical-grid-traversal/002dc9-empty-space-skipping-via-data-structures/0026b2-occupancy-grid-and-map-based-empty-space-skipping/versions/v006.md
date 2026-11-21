# Occupancy Grid and Map-Based Empty Space Skipping

Also called: Occupancy Grid Acceleration, Map-Based Skipping

## Overview
This family of methods accelerates volumetric ray casting by using a secondary, lower-resolution data structure (typically a grid or map) to identify inactive regions of the volume, allowing rays to skip sampling points in empty space [G]. Unlike hierarchical tree structures or distance-field-based approaches, these methods rely on direct indexing into a flat or multi-resolution occupancy grid to determine entry and exit points of active regions.

## Shared principles
All methods in this family share the core mechanism of using a secondary data structure to represent the spatial distribution of active volume data, allowing rays to skip over inactive regions without sampling the primary volume texture [G].

1.  **Acceleration Structure:** A secondary data structure (e.g., occupancy grid, min-max block map, BVH, or hash table) is constructed to represent the volume. Each element in this structure corresponds to a block or region of volume voxels and stores metadata indicating occupancy or activity [G].
2.  **Active Region Definition:** A block or region is defined as "active" if it contributes to the final image. This is typically determined by density, opacity, or scalar value thresholds (e.g., $\max(\text{block}) > \tau$) [G].
3.  **Ray Query and Skip:** During rendering, the algorithm queries the acceleration structure to find entry and exit points of active regions. Rays advance past inactive blocks without sampling, and perform standard integration only within active segments [G].
4.  **Coordinate Mapping:** Efficient mapping between ray positions and grid indices is required, typically via division by block size: $u_{\text{occ}} = \lfloor u_{\text{vol}} / B \rfloor$ [G].

## Subtopics

### [Distance Map Acceleration](972-distance-map-acceleration.md)
Distance Map Acceleration uses a precomputed Chebyshev distance map derived from an occupancy map to skip sampling points in empty regions. It encodes the *distance* to the nearest occupied region, enabling larger, more informed skip steps than binary occupancy grids. It builds on [Occupancy Grid and Map-Based Empty Space Skipping](3852-occupancy-grid-and-map-based-empty-space-skipping.md) by adding a distance field to the occupancy data.

### [Two-stage Ray Marching with Empty-Space Skipping](2985-two-stage-ray-marching-with-empty-space-skipping.md)
Two-stage Ray Marching decouples the search for active data from density sampling using a flat occupancy map and fixed-step skipping. Stage 1 performs coarse occupancy lookups to skip inactive blocks, while Stage 2 performs fine-grained sampling only within active regions. It differs from hierarchical methods by using a flat map and from Distance Map Acceleration by relying on binary flags rather than distance metrics.

### [Hierarchical Sampling with Occupancy Grid](3035-hierarchical-sampling-with-occupancy-grid.md)
This method combines empty-space skipping via an occupancy grid with importance-based hierarchical sampling to accelerate NeRF training and inference. It decouples coarse sampling (guided by spatial occupancy) from fine sampling (guided by rendered weights), allowing efficient refinement of high-density regions without re-evaluating empty space.

### [Empty Space Filtering (Training-Time Bounding Box)](6752-empty-space-filtering-training-time-bounding-box.md)
This is a training-time acceleration strategy that restricts optimization to the scene's active geometry using a coarse deformation feature grid. It identifies inactive regions using alpha values and filters out sampled points falling within these regions, reducing queries to the light-weight MLP during training. It operates on the training data sampling pipeline rather than ray traversal logic [G].

### [MERF Real-Time Rendering](10175-merf-real-time-rendering.md)
MERF Real-Time Rendering uses a multi-resolution occupancy grid hierarchy for empty space skipping in WebGL-based NeRF synthesis. It prioritizes browser portability and memory efficiency by combining coarse-to-fine grid queries with deferred appearance sampling, achieving real-time frame rates on consumer hardware.

### Ray Marching with Occupancy Grid (Instant-NGP style)
#### definition
An acceleration strategy that maintains an occupancy grid marking voxels as empty or non-empty to prevent sampling in empty spaces, updated periodically based on SDF values [S1].
#### step
- Maintain an occupancy grid that marks each voxel grid as empty or non-empty to guide the marching process [S1].
- Periodically update the occupancy grid based on the SDF value predicted by the model [S1].
- Update the occupancy grid using the calculated density value in the same manner as Instant-NGP [S1].
- Sample 90% of rays on foreground pixels and 10% of rays on background pixels to achieve faster convergence [S1].
#### parameter
- Foreground ray sampling ratio: 90% [S1].
- Background ray sampling ratio: 10% [S1].

## Background
The concept of using occupancy grids for empty space skipping in volume rendering is foundational to many modern neural rendering techniques. The specific application of maintaining an occupancy grid updated based on SDF values for ray marching acceleration is described in [S1], which cites Instant-NGP [37] as the origin of this specific update strategy. The method was applied to fast learning of neural implicit surfaces for multi-view reconstruction.

## Sources
[S1] NeuS2: Fast Learning of Neural Implicit Surfaces for Multi-view Reconstruction, https://doi.org/10.1109/iccv51070.2023.00305, fetched 2026-10-26
