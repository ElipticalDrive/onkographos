# Specific Data Formats and Primitives
## Overview
This family encompasses methods that define the specific volumetric data structures (formats) and the geometric primitives used to represent or generate volume data for rendering. It addresses the problem of how continuous geometric models, scalar fields, or complex CSG trees are discretized into a form suitable for efficient storage and rendering. It sets itself apart from sibling methods by focusing on the *representation* (e.g., voxel buffers, sorted point lists, binary classification maps) and the *primitive types* (e.g., CSG nodes, hypertextures) rather than the specific rasterization or ray-casting algorithms used to visualize them.
## Shared principles
The parent's Shared principles apply:
1. **Domain Definition**: Establishing a spatial domain, typically a 3D grid with resolution $(N_x, N_y, N_z)$ and bounding box, or an infinite analytic space [G].
2. **Value Assignment**: Determining the value (density, distance, color) for each point in the domain based on the source method (geometric intersection, function evaluation, or physical state) [G].
3. **Storage/Access**: Either storing the result in a discrete array (voxelization, simulation) or providing an interface to compute it on-the-fly (procedural) [G].
## Subtopics

### [CSG Model Voxelization](2497-csg-model-voxelization.md)
Hardware-accelerated binary voxelization of Constructive Solid Geometry models using frame buffer bit-masks and Point-Classification Maps. Choose when rendering complex boolean combinations of primitives efficiently on GPU.

### [Volume Graphics](6282-volume-graphics.md)
Decoupled representation where geometric models are voxelized into a buffer with pre-computed view-independent attributes (normals, texture). Choose when viewpoint-independent rendering and anti-aliasing are prioritized over scene complexity.

### [Out-of-Core Point-Based Rendering](6825-out-of-core-point-based-rendering.md)
Interactive visualization of massive datasets by sorting voxels by function value into a single file with a look-up table. Choose when datasets exceed main memory and interactive isovalue selection is required.

### [Point Feature Generation](6972-point-feature-generation.md)
Aggregation of multi-view image features into ray-aligned tensors for neural network processing. Choose when preparing data for ray-based neural volume rendering rather than traditional rasterization.

### Hypertexture Mesh Amplification
Used to define a volume by the interior of a mesh [S1, unofficial]. Currently, animation is a significant problem for this volume data type [S1, unofficial].

## Background
The concept of "Volume Graphics" was introduced to distinguish modeled geometric objects voxelized into a regular grid from volume visualization of sampled datasets [S1, unofficial][S2]. Hypertextures and mesh amplification were explored as methods to define volumes via mesh interiors, with early implementations noting limitations in animation [S1, unofficial].
## Sources
[S1] Ray-Guided Streaming for Efficient and Detailed Voxel Rendering, https://www-sop.inria.fr/reves/Basilic/2009/CNLE09/CNLE09.pdf, fetched 2026-10-26 (unofficial)
