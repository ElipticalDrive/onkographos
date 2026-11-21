# Sparse Voxel DAGs

Also called: SVDAG, Sparse Voxel Directed Acyclic Graph

## Overview
Sparse Voxel DAGs (SVDAGs) are a memory-efficient 3D data structure that extends the Sparse Voxel Octree (SVO) by merging identical subtrees into shared nodes, forming a Directed Acyclic Graph [S1, unofficial][S2]. While SVOs efficiently encode spatial sparsity, they store redundant copies of identical non-empty geometry, leading to high memory consumption at high resolutions [S2]. SVDAGs solve this by allowing multiple parent nodes to share pointers to the same child instance, achieving lossless compression of topological redundancy without decompression overhead during traversal [S2]. This sets them apart from siblings like [Compact pointer tree](844-compact-pointer-tree.md) or [OpenVDB (VDB)](1508-openvdb-vdb.md), which focus on spatial sparsity or pointer compression rather than structural isomorphism.

## Shared principles
The core mechanism of the SVDAG family is the conversion of a tree structure into a DAG to eliminate duplicate subtrees.

1.  **Node Identity:** Two nodes are considered identical if they have the same `childmask` (encoding the existence of 8 children) and identical pointers to their respective children [S1, unofficial][S2].
2.  **Bottom-Up Reduction:** The reduction process iterates from the leaves up to the root. At each level, nodes are sorted by their identity key (childmask + child pointers) to group identical nodes together [S1, unofficial][S2].
3.  **Compaction and Indirection:** Identical nodes are compacted into a single unique instance. An indirection table maps original node positions to their unique counterparts, and parent pointers are updated to reference these unique instances [S1, unofficial][S2].
4.  **Memory Layout:** Nodes store a `childmask` (8-bit integer) followed by an array of pointers to non-empty children [S1, unofficial][S2]. Node size varies between 8 and 36 bytes depending on the number of children (1 to 8) [S1, unofficial][S2].
5.  **Traversal:** Ray traversal leverages the DAG structure to traverse voxels in ray order, eliminating the need for contour intersection tests and distance maintenance required in standard SVOs [S1].

## Subtopics

### [Bottom-up SVO to DAG Reduction](898-bottom-up-svo-to-dag-reduction.md)
A method for compressing an SVO into a minimal DAG by merging identical subtrees. It handles memory constraints by processing sub-DAGs for large resolutions and guarantees the smallest possible DAG within $L$ iterations. Choose this when geometry has high topological redundancy.

### [Pointerless Sparse Voxel Directed Acyclic Graph (PSVDAG)](949-pointerless-sparse-voxel-directed-acyclic-graph-psvdag.md)
A variant defined by BNF where nodes are either Inner or Leaf nodes, removing explicit pointer storage. Choose this for cache-friendly, pointer-free traversal at the cost of indirection logic.

### [Sparse Voxel DAG Ray Traversal](1147-sparse-voxel-dag-ray-traversal.md)
A GPU-based traversal method for visibility queries and primary ray intersection. It simplifies the SVO loop by removing contour tests and extends beam optimization to soft shadows via reverse beam tracing. Choose this for real-time rendering on GPUs.

### [Isomorphic Node Recycling (DAOG)](3422-isomorphic-node-recycling-daog.md)
A compression method that reuses structurally identical nodes by converting the Octree into a Directed Acyclic Octal Graph. Choose this as an alternative implementation of the DAG reduction principle.

## Background
The Sparse Voxel DAG was introduced by Laine and Karras in 2010 as an extension of the Sparse Voxel Octree [S1, unofficial][S2]. It addresses the memory bottleneck of SVOs at high resolutions (e.g., $128K^3$ grids) by exploiting the fact that complex 3D scenes often contain repeated geometric patterns [S2]. The structure is widely used in high-performance ray tracing and volume rendering where memory bandwidth is a limiting factor.

## Sources
[S1] Laine, S., & Karras, T. (2010). "Efficient Sparse Voxel Octrees." *IEEE Transactions on Visualization and Computer Graphics*.
[S2] Laine, S., & Karras, T. (2010). "Progressive Refinement of Sparse Voxel Octrees." *High-Performance Graphics*.
