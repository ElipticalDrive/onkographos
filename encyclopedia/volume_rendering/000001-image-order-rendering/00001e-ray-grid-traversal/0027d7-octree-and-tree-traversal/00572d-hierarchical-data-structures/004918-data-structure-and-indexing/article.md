# Data Structure and Indexing

Also called: Hierarchical Data Structures Indexing

## Overview
This family defines the memory layout, pointer encoding, and traversal logic for hierarchical acceleration structures (primarily octrees) used in volume rendering. It solves the problem of efficiently navigating sparse 3D data by mapping tree topology to linear memory or GPU textures, enabling fast empty-space skipping and neighbor-based boundary crossing. It differs from sibling methods by focusing on the low-level representation of the tree hierarchy itself (e.g., texture encoding, point storage) rather than high-level traversal optimizations or specific acceleration heuristics.

## Shared principles
All methods in this family share the core mechanism of hierarchical descent and ascent [G]. The traversal begins at the root of the octree and determines which of the eight children (octants) the ray intersects. This is typically done by comparing the ray's position relative to the center of the current node's bounding box [S1][S2, unofficial][G].

1.  **Child Selection:** For a node with center $\mathbf{c}$ and half-extent $s$, the ray position $\mathbf{p}$ determines the child index $k \in \{0, \dots, 7\}$:
    $$
    k = 4 \cdot \mathbb{I}(p_x \ge c_x) + 2 \cdot \mathbb{I}(p_y \ge c_y) + 1 \cdot \mathbb{I}(p_z \ge c_z)
    $$
    where $\mathbb{I}(\cdot)$ is the indicator function [S1][S2, unofficial][G].

2.  **Empty Space Skipping:** If the selected child is marked as empty (or uniform), the traversal skips the subtree [G]. This is achieved by advancing the ray parameter $t$ to the exit of the current node or using neighbor pointers to jump to the next potentially occupied node [S1][G].

3.  **Leaf Handling:** Upon reaching a leaf node, the method typically switches to a fixed-step traversal (like DDA) within the leaf's voxel grid to sample the volume data [S1][G].

4.  **Backtracking:** Because octrees are sparse and irregular, traversal often requires backtracking to the parent node when a ray exits a subtree or encounters an empty branch [G]. This is implemented using an explicit stack (on GPU) or recursion (on CPU) [G].

## Subtopics

### [Octree with Neighbor Pointers](5699-octree-with-neighbor-pointers.md)
A spatial index structure that augments a branch-on-need octree with links to allow direct traversal to adjacent nodes, used to accelerate ray traversal in volume rendering. It differs from full 26-neighbor structures by maintaining only x-neighbor pointers, reducing memory overhead while enabling efficient skipping of empty space and boundary crossing. The structure is encoded into an 8-bit RGBA 3D texture. Each octree node is encoded in 8 consecutive texels arranged in the x-direction. The layout uses a tagged pointer scheme where the Alpha (A) component determines the node kind: $A=1.0$ for inner nodes, $A=0.5$ for data nodes (leaves with data), and $A=0.0$ for empty nodes. Inner node RGB components point to the first of 8 children, arranged consecutively in the x-direction. Leaf node RGB components point to leaf info, consisting of a data pointer and 6 consecutive texels storing neighbor pointers for the 6 faces. The last leaf data texel contains a NULL pointer. Neighbor pointers use their A value to encode the octree level of the neighbor. Construction involves initializing top-level nodes (typically two levels below the root) and establishing x-neighbor pointers, propagating these pointers down to child nodes that share boundaries but have different parents, and finally encoding the structure into a 3D texture. Boundary nodes link to the smallest node enclosing all adjacent nodes if multiple exist. Traversal stops before reaching a single root node at the top level to optimize performance. See [Octree with Neighbor Pointers](5699-octree-with-neighbor-pointers.md).

### [Point Octree](11339-point-octree.md)
A Point Octree is a recursive hierarchical data structure designed to manage a dynamic set of 3D points, supporting insertion, removal, movement, and spatial queries such as nearest neighbor and radius search. It differs from voxel-based octrees by storing data only at leaf nodes corresponding to occupied regions, allowing for efficient handling of sparse point clouds without allocating memory for empty space. The tree consists of nodes containing an array of up to 8 child octants and a `PointData` object at leaves. `PointData` uses two parallel arrays: `points` (Vector3[]) and `data` (T[]). Key parameters include `maxPoints` (leaf capacity), `maxDepth` (recursion limit), and `bias` (padding for boundary checks). Insertion recursively finds the leaf, updates existing points, or splits the node if full, redistributing points to children using a bias for containment checks. Radius search recursively traverses intersecting octants and filters leaves by squared distance. Movement updates positions in place or removes/reinserts if the octant changes. See [Point Octree](11339-point-octree.md).

### Perspective-Correct AMR Patch Traversal
A hybrid CPU-GPU traversal strategy where the CPU traverses the KD-tree of AMR patches back-to-front, and the GPU traverses cells within a patch. Sub-patches are generated to ensure perspective-correct sorting of cells relative to the observer [S1]. The CPU traverses the KD-tree hierarchy in back-to-front fashion, choosing the appropriate order at each internal node until a leaf node (patch) is found [S1]. The vertex shader modifies each index according to the current observer position to generate proper cell ordering, ensuring perspective-correct sorting [S1].

### Gradient Octree Transfer
Transfers gradient octrees computed on the server side to a rendering client to reduce network traffic and avoid the need to compute gradients on the client [S2, unofficial].

### Binary Tree Indexing
A simple binary tree indexing scheme to identify and organize patches in a breadth-first manner (coarse to fine). Allows quick identification of cousin patches. View-dependent mesh generation uses recursive traversal over the patch hierarchy by indexing into a fixed-size 1D array of patches, accumulating patch IDs for later decompression and rendering [S3].

## Background
The use of hierarchical data structures for volume rendering dates back to the early 1990s, with octrees being a primary choice for representing sparse volumetric data [G]. Early work focused on CPU-based traversal using linked lists or arrays [G]. With the advent of GPUs, texture-based encodings became prevalent to leverage hardware texture units for pointer chasing and empty-space skipping [S2, unofficial]. The specific encoding of octrees into 3D textures with tagged pointers was developed to facilitate efficient traversal on graphics hardware [S2, unofficial]. Neighbor pointer techniques were introduced to handle boundary crossings efficiently without backtracking [S1]. Point octrees are widely used in computer graphics for managing particle systems and dynamic point clouds [S1]. AMR (Adaptive Mesh Refinement) structures, often indexed via binary trees or KD-trees, are used in scientific visualization to handle multi-resolution data [S1, S3].

## Sources
[S1] High-Quality, Semi-Analytical Volume Rendering for AMR ..., https://people.freedesktop.org/~marcheu/vis09-amr.pdf, fetched 2026-10-26
[S2] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
[S3] On-the-fly decompression and rendering of multiresolution terrain, https://doi.org/10.1145/1730804.1730815, fetched 2026-11-01
