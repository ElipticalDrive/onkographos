# Parallel Processing and Pipeline Optimization


## Overview
This family covers parallel processing and pipeline optimization for Dual Marching Cubes surface extraction [S1]. It addresses topological consistency and parallel reconstruction by encoding neighborhood and connectivity information using a halfedge data structure [S1]. Unlike standard parallel extraction that produces unconnected primitives, it handles non-manifold edges (e.g., tunnels spanning two neighbor cells) and enables parallel mesh simplification [S1].

## Shared principles
The pipeline operates on a regular 3D grid and processes cells in parallel [S1]. It resolves topological inconsistencies by generating a valid halfedge data structure that correctly handles non-manifold edges, such as tunnels spanning two neighbor cells [S1]. Connectivity and neighborhood information are encoded to support parallel processing and simplification [S1]. The specific grid definitions, lookup tables, and two-pass counting and generation structure are detailed in the child articles [S1].

## Subtopics

### [Dual Marching Cubes (Counting Pass)](3766-dual-marching-cubes-counting-pass.md)
The first phase of the algorithm. It determines exact memory requirements for vertex and quad buffers. It iterates over a voxel grid, classifying cells and accumulating counts based on precomputed lookup tables. It is read-only regarding output buffers.

### [Parallel Dual Marching Cubes (DMC) Pipeline](6387-parallel-dual-marching-cubes-dmc-pipeline.md)
The complete surface extraction method. It generates a quadrilateral mesh with full connectivity information (1-ring neighbors). It combines static and dynamic lookup tables with parallel reduction algorithms. It builds on the Counting Pass but extends it for interactive performance on parallel hardware.

### Parallel Vertex and Face Coloring
#### definition
A scheme to encode neighborhood and connectivity information in the DMC mesh, enabling parallel processing and simplification [S1].
#### usage
Used to facilitate parallel simplification methods by identifying irregular elements based on face colors [S1].

## Background
The method originated in "A parallel dual marching cubes approach to quad only surface reconstruction" [S1]. It addresses topological consistency and parallel reconstruction issues in Dual Marching Cubes [S1]. The approach was developed to handle ambiguous configurations and enable efficient parallel simplification of volume data [S1].

## Sources
[S1] A parallel dual marching cubes approach to quad only surface reconstruction, https://doi.org/10.1007/s00371-021-02139-w, fetched 2026-10-26
