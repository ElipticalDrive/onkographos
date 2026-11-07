# Sparse Radiance Field (SRF) Representation


## Overview
A Sparse Radiance Field (SRF) is a voxel-based representation that encodes volumetric density and view-dependent radiance into a sparse 3D grid, serving as a discrete, efficient alternative to the continuous MLP-based NeRF representation [S1]. It sets itself apart from dense voxel methods by storing only non-empty voxels in a Coordinate Format (COO), and from continuous MLPs by enabling direct mesh extraction via Marching Cubes and faster optimization through sparse data structures [S1].

## How it works
**Inputs and Outputs:**
The SRF representation takes a 3D scene or object and outputs a sparse set of voxels, each containing density and color information. The primary input for construction is a set of posed images (typically 400+ views for high-quality dataset generation) [S1].

**Data Structures:**
The SRF is defined as a voxel grid of dimension $1+d$, where $d$ is the dimension of the radiance colors and 1 is for density [S1].
*   **Grid Size:** $H \times H \times H$, where $H$ is the resolution in each dimension [S1].
*   **Voxel Content:** Each voxel at coordinates $(i, j, k)$ contains:
    *   Density $\alpha_{i,j,k} \in \mathbb{R}$
    *   Radiance colors $\rho_{i,j,k} \in \mathbb{R}^d$
*   **Storage Format:** COO (Coordinate) format [S1]. The SRF is represented as a set of $M$ tuples:
    $$ X_{\text{non-empty}} = \{(c_m, f_m)\}_{m=1}^M $$
    where $c_m = (i, j, k) \in \mathbb{Z}^+$ are the positive integer coordinates of the $m$-th non-empty voxel, and $f_m \in \mathbb{R}^{d+1}$ is the feature vector containing the density and radiance colors [S1]. The ordering of tuples is arbitrary [S1].

**Parameters:**
*   **Resolution ($H$):** Typical values used in the SPARF dataset are $H \in \{32, 128, 512\}$ [S1].
*   **Radiance Dimension ($d$):**
    *   For view-dependent SRF using 4 spherical harmonics (SH) factors per RGB channel: $d = 3 \times 4 = 12$ [S1].
    *   For fixed RGB colors: $d = 3 \times 1 = 3$ [S1].
    *   Note: Using 4 SH factors reduces optimization space by 40% and time by 10% compared to 9 SH factors, while maintaining PSNR [S1]. Using raw RGB reduces PSNR by ~1 dB but saves 80% space and 20% time [S1].

**Algorithm for SRF Construction (from images):**
1.  **Initialization:** Initialize a dense grid of size $H \times H \times H$ with zeros or small random values for density and radiance [G].
2.  **Optimization:** Optimize the grid values to minimize the difference between rendered images and input images. This is typically done using differentiable volume rendering [S1].
    *   *Rendering:* For each camera ray, sample points along the ray. Interpolate density $\sigma$ and color $\mathbf{c}$ from the nearest voxels in the grid. Compute the accumulated color $C(r)$ using the standard volume rendering equation:
        $$ C(r) = \sum_{i=1}^{N} T_i \alpha_i \mathbf{c}_i, \quad T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right), \quad \alpha_i = 1 - \exp(-\sigma_i \delta_i) $$
    *   *Loss:* Minimize photometric loss (e.g., L1 or L2) between rendered and ground truth images [G].
3.  **Sparsification:** After optimization, prune voxels with density below a threshold to create the sparse COO representation [G].
4.  **Mesh Extraction (Optional):** Apply Marching Cubes on the density field $\alpha_{i,j,k}$ of the SRF to extract a 3D mesh [S1].

**Edge Cases and Pitfalls:**
*   **Low-Density Noise:** Non-empty voxels may have low densities that do not affect volume rendering but contain random colors, which can confuse the network during training [S1].
*   **Vanishing Gradients:** High sparsity (~99%) and misalignment between input and output SRF coordinates (due to sparse convolutional strides and pruning) can cause vanishing gradients [S1].
*   **Precision:** Small errors in predicting densities or radiance colors can result in large distortions in rendered images [S1].

## Operating characteristics
*   **Memory Complexity:** Proportional to the number of non-empty voxels $M$, not $H^3$. This makes it significantly more memory-efficient than dense voxel grids for sparse scenes [S1].
*   **Time Complexity:** Optimization time depends on the number of voxels and the number of training views. Using 4 SH factors reduces time by 10% compared to 9 SH factors [S1]. Running Plenoxels for fewer iterations (3x12K) reduces time by 30% [S1].
*   **Strengths:** Enables direct mesh extraction via Marching Cubes; efficient storage for sparse scenes; faster optimization than dense MLPs for large-scale datasets [S1].
*   **Limitations:** Sensitive to sparsity-induced vanishing gradients; low-density voxels can introduce noise; requires careful handling of coordinate misalignment [S1].
*   **When to use:** When mesh extraction is required, or when dealing with large-scale datasets where memory efficiency is critical [S1].
*   **When not to use:** For highly dense scenes where sparsity is low, or when view-dependent effects require high-order spherical harmonics (increasing $d$ significantly) [S1].

## Background
The Sparse Radiance Field (SRF) representation was introduced in the context of the SPARF (Sparse Radiance Field) dataset, which contains one million SRFs extracted from 3D objects [S1]. The concept builds upon Plenoxels, which proposed using spherical harmonics for view-dependent radiance in voxel grids [S1]. The SPARF dataset was created to facilitate large-scale learning of 3D sparse radiance fields from few input images, addressing the need for scalable 3D representations [S1]. The representation is particularly useful for tasks involving multiple resolutions and robust novel view synthesis [S1].

## Sources
[S1] SPARF: Large-Scale Learning of 3D Sparse Radiance Fields from Few Input Images, https://doi.org/10.1109/iccvw60793.2023.00315, fetched 2026-10-26
