# Voxel and Sparse Grid Representations

Also called: Grid-based Radiance Fields, Hash-Encoded NeRFs

## Overview
Voxel and Sparse Grid Representations replace the continuous Multi-Layer Perceptron (MLP) function approximation of standard NeRF with discrete, structured data structures (voxel grids or hash tables) to store scene attributes like density and color [S1]. This substitution trades the high-capacity, memory-heavy neural computation for direct interpolation or hashing, enabling significantly faster rendering and training speeds [S1]. It differs from sibling methods that modify MLP inputs or outputs by fundamentally altering the underlying scene representation from a parametric function to a spatially indexed structure.

## Shared principles
The core mechanism involves mapping a 3D query position $\mathbf{x}$ to a set of learnable features using a discrete spatial structure, rather than passing $\mathbf{x}$ through a neural network.

1.  **Spatial Indexing:** The 3D space is discretized into a grid (dense or sparse) or hashed into a table.
2.  **Feature Storage:** Learnable parameters (scalars, vectors, or coefficient arrays) are stored at grid vertices, cells, or hash table entries.
3.  **Interpolation/Retrieval:** For a query point $\mathbf{x}$, the features are retrieved via trilinear interpolation (for dense/sparse voxels) or hash lookup (for sparse grids).
4.  **Decoupled Prediction:** The retrieved features are typically passed to a small, shared MLP or directly combined with view directions to predict density $\sigma$ and color $\mathbf{c}$.

## Subtopics

### [Voxel-based Radiance Fields](11470-voxel-based-radiance-fields.md)
Replaces the MLP with two dense 3D voxel grids: a density grid $G_\sigma$ and a color grid $G_{sh}$ storing degree-2 spherical harmonic (SH) coefficients. It enables real-time rendering by eliminating heavy feed-forward neural computations for density and color prediction, relying instead on trilinear interpolation.

### [Neural Multi-View Image Reconstruction with Hash Encoding](13013-neural-multi-view-image-reconstruction-with-hash-encoding.md)
Combines a hash-encoded implicit surface representation with a neural radiance field. It uses a multi-scale hash grid for geometry (SDF) and a separate light field network for color, enforcing explicit geometric constraints via SDFs rather than volume density. It differs from standard voxel methods by using hash tables for memory efficiency and SDFs for geometry, allowing faster training and lower memory usage while preserving fine details in textureless areas.

### Neural Sparse Voxel Fields (NSVF)
A voxel-based NeRF model that models the scene as a set of radiance fields bounded by voxels [S1]. It uses sparse voxel intersection-based point sampling for rays, which is more efficient than dense sampling or hierarchical two-step approaches, but is more memory-intensive due to storing feature vectors on a potentially dense voxel grid [S1]. Learnable features are stored at voxel vertices and interpolated, then processed by a shared MLP to compute volumetric density ($\sigma$) and color ($\mathbf{c}$) [S1].

## Background
The shift from MLP-based to grid-based representations emerged to address the computational bottlenecks of standard NeRF. Neural Sparse Voxel Fields (NSVF) was introduced in July 2020 by Liu et al. [S1]. It was developed as a voxel-based NeRF model that used sparse voxel intersection-based point sampling, which was much more efficient than the dense sampling or hierarchical two-step approach of Mildenhall et al. [S1]. However, it was noted to be more memory-intensive due to storing feature vectors on a potentially dense voxel grid [S1]. Subsequent work, such as the hash-encoding method described in [S1], further optimized memory usage and training speed by combining hash-encoded implicit surfaces with neural radiance fields, achieving lower memory consumption (5 GB on RTX 3090) compared to baselines like IDR (6.5 GB) and NeuS (7 GB) [S1].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
