# Sparse Radiance Field (SRF) Representation


## Overview
A Sparse Radiance Field (SRF) is a discrete, voxel-based representation of a 3D scene or object that stores volume density and view-dependent radiance colors in a sparse grid structure [S1]. It serves as an efficient alternative to continuous Multi-Layer Perceptrons (MLPs) by encoding the radiance field function $f: (\mathbf{x}, \mathbf{d}) \rightarrow (\sigma, \mathbf{c})$ into a set of non-empty voxels, enabling fast optimization and direct mesh extraction via Marching Cubes [S1]. Unlike dense voxel grids, SRFs utilize a Compressed Sparse Order (COO) format to store only occupied regions, significantly reducing memory usage while maintaining high-fidelity rendering capabilities [S1].

## How it works
**Inputs and Outputs**
The SRF representation takes a 3D scene defined by posed images and outputs a sparse set of voxels. Each voxel is defined by its integer coordinates $(i, j, k)$ and a feature vector containing density and radiance information [S1].

**Data Structures**
The SRF is stored in COO (Coordinate) format, which consists of a list of $M$ tuples. Each tuple $m$ contains:
1.  **Coordinates** $c_m = (i, j, k)$: Positive integers representing the voxel index in the 3D grid.
2.  **Features** $f_m \in \mathbb{R}^{d+1}$: A vector of size $d+1$, where the first component is the density $\alpha_{i,j,k} \in \mathbb{R}$ and the remaining $d$ components are the radiance colors $\rho_{i,j,k} \in \mathbb{R}^d$ [S1].

The ordering of the $M$ tuples is arbitrary [S1].

**Parameters**
*   **Grid Resolution ($H$):** The side length of the cubic bounding box in voxels. The grid is of size $H \times H \times H$. Typical values used in the SPARF dataset are $H \in \{32, 128, 512\}$ [S1].
*   **Radiance Dimension ($d$):** The number of features per voxel for color representation.
    *   For view-dependent SRF using 4 spherical harmonics (SH) factors per RGB channel: $d = 3 \times 4 = 12$ [S1].
    *   For fixed RGB colors (no view dependence): $d = 3 \times 1 = 3$ [S1].
    *   Note: Using fewer SH components (e.g., 4 per channel instead of 9) reduces optimization space by 40% and time by 10% with negligible PSNR loss [S1]. Using raw RGB instead of SH reduces PSNR by ~1 dB but saves 80% space and 20% time [S1].

**Algorithm: SRF Construction and Rendering**
1.  **Initialization:** Define a bounding box of size $H \times H \times H$. Initialize an empty list for COO tuples.
2.  **Optimization:** Optimize the SRF parameters (densities and colors) to minimize the difference between rendered images and ground truth images. This is typically done using differentiable volume rendering [S1].
    *   *Note:* The specific optimization loop is handled by the parent method [Specific Model Architectures](8585-specific-model-architectures.md).
3.  **Sparsification:** After optimization, remove voxels with density below a threshold or those that do not contribute significantly to the rendered image. Store only the non-empty voxels in the COO format [S1].
4.  **Rendering:** To render a view from camera pose $(\mathbf{C}, \mathbf{D})$:
    *   Cast rays through the scene.
    *   For each ray, query the SRF for density $\alpha$ and color $\mathbf{c}$ at sampled points.
    *   Perform standard volume rendering integration:
        $$ C(\mathbf{r}) = \sum_{i=1}^{N} T_i \alpha_i \mathbf{c}_i $$
        where $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j)$ is the accumulated transmittance, $\sigma$ is derived from $\alpha$, and $\delta$ is the distance between samples [S1].

**Pitfalls and Edge Cases**
*   **Low-Density Noise:** Non-empty voxels may have low densities that do not affect volume rendering but contain random colors. This noise can confuse the network during training [S1].
*   **Vanishing Gradients:** High sparsity (typically ~99%) can lead to vanishing gradients. Misalignment between input and output SRF coordinates, caused by sparse convolutional strides and pruning, further harms gradient flow [S1].
*   **Precision Sensitivity:** Small errors in predicting densities or radiance colors can result in large distortions in rendered images, hurting novel view synthesis performance [S1].

## Operating characteristics
*   **Memory Complexity:** $O(M)$, where $M$ is the number of non-empty voxels. This is significantly more memory-efficient than dense grids $O(H^3)$ for sparse scenes [S1].
*   **Time Complexity:** Rendering time depends on the number of samples per ray and the number of non-empty voxels intersected. Optimization time is reduced compared to MLPs due to the discrete nature and sparsity [S1].
*   **Strengths:**
    *   Enables direct mesh extraction using Marching Cubes [S1].
    *   Supports multi-resolution representations ($H \in \{32, 128, 512\}$) [S1].
    *   Faster optimization and rendering than dense voxel grids or large MLPs [S1].
*   **Limitations:**
    *   Sensitive to sparsity-induced gradient vanishing [S1].
    *   Requires careful handling of low-density noise [S1].
    *   View-dependent color representation (SH) increases memory usage compared to fixed RGB [S1].

## Background
The Sparse Radiance Field (SRF) representation was introduced in the context of the SPARF (Sparse Radiance Field) dataset and SuRFNet architecture [S1]. It builds upon the Plenoxels method, which uses spherical harmonics for view-dependent radiance in voxel grids [S1]. The SPARF dataset contains one million SRFs collected from multiple 3D object classes at various resolutions, aiming to scale up learning of 3D representations from few input images [S1]. The representation allows for efficient large-scale learning and generalization to novel view synthesis tasks [S1].

## Sources
[S1] SPARF: Large-Scale Learning of 3D Sparse Radiance Fields from Few Input Images, https://doi.org/10.1109/iccvw60793.2023.00315, fetched 2026-10-26
