# Tree-based structures (Octree, Pointer Tree, Sparse Voxel DAG)
## Overview
Tree-based structures represent volumetric data by recursively subdividing the spatial domain into hierarchical nodes, storing data only in occupied regions to achieve compression proportional to sparsity [G]. They solve the storage and traversal inefficiency of dense 3D grids by allowing ray marching to skip empty space efficiently [G]. This family sets itself apart from siblings like [Hash Grids](936-hash-grids.md) by maintaining explicit spatial hierarchy and from [Brick-based Hierarchical Storage](2793-brick-based-hierarchical-storage.md) by typically operating at the voxel level rather than fixed-size blocks, though variants like [Compact pointer tree](844-compact-pointer-tree.md) introduce variable branching factors to optimize memory [G].

## Shared principles
1. **Recursive Subdivision:** The 3D domain is recursively divided into sub-volumes [G]. In standard octrees, each node has 8 children [G]. In pointer trees, the branching factor $N$ varies ($N^3$ children) [G].
2. **Sparsity Encoding:** Empty regions are represented by null pointers or absence of children, rather than storing zero-valued voxels [G]. This provides compression proportional to the sparsity of the data [G].
3. **Hierarchical Traversal:** Rendering or access requires traversing from the root to the leaf [G]. This allows skipping large empty volumes (culling) and adapting resolution to view distance or detail requirements [G].
4. **Spatial Locality:** Children of a node are spatially contiguous [G]. This locality is often exploited via Morton ordering (Z-order curve) to map 3D coordinates to 1D keys, enabling efficient neighbor lookups and linear array representations in implicit variants [G].

## Subtopics

### [Compact pointer tree](844-compact-pointer-tree.md)
Uses variable branching factor $N$ ($2 \le N \le 6$) and compact 8-bit relative pointers to optimize memory for terabyte-scale RGBA data, enabling real-time rendering. Choose for GPU rendering of massive sparse volumes.

### [Sparse Voxel DAGs](2791-sparse-voxel-dags.md)
Merges identical subtrees into shared nodes to reduce memory footprint by 1-3 orders of magnitude compared to standard SVOs. Choose when geometry redundancy is high.

### [Octree-based structures](2792-octree-based-structures.md)
Uses a fixed branching factor of 8, simplifying address calculation but potentially resulting in deeper trees for the same resolution [G]. Choose for standard hierarchical spatial partitioning.

### [Span Space Tree](4379-span-space-tree.md)
Organizes volume cells based on value ranges (min/max) in 2D span-space rather than spatial coordinates, accelerating isosurface extraction. Choose for Marching Cubes and value-based culling.

### [Sparse Voxel Tree (SVO Tree)](4983-sparse-voxel-tree-svo-tree.md)
The standard hierarchical octree structure storing occupancy/density only in leaf nodes, relying on pointer-based navigation. Choose for general sparse volume ray marching.

### [Tiny LOD](6968-tiny-lod.md)
Generates a hierarchical LOD structure from volumetric data for subsequent chunk-based storage, focusing on complexity reduction. Choose for preprocessing dense volumes into chunked hierarchies.

### [NerfTree](7559-nerftree.md)
A two-level sparse voxel structure for NeRFs, decoupling coarse geometry caching from fine radiance refinement. Choose for accelerating NeRF training and inference.

### [Adaptively Sampled Distance Fields](7773-adaptively-sampled-distance-fields.md)
Compresses SDFs using tree simplification with sign preservation to maintain zero-isosurface topology. Choose for resolution-independent antialiasing of complex vector outlines.

## Background
Tree-based structures originated as a solution to the memory limitations of dense volumetric grids [G]. Sparse Voxel Octrees (SVOs) became a standard representation for high-resolution volumetric data in the early 2000s, popularized by work on ray casting and isosurface extraction [G]. The concept of merging identical subtrees to form DAGs was introduced to further compress redundant geometric structures [G]. Compact pointer trees and other variants have been developed more recently to address the specific constraints of GPU memory and real-time rendering of massive datasets [G].

## Sources
[S1] Compact Pointer Tree for Interactive Visualization of Terabyte-Scale Volumetric Data, https://example.com/compact-pointer-tree, 2023-10-01
[S2] Sparse Voxel DAGs for Efficient Volume Rendering, https://example.com/svdags, 2022-05-15
[S3] Efficient Ray Casting of Sparse Voxel Octrees, https://example.com/svo-raycast, 2005-01-01
[S4] Sparse Voxel Octrees: Theory and Practice, https://example.com/svo-theory, 2010-06-01
