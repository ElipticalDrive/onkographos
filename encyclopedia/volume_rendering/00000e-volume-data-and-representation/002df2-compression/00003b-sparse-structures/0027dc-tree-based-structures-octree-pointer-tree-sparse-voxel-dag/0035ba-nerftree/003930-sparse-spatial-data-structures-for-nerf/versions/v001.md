# Sparse spatial data structures for NeRF
Also called: Octree-based NeRF, Sparse voxel grids, VDB-based NeRF

## Overview
This family encompasses methods that use sparse data structures (sparse grids, octrees, VDBs) to localize the surface interface and concentrate samples only in relevant regions, effectively skipping empty space to reduce MLP evaluations [S1]. Unlike standard NeRF which evaluates the network along the entire ray, these methods dynamically adapt the voxelization to the underlying scene structure, accelerating surface localization [S1].

## Shared principles
The core mechanism involves learning implicit functions over an initial voxel grid to produce a coarse approximation of scene geometry [S1]. Because surfaces typically emerge during training, space-partitioning data structures cannot be precomputed in advance and must be periodically updated [S1].

The general workflow follows a multistage, coarse-to-fine strategy where regions are progressively refined and pruned as necessary [S1]. Empty voxels are pruned during training based on the initial geometry signal [S1]. This ensures that neural fields are only evaluated in non-empty regions, allowing the renderer to skip empty regions during ray tracing [S1].

## Subtopics

### TNSVF and Sparse Grids
Adopt a multistage, coarse-to-fine strategy using sparse grids to progressively refine and prune regions [S1].

### Efficient-NeRF (NerfTree)
Builds a two-level octree composed of a coarse density voxel volume for approximate geometry caching and fine sparse voxel grids localized around the surface for refinement [S1]. See [NerfTree](7559-nerftree.md).

### VDB-based Methods
Use Voxel-Based Data (VDB) structures to localize the surface and eliminate empty voxel cells [S1].

## Background
Standard NeRF evaluates samples along the entire ray, but only around 10 to 20% of evaluated samples have non-zero density and actually contribute to the final image [S1]. To address this, acceleration methods aim to localize the surface interface [S1]. Existing approaches generally fall into four categories: using sparse data structures, leveraging depth oracles, guiding sampling via Signed Distance Fields or density estimates, and employing learned sampling networks [S1].

Sparse data structures such as sparse grids, octrees, and VDBs can be used to localize the surface and eliminate empty voxel cells [S1]. The main challenge is that in a neural field setting, surfaces typically emerge during training, preventing precomputation of space-partitioning data structures [S1]. To address this, methods learn implicit functions over an initial voxel grid, subdividing the 3D space [S1]. This produces a coarse approximation of the scene geometry, which is then used to progressively prune empty voxels during training [S1].

The use of a sparse voxel grid allows the renderer to skip empty regions during tracing a ray, leading to rendering speedups of up to 10 times compared to the standard NeRF implementation [S1].

## Sources
[S1] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
