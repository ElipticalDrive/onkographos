# Parallel Processing and Pipeline Optimization

Also called: Parallel DMC Pipeline, Dual Marching Cubes Counting Pass

## Overview
This family defines a two-pass parallel surface extraction algorithm for regular 3D grids that generates a manifold quadrilateral mesh with full 1-ring connectivity [S1]. It solves the problem of dynamic buffer allocation and unconnected primitives inherent in standard parallel extraction by separating geometry sizing from generation [S1]. Unlike standard parallel Marching Cubes which produces redundant fragments, this method ensures manifold connectivity through a multi-staged pipeline that switches from block-based to cell-based processing [S1].

## Shared principles
The pipeline operates on a regular grid $L$ with spacing factor 4:
$$L = \{(i \cdot 4 \cdot x, j \cdot 4 \cdot y, k \cdot 4 \cdot z) : i = 0,...,N_x; j = 0,...,N_y; k = 0,...,N_z\}$$
Cell $(i,j,k)$ is defined by the diagonal from $(i \cdot 4 \cdot x, j \cdot 4 \cdot y, k \cdot 4 \cdot z)$ to $((i+1) \cdot 4 \cdot x, (j+1) \cdot 4 \cdot y, (k+1) \cdot 4 \cdot z)$ [S1].

The process consists of two main phases:
1.  **Counting Pass:** Iterates over the voxel grid to determine exact memory requirements for vertex and quad buffers without generating geometry [S1]. It uses static lookup tables `DM_VERT_COUNT[256]` and `DM_QUAD_COUNT[256]` to map cube configuration masks (0-255) to counts [S1].
2.  **Generation Pass:** Generates the mesh using parallel reduction algorithms and lookup tables to ensure connectivity [S1].

Key data structures include:
*   **Static Lookup Tables:** `DM_VERT_COUNT`, `DM_QUAD_COUNT`, `EDGE_TO_CELLS_EDGES`, `EDGE_TO_CELLS_OFFS` [S1].
*   **Dynamic Tables:** `Case Table` (stores cell cases), `Index Table` (maps cells to buffer locations via parallel prefix-sum/scan) [S1].
*   **Connectivity Tables:** `Vertex-Edges Table`, `Edge-Vertex Table`, `Vertex-Adjacencies Table`, `Face Table` [S1].

The pipeline handles non-manifold edges (tunnels covering two neighbor cells) by generating a valid halfedge data structure [S1].

## Subtopics

### [Dual Marching Cubes (Counting Pass)](3766-dual-marching-cubes-counting-pass.md)
The first phase of the algorithm. It determines exact memory requirements for vertex and quad buffers. It iterates over a voxel grid, classifying cells and accumulating counts based on precomputed lookup tables. It is read-only regarding output buffers.

### [Parallel Dual Marching Cubes (DMC) Pipeline](6387-parallel-dual-marching-cubes-dmc-pipeline.md)
The complete surface extraction method. It generates a manifold quadrilateral mesh with full connectivity information (1-ring neighbors). It combines static and dynamic lookup tables with parallel reduction algorithms. It builds on the Counting Pass but extends it for interactive performance on parallel hardware.

### Parallel Vertex and Face Coloring
#### definition
A scheme to encode neighborhood and connectivity information in the DMC mesh, enabling parallel processing and simplification [S1].
#### usage
Used to facilitate parallel simplification methods by identifying irregular elements based on face colors [S1].

## Background
Originated in "A parallel dual marching cubes approach to quad only surface reconstruction" [S1]. The method addresses topological consistency and parallel reconstruction issues in Dual Marching Cubes. It was developed to handle ambiguous configurations and enable efficient parallel simplification of volume data [S1].

## Sources
[S1] A parallel dual marching cubes approach to quad only surface reconstruction, https://doi.org/10.1007/s00371-021-02139-w, fetched 2026-10-26
