# Voxel Representation

Also called: Voxel Approach, Voxel Grid Representation

## Overview
Voxel representation discretizes 3D space into a grid of unit cubes to store scene parameters such as occupancy, density, or color [S2, unofficial]. It solves the problem of representing continuous 3D geometry and appearance by assigning a constant value to the entire volume surrounding each grid point, avoiding assumptions about data behavior between grid points [S1, unofficial]. Unlike the cell approach, which assumes values vary between grid points via interpolation, the voxel approach uses only known data values, resulting in images that may appear less smooth but are strictly verified by the stored data [S1, unofficial].

## Shared principles
Builds on [Explicit Voxel/Grid Representations](12527-explicit-voxel-grid-representations.md) [G]. Each voxel is modeled as a rectangular cuboid with six faces, twelve edges, and eight corners [S1, unofficial]. The core definition dictates that the area around a grid point has the same value as the grid point itself [S1, unofficial]. Consequently, data values do not vary within a voxel; the voxel is an area of non-varying value surrounding a central grid point [S1, unofficial]. This approach explicitly does not make assumptions about data behavior between grid points [S1, unofficial]. Only known data values are used for image generation [S1, unofficial]. This stands in contrast to the cell approach, which views a volume as a collection of hexahedra whose corners are grid points and whose value varies between them, typically using trilinear or tricubic interpolation [S1, unofficial]. Because the voxel approach relies solely on known data values, the validity of the resulting images can be verified, whereas cell-based images cannot be strictly verified due to interpolation artifacts [S1, unofficial]. A key advantage of this representation is that the position of a voxel is fixed in 3D space [S2, unofficial]. This fixed positioning prevents the gradient issues caused by the displacement of shape primitives (discontinuity) that occur in other primitive-based representations [S2, unofficial].

## Subtopics

### Explicit Radiance Fields
See [Explicit Radiance Fields](806-explicit-radiance-fields.md).

### Hierarchical Grid Optimization
See [Hierarchical Grid Optimization](1414-hierarchical-grid-optimization.md).

### Direct Voxel Grid Optimization
See [Direct Voxel Grid Optimization](4694-direct-voxel-grid-optimization.md).

### Canonical Radiance Field with Voxel Grid
See [Canonical Radiance Field with Voxel Grid](8920-canonical-radiance-field-with-voxel-grid.md).

### Forward Flow for Novel View Synthesis of Dynamic Scenes
See [Forward Flow for Novel View Synthesis of Dynamic Scenes](13903-forward-flow-for-novel-view-synthesis-of-dynamic-scenes.md).

### DeepVoxels
See [DeepVoxels](16577-deepvoxels.md).

### Voxel Occupancy Model
A specific definition model where each voxel is defined by a material (enum) and an occupancy value (floating-point in the 0..1 range) [S3, unofficial]. Occupancy 0 means the voxel is completely empty, and occupancy 1 means the voxel has as much matter as it can possibly contain, though not necessarily in a cube shape [S3, unofficial]. The material storage is an index into a material table, which contains 23 entries [S3, unofficial]. To conserve space, occupancy is stored as a quantized 8-bit value, resulting in two bytes per voxel [S3, unofficial]. This model was chosen over alternatives for its balance of flexibility and simplicity. The pure Material model was rejected because it lacked sub-voxel flexibility for tools like erosion [S3, unofficial]. Hermite data was rejected because it lacked a sufficient API for tools (e.g., smoothing) and exposing raw data was too complex [S3, unofficial]. Signed Distance Fields were rejected because they had a more complicated mental model and provided similar content results [S3, unofficial]. In this model, an explicit Air material is used to define empty voxels, and occupancy is decoded specifically for non-air materials as `(value + 1) / 256.0` to ensure precise floating-point representation [S3, unofficial].

## Background
The voxel approach is a foundational technique in volume graphics and visualization [S1, unofficial]. It is characterized by its discrete nature, where the volume is partitioned into a grid [S1, unofficial]. The distinction between the voxel approach (constant value per voxel) and the cell approach (interpolated values) is a classic classification in volume rendering algorithms [S1, unofficial]. In the context of differentiable rendering, voxels are used to represent data as unit cubes parametrized as N-dimensional vectors containing occupancy, transparency, or distance information [S2, unofficial]. Occupancy information is commonly encoded using binary values or non-binary transparency [S2, unofficial]. For applications predicting occupancy, non-binary occupancy probability $P_o \in [p_{min}, p_{max}]$ is usually stored [S2, unofficial]. Although occupancy probabilities are distinct from transparency values, they can be interpreted similarly to maintain differentiability during the ray marching operation, where $P_o$ denotes a ray's absorption (transparency) at a certain point [S2, unofficial]. Rendering involves collecting all voxels located along a ray projecting to a pixel [S2, unofficial]. Two primary strategies exist for this collection: performing the operation in world space [S2, unofficial] or projecting voxels from world space to screen space using camera parameters and performing bilinear sampling, which is more computationally efficient [S2, unofficial].

## Sources
[S1] Volume Rendering, https://www.cs.unh.edu/~cs880/visualization/Celebi-Volume%20Rendering.pdf, fetched 2026-10-26 (unofficial)
[S2] Differentiable Rendering: A Survey, https://doi.org/10.48550/arxiv.2006.12057, fetched 2026-10-26 (unofficial)
[S3] Voxel terrain: storage, https://zeux.io/2017/03/27/voxel-terrain-storage/, fetched 2026-10-29 (unofficial)
