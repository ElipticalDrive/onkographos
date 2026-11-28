# Sparse Radiance Field (SRF) Representation


## Overview
A Sparse Radiance Field (SRF) is a voxel-based representation of a 3D scene or object that stores density and view-dependent radiance colors in a sparse grid structure [S1]. It serves as an efficient alternative to dense neural radiance fields (NeRFs) by mapping the continuous MLP-based representation into discrete, sparse voxels, enabling faster optimization and rendering while maintaining high visual fidelity [S1]. Unlike its siblings, SRF explicitly leverages the Coordinate Format (COO) to handle extreme sparsity (~99%), making it suitable for large-scale datasets and multi-resolution applications [S1].

## How it works
**Inputs:** A set of posed 2D images of a scene or object.
**Outputs:** A sparse set of voxel tuples containing density and radiance features.

**Data Structures:**
The SRF is defined as a voxel grid of dimension $1+d$, where $d$ is the dimension of the radiance colors and 1 dimension is for density [S1].
*   **Grid Size:** $H \times H \times H$ in each of the three spatial dimensions [S1].
*   **Feature Vector:** Each voxel $(i, j, k)$ stores a feature vector $f \in \mathbb{R}^{1+d}$ [S1].
    *   The first component is the density $\alpha_{i,j,k} \in \mathbb{R}$ [S1].
    *   The remaining $d$ components are the radiance colors $\rho_{i,j,k} \in \mathbb{R}^d$ [S1].
*   **Storage Format:** The SRF is stored in Coordinate Format (COO) as a set of $M$ tuples [S1]:
    $$X_{\text{non-empty}} = \{(c_m, f_m)\}_{m=1}^M$$
    where $c_m = (i, j, k)$ are positive integer coordinates ($c \in \mathbb{Z}^+$) and $f_m$ is the feature vector at that location [S1]. The ordering of tuples is arbitrary [S1].

**Parameters:**
*   **Resolution ($H$):** The grid size in each dimension. Typical values used in SPARF are $H \in \{32, 128, 512\}$ [S1].
*   **Radiance Dimension ($d$):**
    *   For view-dependent SRF using 4 spherical harmonics (SH) factors per RGB channel: $d = 3 \times 4 = 12$ [S1].
    *   For fixed RGB colors: $d = 3 \times 1 = 3$ [S1].
    *   Note: Using 4 SH components reduces optimization space by 40% and time by 10% compared to 9 SH components, while maintaining PSNR [S1]. Using raw RGB reduces PSNR by ~1 dB but saves 80% space and 20% time [S1].

**Algorithm Steps:**
1.  **Initialization:** Initialize a sparse grid of size $H \times H \times H$.
2.  **Optimization:** Optimize the SRF parameters (density $\alpha$ and radiance $\rho$) using posed images. This involves minimizing the difference between rendered images and ground truth images [S1].
    *   *Implementation Note:* The optimization typically uses an adaptation of Plenoxels [S1]. Running Plenoxels for fewer iterations (e.g., $3 \times 12$k) reduces time by 30% with negligible PSNR loss [S1].
3.  **Sparsity Handling:** The resulting grid is highly sparse (~99% empty). Only non-empty voxels are stored in the COO format [S1].
4.  **Rendering:** To render a view, cast rays through the sparse grid. For each ray, sample the density and radiance from the stored COO tuples. Compute the volume rendering integral using standard differentiable volume rendering equations [S1].
5.  **Mesh Extraction:** Since SRFs lie on a 3D voxel space, meshes can be extracted directly using a single pass of Marching Cubes on the density field $\alpha$ [S1].

**Edge Cases and Pitfalls:**
*   **Low-Density Voxels:** Non-empty voxels may have low densities that do not significantly affect volume rendering but contain random colors. These can confuse the network during training [S1].
*   **Vanishing Gradients:** High sparsity (~99%) and misalignment between input and output SRF coordinates (caused by sparse convolutional strides and pruning) can lead to vanishing gradients [S1].
*   **Precision:** Small errors in predicting densities or radiance colors can result in large distortions in rendered images [S1].

## Operating characteristics
*   **Memory Complexity:** Scales with the number of non-empty voxels $M$ rather than $H^3$. The COO format stores only $M$ tuples, each containing coordinates and $1+d$ features [S1].
*   **Time Complexity:** Rendering and optimization are faster than dense NeRFs due to the reduced number of active voxels. Using 4 SH components reduces optimization time by 10% compared to 9 SH components [S1]. Reducing Plenoxels iterations reduces time by 30% [S1].
*   **Performance:** SPARF achieves high average PSNR across one million shapes at resolutions 32, 128, and 512 [S1].
*   **Strengths:** Efficient optimization, easy mesh extraction, scalable to large datasets, supports multi-resolution [S1].
*   **Limitations:** Sensitive to sparsity-induced gradient issues, potential for noisy color information in low-density voxels [S1].
*   **When to use:** Large-scale 3D object reconstruction, multi-resolution applications, scenarios requiring fast mesh extraction [S1].
*   **When not to use:** Scenes with very complex, continuous geometry where sparse voxelization loses detail, or when strict memory limits prevent storing even sparse COO structures for very large $H$ [S1].

## Background
The Sparse Radiance Field (SRF) was introduced in the context of the SPARF dataset, which contains one million 3D sparse radiance fields [S1]. The concept builds on Plenoxels, which introduced view-dependent radiance using spherical harmonics in a voxel grid [S1]. SPARF adapted Plenoxels to scale up to a million shapes, utilizing multiple resolutions (32, 128, 512) and specialized losses to handle challenges like vanishing gradients and noisy low-density voxels [S1]. The representation allows for straightforward mesh extraction via Marching Cubes [S1].

## Sources
[S1] SPARF: Large-Scale Learning of 3D Sparse Radiance Fields from Few Input Images, https://doi.org/10.1109/iccvw60793.2023.00315, fetched 2026-10-26
