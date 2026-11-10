# Sparse spatial data structures for NeRF

Also called: Octree-based NeRF, Sparse Parametric Encodings (Octree/Sparse Grid), Sparse voxel grids, VDB-based NeRF

## Overview
This family of methods uses sparse data structures (sparse grids, octrees, VDBs) to localize the surface interface and concentrate samples only in relevant regions, effectively skipping empty space to reduce MLP evaluations [S1]. By refining the sampling strategy to focus solely on the localized surface, these methods significantly reduce the number of required MLP evaluations, thereby improving computational efficiency [S1]. Sparse voxel grids allow skipping empty regions during ray tracing, leading to rendering speedups of up to 10 times compared to standard NeRF [S1].

## Shared principles
The core challenge in applying sparse structures to NeRF is that surfaces emerge during training, so space-partitioning data structures cannot be precomputed in advance and must be periodically updated [S1, S2]. To address this, methods typically adopt a multistage, coarse-to-fine strategy where regions are progressively refined and pruned as necessary [S1, S2].

The general workflow involves:
1.  **Coarse Approximation:** Learn implicit functions over an initial voxel grid to produce a coarse approximation of scene geometry [S1].
2.  **Pruning:** Progressively prune empty voxels during training based on this initial geometry signal [S1].
3.  **Refinement:** Use hierarchical structures (like octrees) or sparse grids to cull unused features from dense grids, improving efficiency when the surface of interest is known or can be progressively refined [S2].

## Subtopics

### [VDB-based Occupancy Grid Conversion](11675-vdb-based-occupancy-grid-conversion.md)
Replaces the dense binary grid of an Occupancy Grid with a VDB grid to reduce spatial redundancy, accelerating ray-sampling without re-training. It sets itself apart by fully exploiting VDB’s hierarchical structure for ray-traversal using HDDA to skip merged empty regions.

## Background
The observation that only around 10 to 20% of evaluated samples have non-zero density and contribute to the final image motivated acceleration methods that localize the surface interface [S1]. Early approaches like NSVF [LGZL∗ 20] and Efficient-NeRF [HLC∗ 22] introduced multistage coarse-to-fine strategies using sparse grids or two-level octrees (NerfTree) to dynamically adapt voxelization to the underlying scene structure [S1, S2]. Later works incorporated VDB structures [YLMM23] to further optimize traversal and memory usage [S1].

## Sources
[S1] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S2] Instant Neural Graphics Primitives with a Multiresolution ..., https://www.cs.jhu.edu/~misha/ReadingSeminar/Papers/Mueller22.pdf, fetched 2026-10-28
