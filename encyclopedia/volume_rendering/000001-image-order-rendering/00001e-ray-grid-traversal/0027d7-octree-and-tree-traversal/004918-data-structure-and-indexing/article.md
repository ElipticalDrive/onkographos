# Data Structure and Indexing

Also called: Spatial Indexing for Octrees

## Overview
Data Structure and Indexing defines how hierarchical volume data (typically octrees) is organized in memory and accessed during ray traversal. It solves the problem of efficiently mapping spatial coordinates to volume samples while minimizing memory bandwidth and traversal overhead. Unlike sibling methods that focus on traversal algorithms or rendering techniques, this family focuses on the **storage layout** (e.g., texture encoding, array indexing) and **connectivity** (e.g., neighbor pointers) required to support those algorithms. It differs from [Ray Traversal Algorithms](7918-ray-traversal-algorithms.md) by defining *where* data lives, whereas traversal algorithms define *how* to move through it.

## Shared principles
All methods in this family share the core mechanism of hierarchical descent and ascent. The traversal begins at the root of the octree and determines which of the eight children (octants) the ray intersects. This is typically done by comparing the ray's position relative to the center of the current node's bounding box [S1][S2, unofficial][S6, S8, unofficial].

1.  **Child Selection:** For a node with center $\mathbf{c}$ and half-extent $s$, the ray position $\mathbf{p}$ determines the child index $k \in \{0, \dots, 7\}$:
    $$
    k = 4 \cdot \mathbb{I}(p_x \ge c_x) + 2 \cdot \mathbb{I}(p_y \ge c_y) + 1 \cdot \mathbb{I}(p_z \ge c_z)
    $$
    where $\mathbb{I}(\cdot)$ is the indicator function [S1][S2, unofficial][S6, S8, unofficial].

2.  **Empty Space Skipping:** If the selected child is marked as empty (or uniform), the traversal skips the subtree. This is achieved by advancing the ray parameter $t$ to the exit of the current node or using neighbor pointers to jump to the next potentially occupied node [S1, S6, S8, unofficial][G].

3.  **Leaf Handling:** Upon reaching a leaf node, the method typically switches to a fixed-step traversal (like DDA) within the leaf's voxel grid to sample the volume data [S1, S6, S8, unofficial][G].

4.  **Backtracking:** Because octrees are sparse and irregular, traversal often requires backtracking to the parent node when a ray exits a subtree or encounters an empty branch. This is implemented using an explicit stack (on GPU) or recursion (on CPU) [S4, S6, S8, unofficial].

## Subtopics

### [Octree with Neighbor Pointers](5699-octree-with-neighbor-pointers.md)
This method augments a branch-on-need octree with explicit links to adjacent nodes, enabling direct traversal to neighboring spatial regions without backtracking through the parent hierarchy. It solves the latency penalty of recursive ascent in sparse octrees by allowing rays to jump horizontally across node boundaries. Unlike structures that maintain full 26-neighbor connectivity, this variant typically restricts pointers to the X-axis (or specific face directions) to minimize memory overhead, distinguishing it from comprehensive adjacency graphs.

### Perspective-Correct AMR Patch Traversal
#### definition
- [S1] A hybrid CPU-GPU traversal strategy where the CPU traverses the KD-tree of AMR patches back-to-front, and the GPU traverses cells within a patch. Sub-patches are generated to ensure perspective-correct sorting of cells relative to the observer.
#### step
- [S1] CPU traverses the KD-tree hierarchy in back-to-front fashion, choosing the appropriate order at each internal node until a leaf node (patch) is found.
- [S1] Vertex shader modifies each index according to the current observer position to generate proper cell ordering, ensuring perspective-correct sorting.

### Gradient Octree Transfer
#### step
- [S2, unofficial] Transfers gradient octrees computed on the server side to a rendering client to reduce network traffic and avoid the need to compute gradients on the client.

### Binary Tree Indexing
#### definition
- [S3] A simple binary tree indexing scheme to identify and organize patches in a breadth-first manner (coarse to fine). Allows quick identification of cousin patches. View-dependent mesh generation uses recursive traversal over the patch hierarchy by indexing into a fixed-size 1D array of patches, accumulating patch IDs for later decompression and rendering.

### Point Octree
#### definition
- [S4, unofficial] A recursive octree data structure for managing 3D points, supporting insertion, removal, retrieval, movement, and spatial queries (nearest neighbor, radius search).
#### step
- [S4, unofficial] Insertion (`set`): Recursively finds the leaf octant containing the point. If the leaf is full and depth < maxDepth, it calls `split()` and `redistribute()`. If the point already exists, it updates the data array. Otherwise, it appends the point and data to the leaf arrays.
- [S4, unofficial] Radius Search (`findPoints`): Recursively traverses child octants that intersect the query sphere. In leaf nodes, computes squared distance `dSq` and compares against `radius * radius` to filter points.
#### parameter
- [S4, unofficial] bias: Octant boundary bias. Values > 0 make the octree "loose" to handle points near boundaries.
- [S4, unofficial] maxPoints: Maximum number of distinct points allowed per leaf octant before triggering a split.
- [S4, unofficial] maxDepth: Maximum recursion depth for the tree. Prevents infinite subdivision.
#### layout
- [S4, unofficial] PointData<T>: Stores two parallel arrays: `points: Vector3[]` for coordinates and `data: T[]` for associated user data.
#### pitfall
- [S4, unofficial] `maxDepth` acts as a hard limit to prevent infinite recursion and poor performance from infinitely small octants.

## Background
The use of hierarchical data structures for volume rendering originated with the need to handle large, sparse datasets efficiently. Early works focused on octree-based traversal to skip empty space [S1]. The specific encoding of octrees into 3D textures for GPU acceleration was developed to leverage hardware texture sampling for pointer chasing [S2, unofficial]. Hybrid CPU-GPU approaches for AMR (Adaptive Mesh Refinement) data were introduced to handle perspective-correct rendering of non-uniform grids [S1]. Binary tree indexing for terrain patches was proposed to simplify view-dependent LOD management [S3]. Point-based octrees are widely used in general-purpose graphics libraries for spatial queries [S4, unofficial].

## Sources
[S1] High-Quality, Semi-Analytical Volume Rendering for AMR ..., https://people.freedesktop.org/~marcheu/vis09-amr.pdf, fetched 2026-10-26
[S2] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
[S3] On-the-fly decompression and rendering of multiresolution terrain, https://doi.org/10.1145/1730804.1730815, fetched 2026-11-01
[S4] sparse-octree module, https://github.com/vanruesc/sparse-octree, fetched 2026-10-27 (unofficial)
