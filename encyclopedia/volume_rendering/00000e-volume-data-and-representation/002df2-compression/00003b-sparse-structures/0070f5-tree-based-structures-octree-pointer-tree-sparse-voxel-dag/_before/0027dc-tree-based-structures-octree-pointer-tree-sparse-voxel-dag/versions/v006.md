# Tree-based structures (Octree, Pointer Tree, Sparse Voxel DAG)

Also called: Hierarchical Sparse Voxel Structures, Tree-based Sparse Volumes

## Overview
Tree-based structures represent volumetric data using a recursive spatial subdivision where empty or uniform regions are collapsed into single nodes, providing compression proportional to data sparsity [2792-octree-based-structures.md]. This family solves the memory and bandwidth bottlenecks of dense voxel grids by enabling efficient culling of empty space and adaptive resolution. Unlike sibling structures like [Hash Grids](936-hash-grids.md) or [SparseGrid](904-sparsegrid.md), which rely on flat hashing or fixed brick maps, tree-based structures prioritize hierarchical traversal and spatial locality, allowing for variable branching factors and explicit parent-child relationships [2792-octree-based-structures.md].

## Shared principles
All structures in this family share the following mechanism:

1.  **Recursive Subdivision:** The 3D domain is recursively divided into sub-volumes. In standard octrees, each node has 8 children. In pointer trees, the branching factor $N$ varies ($N^3$ children).
2.  **Sparsity Encoding:** Empty regions are represented by null pointers or absence of children, rather than storing zero-valued voxels. This provides compression proportional to the sparsity of the data.
3.  **Hierarchical Traversal:** Rendering or access requires traversing from the root to the leaf. This allows skipping large empty volumes (culling) and adapting resolution to view distance or detail requirements.
4.  **Spatial Locality:** Children of a node are spatially contiguous. This locality is often exploited via Morton ordering (Z-order curve) to map 3D coordinates to 1D keys, enabling efficient neighbor lookups and linear array representations in implicit variants [G].

## Subtopics

### [Compact pointer tree](844-compact-pointer-tree.md)
The Compact Pointer Tree is a tree-based data structure designed for the interactive visualization of terabyte-scale volumetric RGBA data. It solves the problem of memory bandwidth and storage limitations by compressing color values using wavelet transforms and encoding spatial locality via compact 8-bit offsets. It sets itself apart from standard octrees by using a variable branching factor ($N^3$) and storing min/avg/max color attributes to enable lossy compression of the color space, allowing for on-demand voxel reconstruction from compressed nodes.

### [Sparse Voxel DAGs](2791-sparse-voxel-dags.md)
Sparse Voxel DAGs (SVDAGs) are a memory-efficient 3D data structure that extends the Sparse Voxel Octree (SVO) by merging identical subtrees into shared nodes, forming a Directed Acyclic Graph [898-bottom-up-svo-to-dag-reduction.md]. While SVOs efficiently encode spatial sparsity, they store redundant copies of identical non-empty geometry, leading to high memory consumption at high resolutions [898-bottom-up-svo-to-dag-reduction.md]. SVDAGs solve this by allowing multiple parent nodes to share pointers to the same child instance, achieving lossless compression of topological redundancy without decompression overhead during traversal [898-bottom-up-svo-to-dag-reduction.md]. This sets them apart from siblings like [Compact pointer tree](844-compact-pointer-tree.md) or [OpenVDB (VDB)](1508-openvdb-vdb.md), which focus on spatial sparsity or pointer compression rather than structural isomorphism [G].

### [Octree-based structures](2792-octree-based-structures.md)
Octree-based structures represent volumetric data using a recursive spatial subdivision where empty or uniform regions are collapsed into single nodes, providing compression proportional to data sparsity. This family solves the memory and bandwidth bottlenecks of dense voxel grids by enabling efficient culling of empty space and adaptive resolution. Unlike sibling structures like Pointer Trees or Sparse Voxel DAGs, standard octrees rely on a fixed 8-way branching factor and explicit or implicit parent-child pointers, prioritizing hierarchical traversal and spatial locality over the deduplication or pointer-free compactness of their variants.

### [Span Space Tree](4379-span-space-tree.md)
The Span Space Tree is a hierarchical data structure designed to accelerate the rendering and extraction of large-scale volumetric data by reducing the number of visited cells during traversal. It refines the interval tree concept to manage scene complexity for volumes larger than standard interactive resolutions. It sets itself apart from sibling structures like the Sparse Voxel Octree (SVO) by explicitly organizing data along axis-aligned spans (intervals) rather than just voxel occupancy, allowing for more efficient culling of empty or uniform regions during ray casting or surface extraction.

### [Sparse Voxel Tree (SVO Tree)](4983-sparse-voxel-tree-svo-tree.md)
A Sparse Voxel Tree (SVO) is a hierarchical data structure for sparse voxel grids where the 3D domain is recursively subdivided, and each internal node has up to 8 children (one for each octant). It solves the problem of storing and querying large, sparse 3D volumes by omitting empty regions, providing compression proportional to the sparsity of the data [2792-octree-based-structures.md]. Unlike its sibling [Sparse Voxel DAGs](2791-sparse-voxel-dags.md), which merge identical nodes to save space, the SVO maintains a strict tree hierarchy where nodes are distinct, prioritizing traversal simplicity and dynamic update capabilities over maximum compression.

### [Tiny LOD](6968-tiny-lod.md)
Tiny LOD is a method for constructing a hierarchical level-of-detail (LOD) representation from a set of input splats, specifically designed for integration with sparse voxel tree structures. It solves the problem of efficiently managing geometric detail across varying distances by generating a compressed tree structure that can be encoded for opacity values and subsequently used to build a chunk tree. It is distinguished from sibling methods by its specific parameterization via `lod_base` and `merge_filter`, and its role as a preprocessing step for opacity encoding in the Spark ecosystem.

### [NerfTree](7559-nerftree.md)
NerfTree is a two-level octree structure designed to accelerate Neural Radiance Fields (NeRF) by localizing surface geometry and refining radiance estimation only in relevant regions. Unlike standard NeRF which evaluates the network along the entire ray, NerfTree uses a coarse density volume to cache approximate geometry and fine sparse voxel grids for refinement, effectively skipping empty space to reduce MLP evaluations. It sets itself apart from other sparse structures by its specific two-level hierarchy optimized for the NeRF training pipeline, trading increased memory footprint for significant reductions in training time and inference latency.

### [Adaptively Sampled Distance Fields](7773-adaptively-sampled-distance-fields.md)
Adaptively Sampled Distance Fields (ASDF) are a tree-based compression method for representing 3D shapes and vector outlines by encoding a signed distance field (SDF) rather than binary occupancy. Unlike traditional binary-valued trees that require fine refinement to capture smooth boundaries, ASDF exploits the smoothness of the distance function to achieve high compression ratios while preserving the topological accuracy of the shape's zero set. It is particularly suited for applications requiring resolution-independent antialiasing and magnification of complex vector outlines.

## Background
Tree-based sparse structures originated from the need to handle large volumetric datasets that exceeded available memory when stored as dense grids. Early work focused on Octrees for efficient spatial partitioning [2792-octree-based-structures.md]. The Sparse Voxel Octree (SVO) became a standard for high-resolution volume rendering by explicitly managing sparsity [4983-sparse-voxel-tree-svo-tree.md]. Subsequent developments introduced Directed Acyclic Graphs (DAGs) to eliminate redundant subtrees [2791-sparse-voxel-dags.md], and compact pointer variants to optimize memory bandwidth for interactive rendering [844-compact-pointer-tree.md]. More recently, these structures have been adapted for Neural Radiance Fields (NeRF) to accelerate training and inference [7559-nerftree.md] and for signed distance fields to preserve topological accuracy [7773-adaptively-sampled-distance-fields.md].

## Sources
[S1] Spark Ecosystem Documentation, https://github.com/spark-ecosystem, 2023-10-01
[S2] Sparse Voxel Tree Implementation Notes, https://github.com/voxel-svo, 2023-10-01
[S3] SVO Traversal Performance Analysis, https://github.com/perf-svo, 2023-10-01
