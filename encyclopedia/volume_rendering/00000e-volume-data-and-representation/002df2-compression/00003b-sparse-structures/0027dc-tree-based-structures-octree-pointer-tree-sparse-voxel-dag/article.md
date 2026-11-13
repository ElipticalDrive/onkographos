# Tree-based structures (Octree, Pointer Tree, Sparse Voxel DAG)

Also called: Sparse Voxel Octrees (SVO), Pointer Trees, DAG-based Voxel Structures

## Overview
Tree-based structures represent sparse volumetric data by recursively subdividing the 3D domain into hierarchical nodes, storing data only for occupied regions to achieve compression proportional to sparsity [S1, S2, S3]. They solve the memory bottleneck of dense voxel grids by enabling efficient culling of empty space during rendering or query operations. What sets this family apart from siblings like Hash Grids (fixed resolution, no hierarchy) and Brick Maps (fixed-size blocks) is the variable depth and branching factor, which allows adaptive resolution and precise spatial locality management [S1, S2, S3].

## Shared principles
All structures in this family share the following mechanism:

1.  **Recursive Subdivision:** The 3D domain is recursively divided into sub-volumes. In standard octrees, each node has 8 children. In pointer trees, the branching factor $N$ varies ($N^3$ children).
2.  **Sparsity Encoding:** Empty regions are represented by null pointers or absence of children, rather than storing zero-valued voxels. This provides compression proportional to the sparsity of the data.
3.  **Hierarchical Traversal:** Rendering or access requires traversing from the root to the leaf. This allows skipping large empty volumes (culling) and adapting resolution to view distance or detail requirements.
4.  **Spatial Locality:** Children of a node are spatially contiguous. This locality is often exploited via Morton ordering (Z-order curve) to map 3D coordinates to 1D keys, enabling efficient neighbor lookups and linear array representations in implicit variants [G].
5.  **Subtree Merging:** The core differentiator of the DAG family is the identification of structurally identical subtrees (isomorphic nodes) and their replacement with a single shared instance. This transforms the tree topology into a graph where multiple parents may reference the same child node, eliminating redundancy [S1, S2].

## Subtopics

### [Compact pointer tree](844-compact-pointer-tree.md)
Uses variable branching factor $N$ ($2 \le N \le 6$) and 8-bit relative pointers to compress terabyte-scale RGBA data. Choose for interactive rendering of massive volumes on GPU.

### [Sparse Voxel DAGs](2791-sparse-voxel-dags.md)
Merges identical subtrees into shared nodes to reduce memory footprint by 1-3 orders of magnitude compared to SVOs. Choose when geometry has high redundancy/isomorphism.

### [Octree-based structures](2792-octree-based-structures.md)
Standard fixed 8-way branching. Choose for general-purpose sparse volume storage where pointer overhead is acceptable and spatial locality is prioritized over extreme compression.

### [Span Space Tree](4379-span-space-tree.md)
Organizes cells by value ranges (min/max) in 2D span-space rather than spatial coordinates. Choose for isosurface extraction (Marching Cubes) where value-based culling is more efficient than spatial culling.

### [Sparse Voxel Tree (SVO Tree)](4983-sparse-voxel-tree-svo-tree.md)
Fixed branching factor of 8 with block-based memory pooling. Choose for standard sparse voxel applications requiring fast allocation/deallocation and simple traversal.

### [Tiny LOD](6968-tiny-lod.md)
Constructs LOD trees from splat data with explicit merge filtering. Choose for splat-based rendering requiring hierarchical detail levels.

### [NerfTree](7559-nerftree.md)
Two-level sparse voxel structure for NeRF, decoupling coarse geometry caching from fine radiance refinement. Choose for accelerating Neural Radiance Fields training/inference.

### [Adaptively Sampled Distance Fields](7773-adaptively-sampled-distance-fields.md)
Compresses SDFs using sign-preserving simplification. Choose for resolution-independent antialiasing of complex vector outlines or shapes.

## Background
Tree-based sparse structures originated in the early 1990s for efficient storage and rendering of volumetric medical and scientific data [G]. The Sparse Voxel Octree (SVO) became a standard representation for real-time volume rendering in the 2000s, enabling interactive frame rates for large datasets [S3]. Sparse Voxel DAGs were introduced later to address the redundancy in SVOs where identical geometric structures are stored multiple times, achieving higher compression ratios [S1, S2]. Compact Pointer Trees were developed to handle terabyte-scale datasets on GPUs by optimizing pointer storage and color compression [S1]. These structures are widely used in medical imaging, scientific visualization, and modern neural rendering pipelines [S1, S2, S3].

## Sources
[S1] Compact Pointer Tree documentation, https://github.com/... (unofficial), 2023
[S2] Sparse Voxel DAGs research paper, https://doi.org/... (unofficial), 2022
[S3] Sparse Voxel Tree implementation notes, https://github.com/... (unofficial), 2021
