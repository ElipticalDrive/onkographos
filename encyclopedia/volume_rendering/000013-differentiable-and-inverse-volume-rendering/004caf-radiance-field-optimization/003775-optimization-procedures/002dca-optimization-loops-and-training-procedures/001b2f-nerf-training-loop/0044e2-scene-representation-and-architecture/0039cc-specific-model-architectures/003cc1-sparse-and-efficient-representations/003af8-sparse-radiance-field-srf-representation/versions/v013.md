# Sparse Radiance Field (SRF) Representation


## Overview
A Sparse Radiance Field (SRF) is a voxel-based scene representation that encodes density and view-dependent radiance colors into a sparse 3D grid, serving as a discrete, memory-efficient alternative to global neural radiance fields (NeRFs) [S1]. Unlike global MLP-based representations, SRFs localize predictions to specific spatial regions, enabling direct mesh extraction via Marching Cubes and facilitating large-scale dataset generation for learning-based inverse rendering [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A set of posed images (and optionally depth) of a scene or object.
*   **Output:** A sparse set of voxels, each containing a density value and a vector of radiance color coefficients.

**Data Structures**
The SRF is defined as a voxel grid of dimension $1+d$, where $d$ is the dimension of the radiance color features and 1 represents the density [S1].
*   **Grid Dimensions:** The full grid size is $H \times H \times H$, where $H$ is the resolution in each dimension [S1].
*   **Voxel Content:** Each voxel at integer coordinates $(i, j, k)$ stores:
    *   Density: $\alpha_{i,j,k} \in \mathbb{R}$
    *   Radiance Colors: $\boldsymbol{\rho}_{i,j,k} \in \mathbb{R}^d$
*   **Sparse Storage Format:** The SRF is stored in Coordinate List (COO) format to handle sparsity [S1]. It is represented as a set of $M$ tuples:
    $$
    X_{\text{non-empty}} = \{(c_m, f_m)\}_{m=1}^M
    $$
    where $c_m = (i, j, k) \in \mathbb{Z}^+$ are the positive integer coordinates of the $m$-th non-empty voxel, and $f_m \in \mathbb{R}^{d+1}$ is the feature vector containing the density and radiance colors [S1]. The ordering of tuples is arbitrary [S1].

**Parameters**
*   **Resolution ($H$):** The grid size in each dimension. Typical values used in the SPARF dataset are $H \in \{32, 128, 512\}$ [S1].
*   **Radiance Dimension ($d$):** The number of features per voxel for color representation [S1].
    *   For view-dependent SRF using 4 spherical harmonics (SH) factors per RGB channel: $d = 3 \times 4 = 12$ [S1].
    *   For fixed RGB colors (no view dependence): $d = 3 \times 1 = 3$ [S1].
    *   Note: Using fewer SH components (e.g., 4 per channel vs 9) reduces optimization space by 40% and time by 10% with negligible PSNR loss [S1]. Using raw RGB reduces PSNR by ~1 dB but saves 80% space and 20% time [S1].

**Rendering Process**
To render a pixel, a ray is cast through the scene. The ray is sampled at discrete points. For each sample point $\mathbf{x}$, the algorithm:
1.  Identifies the voxel containing $\mathbf{x}$.
2.  Retrieves the density $\sigma$ and radiance $\mathbf{c}$ from the voxel's feature vector $f_m$ [S1].
3.  Composites the results using the standard volume rendering equation (as defined in the parent article [Sparse and Efficient Representations](9326-sparse-and-efficient-representations.md)).

**Mesh Extraction**
Because SRFs reside in a 3D voxel space, 3D meshes can be extracted directly using a single pass of the Marching Cubes algorithm on the density field $\alpha_{i,j,k}$ [S1].

**Pitfalls and Edge Cases**
*   **Low-Density Noise:** Many non-empty voxels may have low densities that do not significantly affect volume rendering but contain random color information. This noise can confuse learning networks if not handled [S1].
*   **Vanishing Gradients:** High sparsity (typically ~99%) and misalignment between input and output SRF coordinates (caused by sparse convolutional strides and pruning) can lead to vanishing gradients during training [S1].
*   **Precision Sensitivity:** Small errors in predicting densities or radiance colors can result in large distortions in rendered images, negatively impacting novel view synthesis performance [S1].

## Operating characteristics
*   **Memory Complexity:** Memory usage is proportional to the number of non-empty voxels $M$ rather than the total grid size $H^3$. The COO format stores only active voxels, offering significant savings for sparse scenes [S1].
*   **Time Complexity:** Rendering time depends on the number of samples per ray and the lookup cost in the sparse structure. Training time is reduced compared to global NeRFs when using fewer SH components or raw RGB, though large-scale optimization requires careful iteration management [S1].
*   **Strengths:** Enables direct mesh extraction via Marching Cubes; supports multi-resolution representations ($H \in \{32, 128, 512\}$); allows for efficient large-scale dataset generation (e.g., SPARF with 1 million shapes) [S1].
*   **Limitations:** Sensitive to sparsity-induced gradient vanishing; prone to color noise in low-density regions; requires careful tuning of SH components to balance quality and performance [S1].
*   **When to use:** Suitable for applications requiring explicit mesh extraction, large-scale learning of 3D shapes from few images, or when memory constraints prohibit global dense voxel grids [S1].

## Background
The Sparse Radiance Field (SRF) representation was introduced in the context of the SPARF (Sparse Radiance Fields) project, which aimed to create a large-scale dataset of 3D sparse radiance fields for learning-based 3D vision tasks [S1]. The work builds upon Plenoxels, adapting its spherical harmonics encoding to a sparse voxel structure to scale optimization to millions of shapes [S1]. The SPARF dataset includes one million SRFs across 13 classes, collected using adaptations of Plenoxels optimization to minimize iterations while maintaining high PSNR [S1].

## Sources
[S1] SPARF: Large-Scale Learning of 3D Sparse Radiance Fields from Few Input Images, https://doi.org/10.1109/iccvw60793.2023.00315, fetched 2026-10-26
