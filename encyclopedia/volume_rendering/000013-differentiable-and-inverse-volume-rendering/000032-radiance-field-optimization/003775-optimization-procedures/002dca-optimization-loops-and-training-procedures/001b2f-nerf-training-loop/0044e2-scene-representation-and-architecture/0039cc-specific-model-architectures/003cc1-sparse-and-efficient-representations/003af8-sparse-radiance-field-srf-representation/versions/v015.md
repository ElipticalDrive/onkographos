# Sparse Radiance Field (SRF) Representation


## Overview
A Sparse Radiance Field (SRF) is a voxel-based representation of a continuous radiance field where only non-empty voxels are stored, enabling efficient optimization and rendering compared to dense grid methods [S1]. It sets itself apart from dense NeRFs by replacing the implicit MLP with an explicit, sparse 3D grid structure, and from other sparse methods like KiloNeRF by utilizing a direct COO (Coordinate) format storage for irregular volumetric data without hierarchical tree structures [S1, Parent].

## How it works
**Inputs and Outputs:**
*   **Input:** Posed 2D images of a scene or object.
*   **Output:** A set of non-empty voxels containing density and radiance features, represented as a Sparse Radiance Field.

**Data Structures:**
The SRF is defined as a voxel grid of dimension $1+d$, where $d$ is the dimension of the radiance colors and 1 dimension is for density [S1].
*   **Grid Size:** $H \times H \times H$, where $H$ is the resolution in each dimension [S1].
*   **Storage Format:** Compressed Coordinate (COO) format [S1].
*   **Structure:** A set of $M$ tuples $X_{non-empty} = \{(c_m, f_m)\}_{m=1}^M$ [S1].
    *   $c_m = (i, j, k) \in \mathbb{Z}^+$: The positive integer coordinates of the voxel.
    *   $f_m \in \mathbb{R}^{d+1}$: The feature vector for that voxel.
    *   The feature vector $f_m$ consists of the density $\alpha_{i,j,k}$ and the radiance colors $\rho_{i,j,k}$ [S1].
*   **Ordering:** The ordering of the tuples in the set is arbitrary [S1].

**Parameters:**
*   **Resolution ($H$):** Typical values used in SPARF are $H \in \{32, 128, 512\}$ [S1].
*   **Radiance Dimension ($d$):**
    *   For view-dependent SRF using 4 spherical harmonics factors of RGB channels: $d = 3 \times 4 = 12$ [S1].
    *   For fixed RGB colors: $d = 3 \times 1 = 3$ [S1].
    *   Note: Using RGB instead of SH reduces PSNR by ~1 dB, space by 80%, and time by 20% [S1].

**Rendering Integration:**
The SRF acts as an encoding of the NeRF MLP into sparse voxels [S1]. During rendering, rays are sampled. If a ray intersects a voxel in the SRF, the density $\alpha$ and radiance $\rho$ are retrieved. The pixel color $C(\mathbf{r})$ is computed via standard volumetric rendering:
$$
C(\mathbf{r}) = \sum_{i=1}^N T_i \alpha_i \mathbf{c}_i
$$
where $\alpha_i$ is the opacity derived from density, $\mathbf{c}_i$ is the color (radiance), and $T_i$ is the accumulated transmittance [Parent].

**Pitfalls and Edge Cases:**
*   **Low-Density Noise:** Many non-empty voxels have low densities that do not significantly affect volume rendering but contain random color information that can confuse the network during optimization [S1].
*   **Vanishing Gradients:** High sparsity (typically ~99%) and misalignment between input and output SRF coordinates (caused by sparse convolutional strides and pruning) can lead to vanishing gradients [S1].
*   **Sensitivity:** Small errors in predicting densities or radiance colors can result in large distortions in rendered images [S1].

## Operating characteristics
*   **Memory:** Memory usage is proportional to the number of non-empty voxels $M$ rather than the total grid size $H^3$, offering significant savings for sparse scenes [S1].
*   **Performance:** Using 4 spherical harmonics components reduces optimization space by 40% and time by 10% compared to higher orders, while maintaining PSNR [S1].
*   **Mesh Extraction:** 3D meshes can be extracted easily from SRFs using a single pass of Marching Cubes, as the data lies directly in 3D voxel space [S1].
*   **Limitations:** The irregular nature of the SRF means it does not necessarily reflect the underlying 3D shape/scene structure, as it results from optimizing posed images into volume [S1].

## Background
The Sparse Radiance Field representation was introduced in the context of the SPARF (Large-Scale Learning of 3D Sparse Radiance Fields) project [S1]. It builds on the concept of Plenoxels, adapting spherical harmonics for view-dependent learning into a sparse voxel grid [S1, S3]. The method was developed to scale up NeRF-like optimization to large datasets (e.g., one million SRFs) by reducing the number of iterations required while maintaining high average PSNR [S1].

## Sources
[S1] SPARF: Large-Scale Learning of 3D Sparse Radiance Fields from Few Input Images, https://doi.org/10.1109/iccvw60793.2023.00315, fetched 2026-10-26
