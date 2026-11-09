# Tree-based structures (Octree, Pointer Tree, Sparse Voxel DAG)

Also called: Sparse Voxel Trees, Hierarchical Voxel Grids, Subdivision Trees

## Overview
Tree-based structures represent volumetric data using a recursive spatial subdivision where empty or uniform regions are collapsed into single nodes, providing compression proportional to data sparsity [G]. This family solves the memory and bandwidth bottlenecks of dense voxel grids by enabling efficient culling of empty space and adaptive resolution [G]. Unlike sibling structures like Hash Grids or Brick Maps which rely on flat hashing or fixed-size blocks, tree-based methods prioritize hierarchical traversal and spatial locality, allowing for variable depth and explicit parent-child relationships that facilitate complex operations like neighbor lookups and level-of-detail (LOD) generation [G].

## Shared principles
All structures in this family share the following mechanism:

1.  **Recursive Subdivision:** The 3D domain is recursively divided into sub-volumes. In standard octrees, each node has 8 children. In pointer trees, the branching factor $N$ varies ($N^3$ children).
2.  **Sparsity Encoding:** Empty regions are represented by null pointers or absence of children, rather than storing zero-valued voxels. This provides compression proportional to the sparsity of the data.
3.  **Hierarchical Traversal:** Rendering or access requires traversing from the root to the leaf. This allows skipping large empty volumes (culling) and adapting resolution to view distance or detail requirements.
4.  **Spatial Locality:** Children of a node are spatially contiguous. This locality is often exploited via Morton ordering (Z-order curve) to map 3D coordinates to 1D keys, enabling efficient neighbor lookups and linear array representations in implicit variants [G].
5.  **Subtree Merging:** The core differentiator of the DAG family is the identification of structurally identical subtrees (isomorphic nodes) and their replacement with a single shared instance. This transforms the tree topology into a graph where multiple parents may reference the same child node, eliminating redundancy [S1][S2].

## Subtopics

### [Compact pointer tree](844-compact-pointer-tree.md)
Uses variable branching factor $N$ ($2 \le N \le 6$) and compact 8-bit relative pointers to optimize memory for terabyte-scale RGBA data on GPUs. It combines wavelet-based color compression with hierarchical DDA traversal.

### [Sparse Voxel DAGs](2791-sparse-voxel-dags.md)
Merges identical subtrees into shared nodes to reduce memory footprint by 1-3 orders of magnitude compared to standard trees. It decouples geometric hierarchy from material attributes and employs bottom-up reduction or hash-based recycling.

### [Octree-based structures](2792-octree-based-structures.md)
Relies on a fixed 8-way branching factor and explicit or implicit parent-child pointers. It prioritizes hierarchical traversal and spatial locality over the deduplication or pointer-free compactness of DAGs or variable-branching trees.

### [Span Space Tree](4379-span-space-tree.md)
Organizes data along axis-aligned spans (intervals) rather than just voxel occupancy. It refines the interval tree concept to manage scene complexity, allowing for more efficient culling of empty or uniform regions during ray casting.

### [Sparse Voxel Tree (SVO Tree)](4983-sparse-voxel-tree-svo-tree.md)
A standard hierarchical structure with a fixed branching factor of 8. It allocates memory only for non-empty regions, providing compression proportional to sparsity, but allows multiple paths to the same voxel (no sharing).

### [Tiny LOD](6968-tiny-lod.md)
Constructs a hierarchical LOD tree from splat data, parameterized by a base LOD value and a merge filter flag. It outputs encoded opacity values and a chunk tree, distinguished by its specific procedural pipeline for splat-based rendering.

### [NerfTree](7559-nerftree.md)
A two-level octree structure for Neural Radiance Fields (NeRF). It uses a coarse density volume to cache approximate geometry and fine sparse voxel grids for refinement, reducing MLP evaluations by skipping empty space.

### [Adaptively Sampled Distance Fields](7773-adaptively-sampled-distance-fields.md)
Compresses Signed Distance Fields (SDFs) by exploiting smoothness. It uses adaptive simplification with a strict sign-preservation constraint to maintain topological accuracy of the zero isosurface, enabling resolution-independent antialiasing.

## Background
Tree-based structures originated in the 1980s as a solution to the "curse of dimensionality" in spatial data indexing, with octrees being one of the earliest forms [G]. In computer graphics, they were adopted in the 1990s for ray tracing and volume rendering to handle sparse scenes efficiently [G]. The Sparse Voxel Octree (SVO) became a standard for high-quality volume rendering in the 2000s, enabling interactive frame rates for large datasets [G]. Recent developments include DAGs for deduplication [S1][S2], compact pointer trees for GPU efficiency [S1][S2], and adaptations for NeRF [S1] and SDFs [S1].

## Sources
[S1] Compact Pointer Tree, https://github.com/... (unofficial notes)
[S2] Sparse Voxel DAGs, https://github.com/... (unofficial notes)
