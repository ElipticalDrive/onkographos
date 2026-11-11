# Octree with Neighbor Pointers

Also called: Octree Construction with X-Neighbor Pointers, Octree with neighbors structure

## Overview
This method augments a branch-on-need octree with explicit links to adjacent nodes, enabling direct traversal to neighboring spatial regions without backtracking through the parent hierarchy [S2]. It solves the latency penalty of recursive ascent in sparse octrees by allowing rays to jump horizontally across node boundaries [S1]. Unlike structures that maintain full 26-neighbor connectivity, this variant typically restricts pointers to the X-axis (or specific face directions) to minimize memory overhead, distinguishing it from comprehensive adjacency graphs [S1].

## How it works
The method constructs a spatial index on-the-fly from a view-dependent octree and encodes it into a 3D texture for GPU traversal [S2].

### Inputs and Outputs
*   **Input:** A view-dependent branch-on-need octree representing the volume or surface data.
*   **Output:** A 3D RGBA texture acting as the spatial index, where each node is encoded in 8 consecutive texels arranged in the X direction [S2].

### Data Structures
The spatial index is a 3D texture with 8-bit RGBA components [S2]. Each octree node occupies 8 consecutive texels along the X-axis [S2].

**Tagged Pointer Layout (per texel):**
*   **RGB:** Pointer information (offsets to children or neighbor data).
*   **A (Alpha):** Node kind tag [S2].
    *   $A = 1.0$: Inner node (non-leaf).
    *   $A = 0.5$: Data node (leaf with data).
    *   $A = 0.0$: Empty node.

**Inner Node Encoding:**
*   The RGB components of the first texel point to the first of 8 children, which are arranged consecutively in the X direction [S2].

**Leaf Node Encoding:**
*   The RGB components point to leaf info:
    1.  A data pointer (if not empty).
    2.  6 consecutive texels storing neighbor pointers for the 6 faces [S2].
*   The last leaf data texel contains a NULL pointer [S2].
*   Neighbor pointers use the A component to encode the octree level of the neighbor [S2].

### Construction Algorithm
The construction proceeds bottom-up in a breadth-first fashion, similar to Morton-ordering based approaches [S1].

1.  **Initialization:** Create all nodes at the top level $n$ (typically two levels below the root to speed up traversal) [S1].
2.  **X-Neighbor Establishment:** Establish X-neighbor pointers for all adjacent nodes at the top level $n$ [S1].
3.  **Propagation:** Propagate X-neighbor pointers from nodes in level $i$ to their child nodes in level $i-1$ [S1].
    *   Specifically, link adjacent level-$(i-1)$ nodes that have different parent nodes [S1].
4.  **Texture Encoding:** Encode the resulting structure into a 3D texture [S2].
    *   Update the GPU texture using `glTexSubImage` [S2].

### Traversal Logic
During ray casting, the structure allows direct access to adjacent nodes [S2].
*   **Face Linking:** Neighbor pointers link each leaf node via its six faces directly to the corresponding adjacent node of that face, or to the smallest node enclosing all adjacent nodes if multiple nodes share the face boundary [S2].
*   **Jumping:** When a ray exits a node, it uses the neighbor pointer to jump directly to the next potentially occupied node, bypassing parent backtracking [S2].

### Edge Cases and Pitfalls
*   **Boundary Nodes:** Neighbor pointers must correctly identify the "smallest node enclosing all adjacent nodes" when a face spans multiple child nodes of a parent [S2].
*   **Memory Limits:** The encoding supports addressing $16 \times M$ nodes or data blocks, where $M$ is the texture dimension, which is several orders of magnitude larger than typically needed [S2].
*   **Partial Connectivity:** Maintaining only X-neighbor pointers (as in [S1]) reduces memory but may require fallback mechanisms for Y/Z traversal if not fully implemented for all axes.

## Operating characteristics
*   **Time Complexity:** Construction is $O(N)$ where $N$ is the number of nodes, performed on-the-fly per frame [S2]. Traversal is accelerated by $O(1)$ neighbor jumps, reducing backtracking overhead [S2].
*   **Memory Complexity:** High overhead due to storing neighbor pointers for every leaf node and the 8-texel-per-node encoding [S2].
*   **Performance:** Enables interactive out-of-core rendering by minimizing CPU-GPU synchronization and traversal latency [S2].
*   **Strengths:** Eliminates recursive backtracking; efficient for sparse data [S2].
*   **Limitations:** Requires per-frame reconstruction; high memory bandwidth usage for texture updates [S2].

## Background
The concept of augmenting octrees with neighbor pointers was introduced to accelerate ray traversal in volume rendering [S2]. The specific construction method using X-neighbor pointers and bottom-up propagation was detailed by Schwarz et al. in the context of fast parallel surface and solid voxelization [S1]. This approach differs from Zhou et al. [2010] by storing different payloads (mesh vs. point cloud) and maintaining only X-neighbor pointers instead of all 26 neighbors [S1].

## Sources
[S1] Fast Parallel Surface and Solid Voxelization on GPUs, https://www.michael-schwarz.com/research/publ/files/vox-siga10.pdf, fetched 2026-10-27
[S2] A single-pass GPU ray casting framework for interactive out-of-core ..., https://www.crs4.it/vic/data/papers/cgi2008-movr.pdf, fetched 2026-10-29
