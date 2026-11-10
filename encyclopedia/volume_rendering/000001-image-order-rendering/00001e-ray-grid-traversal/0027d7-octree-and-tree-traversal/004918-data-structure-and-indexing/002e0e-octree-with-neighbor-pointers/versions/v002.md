# Octree with Neighbor Pointers

Also called: Octree Construction with X-Neighbor Pointers, Octree with neighbors structure

## Overview
A spatial index structure that augments a branch-on-need octree with links to allow direct traversal to adjacent nodes, used to accelerate ray traversal in volume rendering [S2]. It differs from Zhou et al. [2010] by storing different payload (mesh vs point cloud) and maintaining only x-neighbor pointers instead of pointers to all 26 neighbors [S1].

## How it works
**Inputs:** A view-dependent branch-on-need octree.
**Outputs:** A 3D texture acting as a spatial index, encoded with neighbor pointers.

**Data Structures:**
The spatial index is encoded into an 8-bit RGBA 3D texture [S2]. Each octree node is encoded in 8 consecutive texels arranged in the x-direction [S2]. The layout uses the following tagging scheme in the Alpha (A) component:
*   $A=1.0$: Inner node.
*   $A=0.5$: Data node (leaf).
*   $A=0.0$: Empty node.

**Inner Node Layout:**
The RGB components point to the first of 8 children, which are arranged consecutively in the x-direction [S2].

**Leaf Node Layout:**
The RGB components point to leaf info, consisting of:
1.  A data pointer (if not empty) [S2].
2.  6 consecutive texels storing neighbor pointers [S2].
The last leaf data texel contains a NULL pointer [S2]. Neighbor pointers use their A value to encode the octree level of the neighbor [S2].

**Construction Algorithm:**
The structure is created on-the-fly at each frame directly from the view-dependent octree [S2].

1.  **Initialization:** Create all nodes at the top level $n$ and establish x-neighbor pointers for all adjacent nodes [S1]. In practice, the process stops before reaching a single root node at a level where all nodes are typically present; two levels below the root are chosen to speed up traversal [S1].
2.  **Propagation:** Propagate x-neighbor pointers from nodes in level $i$ to their child nodes in level $i-1$ [S1]. This links adjacent level-$(i-1)$ nodes that have different parent nodes [S1]. The process proceeds bottom-up in a breadth-first fashion using Morton ordering [S1].
3.  **Encoding:** Encode the resulting structure into the 3D texture [S2]. Update the GPU texture using a `glTexSubImage` call [S2].

**Traversal Semantics:**
Neighbor pointers link each leaf node of the octree via its six faces directly to the corresponding adjacent node of that face, or to the smallest node enclosing all adjacent nodes if there are multiple ones [S2].

## Operating characteristics
Not given in the sources.

## Background
This method is used in interactive out-of-core ray casting frameworks [S2] and fast parallel surface and solid voxelization on GPUs [S1]. It shares similarities with the approach by Zhou et al. [2010], such as proceeding bottom-up in a breadth-first fashion and using Morton ordering, but differs in payload and adjacency information [S1].

## Sources
[S1] Fast Parallel Surface and Solid Voxelization on GPUs, https://www.michael-schwarz.com/research/publ/files/vox-siga10.pdf, fetched 2026-10-27
[S2] A single-pass GPU ray casting framework for interactive out-of-core ..., https://www.crs4.it/vic/data/papers/cgi2008-movr.pdf, fetched 2026-10-29
