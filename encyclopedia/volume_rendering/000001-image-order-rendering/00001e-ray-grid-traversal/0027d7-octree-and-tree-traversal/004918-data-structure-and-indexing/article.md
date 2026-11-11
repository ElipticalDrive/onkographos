# Data Structure and Indexing

Also called: Spatial Indexing for Octrees, Octree Data Layouts

## Overview
This family defines how hierarchical volume data (typically octrees) is stored in memory and indexed to enable efficient ray traversal. It solves the problem of mapping irregular, sparse tree structures to linear memory or GPU textures while providing mechanisms to skip empty space and navigate between adjacent nodes. What sets this family apart from its siblings is the focus on the *storage format* and *indexing scheme* (e.g., texture encoding, binary arrays, neighbor pointers) rather than the traversal algorithm itself or the geometric primitives (like marching cubes) used at the leaves.

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
A spatial index structure that augments a branch-on-need octree with links to allow direct traversal to adjacent nodes, used to accelerate ray traversal in volume rendering. It differs from Zhou et al. [2010] by storing different payload (mesh vs point cloud) and maintaining only x-neighbor pointers instead of pointers to all 26 neighbors.

*   **When to choose:** Use when neighbor traversal is required to avoid backtracking overhead or to handle discontinuities in the volume data, specifically in view-dependent branch-on-need octrees.
*   **Link:** [Octree with Neighbor Pointers](5699-octree-with-neighbor-pointers.md)

### Perspective-Correct AMR Patch Traversal
A hybrid CPU-GPU traversal strategy where the CPU traverses the KD-tree of AMR patches back-to-front, and the GPU traverses cells within a patch. Sub-patches are generated to ensure perspective-correct sorting of cells relative to the observer.

*   **Definition:** A hybrid approach where the CPU handles the coarse hierarchy (KD-tree of AMR patches) and the GPU handles fine-grained cell traversal within patches [S1].
*   **Step:**
    1.  CPU traverses the KD-tree hierarchy in back-to-front fashion, choosing the appropriate order at each internal node until a leaf node (patch) is found [S1].
    2.  Vertex shader modifies each index according to the current observer position to generate proper cell ordering, ensuring perspective-correct sorting [S1].
*   **When to choose:** Use for Adaptive Mesh Refinement (AMR) data where perspective-correct ordering of cells within a patch is critical for correct rendering, particularly when the observer position changes significantly relative to the patch geometry.
*   **Link:** N/A (Minor variant)

### Gradient Octree Transfer
Transfers gradient octrees computed on the server side to a rendering client to reduce network traffic and avoid the need to compute gradients on the client.

*   **Step:** Server computes gradients and transfers the resulting octree structure to the client [S2, unofficial].
*   **When to choose:** Use in client-side rendering scenarios (e.g., web-based) where client-side gradient computation is too expensive or bandwidth is limited, and the server can pre-compute the necessary derivative information.
*   **Link:** N/A (Minor variant)

### Binary Tree Indexing
A simple binary tree indexing scheme to identify and organize patches in a breadth-first manner (coarse to fine). Allows quick identification of cousin patches. View-dependent mesh generation uses recursive traversal over the patch hierarchy by indexing into a fixed-size 1D array of patches, accumulating patch IDs for later decompression and rendering.

*   **Definition:** A binary tree indexing scheme that organizes patches from coarse to fine resolution, enabling quick identification of cousin patches and efficient view-dependent mesh generation via recursive traversal over a fixed-size 1D array [S3].
*   **When to choose:** Use for terrain or height-field data where patches are organized in a binary hierarchy and efficient decompression/rendering of view-dependent meshes is required.
*   **Link:** N/A (Minor variant)

### Point Octree
A recursive octree data structure for managing 3D points, supporting insertion, removal, retrieval, movement, and spatial queries (nearest neighbor, radius search).

*   **Definition:** A recursive octree for managing 3D points with support for dynamic operations and spatial queries [S4, unofficial].
*   **Step:**
    *   **Insertion (`set`):** Recursively finds the leaf octant containing the point. If the leaf is full and depth < maxDepth, it calls `split()` and `redistribute()`. If the point already exists, it updates the data array. Otherwise, it appends the point and data to the leaf arrays [S4, unofficial].
    *   **Radius Search (`findPoints`):** Recursively traverses child octants that intersect the query sphere. In leaf nodes, computes squared distance `dSq` and compares against `radius * radius` to filter points [S4, unofficial].
*   **Parameter:**
    *   `bias`: Octant boundary bias. Values > 0 make the octree "loose" to handle points near boundaries [S4, unofficial].
    *   `maxPoints`: Maximum number of distinct points allowed per leaf octant before triggering a split [S4, unofficial].
    *   `maxDepth`: Maximum recursion depth for the tree. Prevents infinite subdivision [S4, unofficial].
*   **Layout:** `PointData<T>`: Stores two parallel arrays: `points: Vector3[]` for coordinates and `data: T[]` for associated user data [S4, unofficial].
*   **Pitfall:** `maxDepth` acts as a hard limit to prevent infinite recursion and poor performance from infinitely small octants [S4, unofficial].
*   **When to choose:** Use for dynamic point cloud data where frequent insertions, deletions, and spatial queries (nearest neighbor, radius search) are required.
*   **Link:** N/A (Minor variant)

## Background
The use of hierarchical data structures for volume rendering dates back to the early 1990s, with octrees being a primary choice for representing sparse volumetric data. Early work focused on CPU-based traversal using linked lists or arrays [S1]. With the advent of GPUs, texture-based encoding of octrees became popular to leverage hardware texture sampling for traversal [S2, unofficial]. The introduction of neighbor pointers allowed for more efficient traversal by avoiding backtracking [S1]. More recent developments include hybrid CPU-GPU approaches for AMR data [S1] and client-side rendering techniques that transfer pre-computed structures from servers [S2, unofficial].

## Sources
[S1] High-Quality, Semi-Analytical Volume Rendering for AMR ..., https://people.freedesktop.org/~marcheu/vis09-amr.pdf, fetched 2026-10-26
[S2] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
[S3] On-the-fly decompression and rendering of multiresolution terrain, https://doi.org/10.1145/1730804.1730815, fetched 2026-11-01
[S4] sparse-octree module, https://github.com/vanruesc/sparse-octree, fetched 2026-10-27 (unofficial)
