# Tree-based structures (Octree, Pointer Tree, Sparse Voxel DAG)

Also called: Sparse Voxel Trees, Hierarchical Sparse Volumes

## Overview
Tree-based structures organize sparse volumetric data into a hierarchical graph where internal nodes define spatial subdivisions and leaf nodes store voxel values. This family solves the memory inefficiency of dense grids by omitting empty space and the traversal overhead of flat sparse lists by leveraging spatial locality. They differ from sibling structures like [Hash Grids](936-hash-grids.md) by providing explicit spatial hierarchy for culling, and from [Brick-based Hierarchical Storage](2793-brick-based-hierarchical-storage.md) by operating on individual voxel granularity rather than fixed-size blocks.

## Shared principles
All structures in this family share the following mechanism:

1.  **Recursive Subdivision:** The 3D domain is recursively divided into sub-volumes. In standard octrees, each node has 8 children. In pointer trees, the branching factor $N$ varies ($N^3$ children).
2.  **Sparsity Encoding:** Empty regions are represented by null pointers or absence of children, rather than storing zero-valued voxels. This provides compression proportional to the sparsity of the data.
3.  **Hierarchical Traversal:** Rendering or access requires traversing from the root to the leaf. This allows skipping large empty volumes (culling) and adapting resolution to view distance or detail requirements.
4.  **Spatial Locality:** Children of a node are spatially contiguous. This locality is often exploited via Morton ordering (Z-order curve) to map 3D coordinates to 1D keys, enabling efficient neighbor lookups and linear array representations in implicit variants [G].

## Subtopics

### [Compact pointer tree](844-compact-pointer-tree.md)
Uses variable branching factors ($N^3$) and 8-bit relative offsets for compact storage. Choose when memory footprint is critical and data has high redundancy.

### [Sparse Voxel DAGs](2791-sparse-voxel-dags.md)
Merges identical subtrees into a Directed Acyclic Graph to eliminate topological redundancy. Choose when the volume contains repeated geometric patterns.

### [Octree-based structures](2792-octree-based-structures.md)
Fixed 8-way branching with implicit or explicit storage. Choose for standard spatial indexing and neighbor queries.

### [Span Space Tree](4379-span-space-tree.md)
Refines interval trees to manage scene complexity for large volumes. Choose for complex scene culling.

### [Sparse Voxel Tree (SVO Tree)](4983-sparse-voxel-tree-svo-tree.md)
Standard sparse voxel octree implementation. Choose for general-purpose ray marching.

## Background
Tree-based sparse structures originated in the early 1990s for efficient storage of 3D medical and scientific data. The Sparse Voxel Octree (SVO) was popularized for volume rendering by Levoy et al. [S1]. The concept of merging identical subtrees into DAGs was introduced to reduce memory usage for complex geometries [S2]. Modern variants like the Compact Pointer Tree address the pointer overhead of traditional trees by compressing indices and integrating data compression into the hierarchy [S1].

## Sources
[S1] Compact Pointer Tree for Terabyte-Scale Volumetric Data, URL, 2023
[S2] Sparse Voxel DAGs for Memory-Efficient Volume Rendering, URL, 2022
