# Specific Data Formats and Primitives
## Overview
This family encompasses methods that define the specific volumetric data structures (formats) and the geometric primitives used to represent or generate volume data for rendering [G]. It addresses the problem of how continuous geometric models, scalar fields, or complex CSG trees are discretized into a form suitable for efficient storage and rendering [G]. It sets itself apart from sibling methods by focusing on the *representation* (e.g., voxel buffers, sorted point lists, binary classification maps) and the *primitive types* (e.g., CSG nodes, hypertextures) rather than the specific rasterization or ray-casting algorithms used to visualize them [G].
## Shared principles
Builds on [Voxelization](12-voxelization.md) shared principles.
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

### Voxel Volume Data Format
A voxel volume is defined by a JSON object containing an `attributes` array, a `position` array, a `size` array, and a `primitives` array [S2, unofficial]. Each attribute object contains a `semantic` string (e.g., "MATERIAL_INDEX", "DENSITY"), a `componentType` integer (e.g., 5123 for UNSIGNED_BYTE), and an optional `normalized` boolean [S2, unofficial]. The `position` and `size` arrays each contain three numeric values representing the origin and dimensions of the volume [S2, unofficial].

## Background
Hypertextures and mesh amplification were explored as methods to define volumes via mesh interiors, with early implementations noting limitations in animation [S1, unofficial]. Voxel volume data formats have been standardized using JSON structures to define attributes, positions, sizes, and primitives [S2, unofficial].
## Sources
[S1] Ray-Guided Streaming for Efficient and Detailed Voxel Rendering, https://www-sop.inria.fr/reves/Basilic/2009/CNLE09/CNLE09.pdf, fetched 2026-10-26 (unofficial)
[S2] Voxel data format, https://github.com/enkisoftware/OpenVoxelFileFormat/issues/4, fetched 2026-10-27 (unofficial)
