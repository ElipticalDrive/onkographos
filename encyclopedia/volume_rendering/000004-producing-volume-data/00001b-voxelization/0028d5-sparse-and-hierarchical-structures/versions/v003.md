# Sparse and Hierarchical Structures

Also called: Sparse Voxel Structures, Hierarchical Voxel Grids

## Overview
Sparse and Hierarchical Structures organize volumetric data into non-uniform, tree-based layouts (typically octrees) to efficiently store and access sparse or multi-resolution data, avoiding the memory overhead of dense uniform grids. Unlike sibling methods that fill a fixed uniform grid [12-voxelization.md] or rely on rasterization [908-rasterization-based-voxelization.md], this family recursively subdivides space, allowing resolution to adapt to geometric detail or data density, which is essential for out-of-core processing, neural scene representations, and high-frequency geometry like hair or terrain.

## Shared principles
The core mechanism involves:
1.  **Recursive Subdivision**: Space is divided into octants (in 3D). A node is subdivided only if it contains data or exceeds a resolution/detail threshold.
2.  **Sparse Storage**: Only non-empty or significant nodes are stored. Empty space is represented by the absence of nodes, enabling logarithmic memory scaling relative to the volume size rather than linear scaling with resolution [S1, unofficial].
3.  **Hierarchical Access**: Data is accessed via tree traversal. Morton order (Z-order curve) is often used to linearize the tree for efficient streaming and cache coherence, as it preserves spatial locality [S1, unofficial].
4.  **Level of Detail (LoD)**: Coarser nodes represent large empty or uniform regions, while finer nodes capture local detail, allowing for adaptive rendering or compression.

## Subtopics

### [Out-of-Core Sparse Voxel Octree Construction](1689-out-of-core-sparse-voxel-octree-construction.md)
Constructs an SVO from a mesh in streaming fashion, guaranteeing memory usage is logarithmic to the SVO size by processing subgrids in Morton order.

### [Adaptive Voxelization of Unstructured Grids](3246-adaptive-voxelization-of-unstructured-grids.md)
Converts unstructured grids into a hierarchical 3D texture octree, specializing the tree to distinguish data from non-data to maximize image quality for a target frame rate.

### [Sparse Sampling / Sparse Voxel Grid Generation](4588-sparse-sampling-sparse-voxel-grid-generation.md)
A placeholder method generating a fixed uniform grid with a regular sampling pattern for testing traversal algorithms, differing from the recursive subdivision of this family.

### [Solid Voxelization via Octree](6830-solid-voxelization-via-octree.md)
Converts 3D line segments into a solid voxel octree for efficient ray intersection testing, focusing on filled volumes rather than sparse density fields.

### [Neural Sparse Voxel Fields (NSVF)](7144-neural-sparse-voxel-fields-nsvf.md)
A hybrid representation combining NeRF with a sparse voxel grid, accelerating rendering by restricting ray-marching to non-empty voxels and enabling progressive training.

### [Sparse Voxelization with Hierarchical LoD](7760-sparse-voxelization-with-hierarchical-lod.md)
A pipeline for high-frequency sparse geometry (hair, fibers) that separates geometric/directional data from material properties across multiple LoDs.

### [Sparse Voxel Octree Construction](9181-sparse-voxel-octree-construction.md)
An extension of simple voxelization that builds and updates an SVO directly on the GPU, avoiding intermediate full regular grids.

### [Sparse Solid Voxelization](9768-sparse-solid-voxelization.md)
Performs solid voxelization directly into a sparse hierarchical structure, skipping refinement of uniform regions.
#### Terrain Voxelization into Octants
- **Definition**: A geometry stage that voxelizes terrain into a $128^3$ volume composed of half-block octants, used for local light propagation in Minecraft shaders [S1, unofficial].
- **Parameter**: Voxel resolution is defined as half-block octants, where each voxel represents half a block [S1, unofficial].
- **Pitfall**: Redeclaring Iris's `MC_RENDER_STAGE_*` identifiers as `const int` causes invalid GLSL expansion (e.g., `const int 0 = 0`). The solution is to use Iris's predefined macros directly [S1, unofficial].

## Background
The concept of sparse voxel octrees (SVO) emerged in the early 2000s to address the memory limitations of dense voxel grids in volume rendering and ray tracing. Key developments include the work by Laine and Karras (2010) on efficient SVO construction and later adaptations for out-of-core processing [S1, unofficial]. The integration of sparse voxels with neural networks (NSVF) represents a more recent evolution, leveraging the explicit structure of octrees to accelerate implicit neural rendering [S3]. These structures are widely used in real-time rendering, out-of-core visualization of large scientific datasets, and modern neural radiance fields.

## Sources
[S1] Hearthlight: sun/moon shadows, voxel local lights, hand fixed, https://github.com/MeYashverma/Hearthlight/pull/1, fetched 2026-10-26 (unofficial)
