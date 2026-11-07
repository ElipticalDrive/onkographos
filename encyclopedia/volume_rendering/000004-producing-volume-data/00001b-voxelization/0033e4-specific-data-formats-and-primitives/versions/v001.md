# Specific Data Formats and Primitives

Also called: Volume Graphics, Hypertexture Mesh Amplification

## Overview
This family encompasses specific data structures, storage formats, and primitive representations used to store and process voxelized volume data. It addresses the problem of efficiently storing, retrieving, and rendering volumetric information by defining how continuous geometry or scalar fields are mapped into discrete voxel grids and how those grids are organized in memory or on disk. It is distinguished from general voxelization algorithms by its focus on the resulting data representation (e.g., out-of-core files, CSG worksheets, or attribute-rich buffers) rather than the geometric conversion process itself.

## Shared principles
The methods in this family share a reliance on discrete spatial indexing and attribute storage. While the specific encoding varies, the core mechanism involves:
1.  **Discretization**: Mapping continuous world-space coordinates or geometric primitives into a finite set of integer grid indices $(i, j, k)$ [G].
2.  **Attribute Association**: Linking geometric or scalar data (such as density, color, or normal vectors) to these indices. This may involve pre-computed view-independent attributes [S2] or binary occupancy flags.
3.  **Storage Organization**: Organizing these voxel attributes into data structures suitable for the intended access pattern, such as linear files with look-up tables for out-of-core access [S1, unofficial] or GPU texture arrays for real-time blending [S1, unofficial].

## Subtopics

### [CSG Model Voxelization](2497-csg-model-voxelization.md)
CSG Model Voxelization converts Constructive Solid Geometry (CSG) trees into voxel grids by voxelizing individual leaf nodes into intermediate "worksheets" and performing boolean operations in image space using GPU blending. It differs from its siblings by leveraging density computation inherent in solid voxelization to recover intersection volumes rapidly, making it suitable for collision detection and haptic feedback. Unlike SDF-based methods, it operates directly on discrete voxel grids for real-time performance. See [CSG Model Voxelization](2497-csg-model-voxelization.md).

### [Out-of-Core Point-Based Rendering](6825-out-of-core-point-based-rendering.md)
Out-of-core point-based rendering visualizes very large volumetric datasets by storing voxel data on disk and bypassing memory limitations. It solves the problem of real-time interaction with high-resolution data by assuming voxel density is sufficient to represent isovalue structures as discrete points, eliminating the need for connectivity. Unlike mesh-based methods, it renders contours directly from sorted point clouds. See [Out-of-Core Point-Based Rendering](6825-out-of-core-point-based-rendering.md).

### [Point Feature Generation](6972-point-feature-generation.md)
Point Feature Generation creates features for ray points by indexing volume features and building color volumes from images. See [Point Feature Generation](6972-point-feature-generation.md).

### Hypertexture Mesh Amplification
Hypertexture Mesh Amplification is used to define a volume by the interior of a mesh [S1, unofficial]. Currently, animation is a significant problem for this volume data type [S1, unofficial].

### Volume Graphics
Volume Graphics is an approach concerned with the synthesis, manipulation, and rendering of volumetric geometric objects, where geometric models are voxelized (3D scan-converted) into a set of voxels stored in a volume buffer with pre-computed view-independent attributes [S2].
*   **Step**: Voxelize the geometric model into a set of voxels that best approximate the model [S2]. Store each voxel in the volume buffer along with pre-computed view-independent attributes such as texture, anti-aliasing, and normal vector [S2].
*   **Comparison**: Volume graphics focuses on modeled geometric scenes represented in a volume buffer, whereas volume visualization focuses on sampled and computed datasets [S2].

## Background
The concept of "Volume Graphics" was introduced by Kaufman et al. [S2] as an emerging approach to advance 3D graphics by offering a comprehensive alternative to traditional surface graphics. It leverages the viewpoint-independence and insensitivity to scene complexity of voxel-based representations. The specific techniques for out-of-core rendering and CSG voxelization were further developed in subsequent research, such as the work by Coeurjolly et al. [S1, unofficial], which explored ray-guided streaming and efficient voxel rendering for large datasets.

## Sources
[S1] [PDF] Ray-Guided Streaming for Efficient and Detailed Voxel Rendering, https://www-sop.inria.fr/reves/Basilic/2009/CNLE09/CNLE09.pdf, fetched 2026-10-26 (unofficial)
[S2] Volume visualization, https://dl.acm.org/doi/pdf/10.1145/234313.234383, fetched 2026-11-01
