# Sparse Radiance Field (SRF) Representation


## Overview
A Sparse Radiance Field (SRF) is a voxel-based representation that encodes the continuous radiance field of a scene or object into a discrete, sparse 3D grid [S1]. It serves as an efficient alternative to dense neural implicit representations (like standard NeRF MLPs) by storing volume density and view-dependent radiance directly in voxel attributes, enabling fast rendering and easy mesh extraction via Marching Cubes [S1]. Unlike dense voxel grids, the SRF utilizes a Coordinate Format (COO) to store only non-empty voxels, significantly reducing memory usage for scenes with large empty spaces [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A set of posed 2D images of a scene/object.
*   **Output:** A Sparse Radiance Field $X_{non-empty}$, defined as a set of $M$ tuples containing 3D coordinates and feature vectors.

**Data Structures**
The SRF is stored in COO (Coordinate) format. The data structure consists of a list of $M$ tuples. Each tuple $m$ contains:
1.  **Coordinates** $c_m = (i, j, k)$: Positive integers representing the voxel indices in the 3D grid.
2.  **Features** $f_m \in \mathbb{R}^{d+1}$: A vector containing:
    *   Density $\alpha_{i,j,k} \in \mathbb{R}$
    *   Radiance colors $\rho_{i,j,k} \in \mathbb{R}^d$

**Parameters**
*   **Grid Resolution $H$:** The side length of the cubic bounding box in voxels. Typical values used in SPARF are $H \in \{32, 128, 512\}$ [S1].
*   **Radiance Dimension $d$:** The number of features per voxel for color representation [S1].
    *   For view-dependent SRF using 4 spherical harmonics (SH) factors per RGB channel: $d = 3 \times 4 = 12$ [S1].
    *   For fixed RGB colors: $d = 3 \times 1 = 3$ [S1].
    *   Note: Using 4 SH components reduces optimization space by 40% and time by 10% compared to 9 components, while maintaining similar PSNR [S1]. Using fixed RGB reduces PSNR by ~1 dB but saves 80% space and 20% time [S1].

**Formulas**
The SRF is defined as a subset of the full grid $X \in \mathbb{R}^{H \times H \times H \times (1+d)}$. The sparse representation is:
$$
X_{non-empty} = \{(c_m, f_m)\}_{m=1}^M
$$
where the sparsity is approximately $1 - \frac{M}{H^3}$ [S1].

Rendering follows the standard volume rendering integral [S1, S2, S3]:
$$
\mathbf{C}(\mathbf{r}) = \sum_{i=1}^{N} T_i \alpha_i \mathbf{c}_i, \quad \alpha_i = 1 - \exp(-\sigma_i \delta_i), \quad T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)
$$
In the SRF context, $\sigma_i$ is interpolated from the density $\alpha$ values of the surrounding voxels, and $\mathbf{c}_i$ is interpolated from the radiance features $\rho$ [S1].

**Algorithm: SRF Construction and Rendering**
1.  **Initialization:** Define grid resolution $H$ and radiance dimension $d$. Initialize an empty list `voxels`.
2.  **Optimization:** Optimize the SRF parameters (densities and radiance features) by minimizing the photometric loss between rendered images and ground truth images. This is typically done using differentiable volume rendering [S1].
    *   *Note:* The specific optimization loop details are covered in the parent article [Specific Model Architectures](8585-specific-model-architectures.md).
3.  **Storage:** After optimization, collect all voxels where the density $\alpha_{i,j,k}$ is non-zero (or above a threshold) into the COO list.
    *   For each voxel $(i,j,k)$ with $\alpha_{i,j,k} > 0$:
        *   Append tuple $(c_m=(i,j,k), f_m=[\alpha_{i,j,k}, \rho_{i,j,k}])$ to `voxels`.
4.  **Rendering a Ray:**
    *   Cast a ray $\mathbf{r}$ through the scene.
    *   Intersect the ray with the bounding box of the SRF.
    *   Sample points along the ray. For each sample point $\mathbf{x}$:
        *   Find the surrounding voxels in the COO structure.
        *   Trilinearly interpolate density $\sigma$ and radiance $\mathbf{c}$ from the features $f_m$ of the surrounding voxels.
        *   Compute transmittance $T$ and opacity $\alpha$ using the formulas above.
        *   Accumulate color $\mathbf{C}(\mathbf{r})$.

**Edge Cases and Pitfalls**
*   **Low-Density Noise:** Non-empty voxels may have very low densities that do not contribute significantly to rendering but contain random color information. This noise can confuse the network during training or downstream tasks [S1].
*   **Vanishing Gradients:** High sparsity (typically ~99%) and misalignment between input and output SRF coordinates (due to sparse convolutional strides and pruning) can cause vanishing gradients, harming the learning process [S1].
*   **Precision Errors:** Small errors in predicting densities or radiance colors can result in large distortions in rendered images [S1].

## Operating characteristics
*   **Memory Complexity:** $O(M)$, where $M$ is the number of non-empty voxels. This is significantly lower than the $O(H^3)$ complexity of dense voxel grids for sparse scenes [S1].
*   **Rendering Speed:** Faster than MLP-based NeRFs because it avoids per-sample neural network inference. Rendering speed depends on the number of samples and the efficiency of the sparse data structure lookup [S1].
*   **Mesh Extraction:** Straightforward and efficient. Since SRFs lie on a 3D voxel space, meshes can be extracted using a single pass of the Marching Cubes algorithm [S1].
*   **Strengths:** Efficient optimization, easy mesh extraction, compact storage for sparse scenes, compatibility with standard volume rendering pipelines [S1].
*   **Limitations:** Prone to vanishing gradients due to sparsity; sensitive to noise in low-density voxels; requires careful handling of coordinate misalignment [S1].

## Background
The Sparse Radiance Field (SRF) representation was introduced in the context of the SPARF (Sparse Radiance Field) dataset and SuRFNet architecture [S1]. The work was presented at the IEEE/CVF International Conference on Computer Vision (ICCV) Workshops in 2023 [S1]. The SRF builds upon the Plenoxels approach, which uses spherical harmonics for view-dependent radiance in voxel grids, adapting it for large-scale learning from few input images [S1]. The SPARF dataset contains one million SRFs across multiple resolutions (32, 128, 512) and 13 different object classes, aimed at facilitating large-scale learning of 3D representations [S1].

## Sources
[S1] SPARF: Large-Scale Learning of 3D Sparse Radiance Fields from Few Input Images, https://doi.org/10.1109/iccvw60793.2023.00315, fetched 2026-10-26
