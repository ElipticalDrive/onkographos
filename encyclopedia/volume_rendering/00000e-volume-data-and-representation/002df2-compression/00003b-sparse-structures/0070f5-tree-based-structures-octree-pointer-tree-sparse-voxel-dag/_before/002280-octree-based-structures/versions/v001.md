# Octree-based structures

Also called: Implicit Octree, OpenVDB, Tree64, DAOG

## Overview
Octree-based structures organize sparse volumetric data into a hierarchical tree where each node has eight children, enabling efficient storage and traversal of large datasets with significant empty space. This family differs from sibling tree-based structures by its fixed 8-way branching factor and specific implementation strategies for memory layout (implicit vs. explicit) and topology (shallow fixed-depth vs. deep variable-depth). Unlike [Sparse Voxel DAGs](2791-sparse-voxel-dags.md) which merge identical nodes to save space, or [Compact pointer trees](844-compact-pointer-tree.md) which use pointer compression, octrees typically rely on Morton ordering or fixed bit-masks for navigation.

## Shared principles
All octree-based structures share the fundamental property of recursive spatial subdivision. The 3D domain is divided into a bounding box, which is recursively split into eight octants until a stopping criterion (such as uniform data density or maximum depth) is met.

1.  **Morton Ordering (Z-order curve):** Many implementations, particularly implicit ones, map 3D coordinates $(x, y, z)$ to a 1D integer key using Morton codes. This preserves spatial locality, allowing linear arrays to represent tree structures and enabling efficient neighbor lookups and range queries.
2.  **Sparsity Handling:** Empty or uniform regions are represented by a single node (or null pointer in implicit structures) rather than storing individual voxel values, providing compression ratios proportional to the sparsity of the data.
3.  **Traversal:** Accessing data requires traversing from the root to the leaf. The complexity is $O(\log N)$ for deep trees or $O(1)$ for shallow trees with direct indexing, where $N$ is the resolution.

## Subtopics

### [Implicit Octree Storage](939-implicit-octree-storage.md)
Focuses on storing the octree hierarchy and data in a linear array or hash map using Morton keys, avoiding explicit node objects and parent pointers. It is chosen when memory overhead of node objects is prohibitive or when cache-friendly linear access is prioritized over pointer chasing.

### [Implicit Octree Traversal](1220-implicit-octree-traversal.md)
A method for constructing sparse octrees from distance fields to facilitate isosurface extraction, specifically targeting dual contouring or marching cubes on sparse data. It differs from explicit octree structures by using Morton coordinates to implicitly define the tree hierarchy, allowing for efficient point location and neighbor retrieval without storing explicit parent pointers or full node metadata. Unlike [Implicit Octree Storage](939-implicit-octree-storage.md), which focuses on the storage layout, this method defines the traversal algorithm to build the primal vertex map and locate dual vertices for mesh generation.

### [OpenVDB (VDB)](1508-openvdb-vdb.md)
OpenVDB is a sparse voxel representation used for the hierarchical encoding and compression of volumetric grids, featuring fast $O(1)$ random access for modification and retrieval operations. It solves the problem of efficiently storing and accessing large, sparse volumetric datasets by using a shallow tree structure that balances memory overhead with traversal speed. It is distinguished from its siblings by its specific four-level topology and 64-bit direct access bit masks that encode topology and data pointers, making it a standard in production visual effects and scientific visualization.

### [Time-Variant Volume Rendering with Octrees](2232-time-variant-volume-rendering-with-octrees.md)
Rendering technique for time-variant volumes using octree compression to fit multiple timesteps into main memory, overcoming GPU memory and bus bandwidth limitations.

### [Domain-based Octree with Range Bounding Boxes](2806-domain-based-octree-with-range-bounding-boxes.md)
An acceleration structure using an octree over the domain to store interval ranges (RBBs) at each node, allowing filtering of tetrahedra that cannot intersect a given FSCP edge by comparing the edge against the node's RBB.

### [Tree64](3486-tree64.md)
A sparse voxel tree with a 4³ branching factor, also known as a contree.

### Dynamic Adaptive Octree Grid (DAOG)
#### definition
- [S1, unofficial] A voxel representation structure that offers significant memory savings compared to Regular Grid and standard Octree, allowing representation of large surfaces with little space.
#### number
- [S1, unofficial] For a 1024x1024x1024 Cornell Box model: Regular Grid uses 1048576 KB, Octree uses 158087 KB, DAOG uses 269 KB.
- [S1, unofficial] For a 1024x1024x1024 Batman model: Regular Grid uses 1048576 KB, Octree uses 106303 KB, DAOG uses 813 KB.
- [S1, unofficial] Construction time for 1024x1024x1024 Cornell Box: Octree takes 5.153s, DAOG takes 67.680s, DAOG serialized takes 0.142ms.

## Background
The concept of octrees for volume representation dates back to the early days of 3D computer graphics, with foundational work by Meagher (1982) and others. OpenVDB, originally developed by DreamWorks Animation as "Grid," was introduced in 2012 to handle large-scale VFX data [S1, unofficial]. The Dynamic Adaptive Octree Grid (DAOG) is a more recent variant discussed in academic studies on compression efficiency, highlighting extreme memory savings for specific surface representations [S1, unofficial].

## Sources
[S1] Estudio de la compresión y la eficiencia del Sparse Voxel Octree en el cálculo de la Iluminación Global, https://hdl.handle.net/2117/445971, fetched 2026-10-26 (unofficial)
