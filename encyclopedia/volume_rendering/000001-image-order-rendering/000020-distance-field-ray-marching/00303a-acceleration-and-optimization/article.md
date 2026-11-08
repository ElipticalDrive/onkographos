# Acceleration and Optimization
## Overview
This family encompasses techniques to reduce the computational cost of distance-field ray marching and sparse volume rendering, primarily by minimizing the number of expensive Signed Distance Function (SDF) evaluations or traversal steps [S1, unofficial][S2, unofficial]. Unlike [Beam Optimization](932-beam-optimization.md), which exploits spatial coherence by grouping rays, or [Block Culling](5431-block-culling.md), which skips empty volume hierarchies, these methods focus on geometric culling, resolution scaling, and step-size adjustments to accelerate the rendering pipeline [S1, unofficial][S2, unofficial].
## Shared principles
The core principle across all optimization techniques is the reduction of redundant work. This is achieved by:
1.  **Culling:** Skipping evaluations in regions known to be empty or irrelevant using bounding volumes or hierarchical structures [S1, unofficial][S2, unofficial].
2.  **Adaptation:** Adjusting step sizes, resolution, or precision based on distance or local geometry to maintain visual fidelity with fewer samples [S1, unofficial][S2, unofficial].
3.  **Precomputation:** Baking static lighting or occlusion data into volume textures to replace runtime calculations [S3, unofficial].
## Subtopics

### [Beam Optimization](932-beam-optimization.md)
Groups adjacent rays into beams to march them together, exploiting spatial coherence to reduce SDF calls. Choose when rays exhibit high coherence, such as in orthographic views or distant scenes.

### [Performance Optimizations for SDF Ray Marching](1629-performance-optimizations-for-sdf-ray-marching.md)
Applies general techniques like over-relaxation, LOD, and resolution scaling to reduce per-pixel cost. Choose for real-time applications where frame rate is critical.

### [Block Culling](5431-block-culling.md)
Skips empty blocks in sparse volume hierarchies (e.g., octrees) based on density thresholds. Choose when rendering sparse volumes with significant empty space.

### [Bounding Shape Optimization](7750-bounding-shape-optimization.md)
Uses tight geometric bounds (spheres, polyhedra) to skip density evaluations in empty space outside the volume. Choose when the volume has a well-defined, compact boundary.

## Background
The need for acceleration in ray marching arises from the high computational cost of evaluating SDFs and performing multiple steps per pixel [S2, unofficial]. Beam optimization was introduced to leverage GPU parallelism by processing rays in groups [S1][S2, unofficial]. General performance optimizations, such as over-relaxation and resolution scaling, have become standard in real-time SDF rendering to maintain interactivity [S1, unofficial][S2, unofficial]. Block culling integrates with hardware ray tracing to handle sparse data structures efficiently [S1, unofficial]. Bounding shape optimization provides a geometric pre-filter to avoid unnecessary volumetric processing [S1, unofficial].
## Sources
