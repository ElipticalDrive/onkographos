# Tree-based structures (Octree, Pointer Tree, Sparse Voxel DAG)

Also called: Sparse Voxel Trees, Hierarchical Voxel Structures, Tree-based Sparse Volumes

## Overview
Tree-based structures represent sparse volumetric data by recursively subdividing the 3D domain into hierarchical nodes, storing data only for occupied regions to achieve compression proportional to sparsity [S1, S2, S3]. They solve the memory bottleneck of dense voxel grids by enabling efficient culling of empty space during rendering or query operations. What sets this family apart from sibling methods like Hash Grids (flat, no hierarchy) and Brick Maps (fixed-size blocks) is the variable depth and branching factor, which allows adaptive resolution and precise spatial locality management [S1, S2, S3].

## Shared principles
All structures in this family share the following mechanism:

1.  **Recursive Subdivision:** The 3D domain is recursively divided into sub-volumes. In standard octrees, each node has 8 children. In pointer trees, the branching factor $N$ varies ($N^3$ children).
2.  **Sparsity Encoding:** Empty regions are represented by null pointers or absence of children, rather than storing zero-valued voxels. This provides compression proportional to the sparsity of the data.
3.  **Hierarchical Traversal:** Rendering or access requires traversing from the root to the leaf. This allows skipping large empty volumes (culling) and adapting resolution to view distance or detail requirements.
4.  **Spatial Locality:** Children of a node are spatially contiguous. This locality is often exploited via Morton ordering (Z-order curve) to map 3D coordinates to 1D keys, enabling efficient neighbor lookups and linear array representations in implicit variants [G].
5.  **Subtree Merging:** The core differentiator of the DAG family is the identification of structurally identical subtrees (isomorphic nodes) and their replacement with a single shared instance. This transforms the tree topology into a graph where multiple parents may reference the same child node, eliminating redundancy [S1, S2].

## Subtopics

### [Compact pointer tree](844-compact-pointer-tree.md)
Uses variable branching factor $N$ ($2 \le N \le 6$) and 8-bit relative pointers to optimize memory for terabyte-scale RGBA data, enabling real-time rendering via hierarchical DDA.

### [Sparse Voxel DAGs](2791-sparse-voxel-dags.md)
Merges identical subtrees into shared nodes to reduce memory footprint by 1–3 orders of magnitude compared to standard SVOs, explicitly decoupling geometry from material attributes.

### [Octree-based structures](2792-octree-based-structures.md)
Uses a fixed 8-way branching factor, offering a trade-off between pointer overhead, traversal depth, and spatial locality management.

### [Span Space Tree](4379-span-space-tree.md)
Embeds axis-aligned span intervals within the hierarchy to reduce visited cells during traversal, refining interval trees for large-scale volumes.

### [Sparse Voxel Tree (SVO Tree)](4983-sparse-voxel-tree-svo-tree.md)
A standard hierarchical structure with fixed 8 children per node, allocating memory only for non-empty regions, providing compression proportional to sparsity.

### [Tiny LOD](6968-tiny-lod.md)
Constructs a hierarchical LOD tree from splat data with explicit merge filtering, outputting encoded opacity and chunk trees.

### [NerfTree](7559-nerftree.md)
A two-level sparse voxel structure for NeRFs, decoupling coarse geometry caching from fine radiance refinement to accelerate training and inference.

### [Adaptively Sampled Distance Fields](7773-adaptively-sampled-distance-fields.md)
Compresses SDFs using tree simplification with sign preservation to maintain zero-isosurface topology, suitable for resolution-independent antialiasing.

## Background
Tree-based sparse structures originated in the early 1990s as a solution to the memory inefficiency of dense voxel grids in volume rendering and geometric modeling [G]. The Sparse Voxel Octree (SVO) was popularized in the mid-2000s for real-time ray tracing and volume visualization, enabling the rendering of large datasets by skipping empty space [S2, S3]. Subsequent developments introduced DAGs to handle geometric redundancy [S1, S2], compact pointer trees for GPU-friendly terabyte-scale data [S1, S2], and adaptations for neural rendering (NeRF) and distance fields [S1, S2, S3]. These structures are now standard in high-performance volume rendering, medical imaging, and geometric compression.

## Sources
[S1] Compact Pointer Tree, Sparse Voxel DAGs, NerfTree, Tiny LOD, Adaptively Sampled Distance Fields notes
[S2] Sparse Voxel DAGs, Octree-based structures, SVO Tree notes
[S3] SVO Tree notes
