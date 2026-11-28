# Sparse Radiance Field (SRF) Representation


## Overview
A Sparse Radiance Field (SRF) is a voxel-based representation that encodes the continuous radiance field of a scene or object into a discrete, sparse 3D grid [S1]. It solves the problem of efficient storage and optimization for large-scale 3D scenes by replacing dense neural networks with a sparse set of active voxels [S1]. Unlike dense MLP-based representations, SRFs allow for direct mesh extraction via Marching Cubes and enable multi-resolution handling through hierarchical grid structures [S1].

## How it works
**Inputs and Outputs:**
The input to the SRF definition is a 3D scene or object. The output is a sparse data structure representing the volume density and view-dependent radiance colors.

**Data Structures:**
The SRF is defined as a voxel grid of dimension $1+d$, where $d$ is the dimension of the radiance colors and 1 represents the volume density [S1].
- **Grid Dimensions:** The grid has size $H$ in each of the three spatial dimensions ($X, Y, Z$). The total theoretical size is $H \times H \times H \times (1+d)$ [S1].
- **Sparse Format:** The grid is stored in Coordinate List (COO) format to handle sparsity. It consists of a set of $M$ tuples:
  $$
  X_{\text{non-empty}} = \{(c_m, f_m)\}_{m=1}^M
  $$
  where $c_m = (i, j, k)$ are positive integer coordinates of the voxel, and $f_m \in \mathbb{R}^{d+1}$ are the features stored at that voxel [S1].
- **Feature Layout:** Each feature vector $f_m$ contains:
  1.  Density $\alpha_{i,j,k} \in \mathbb{R}$
  2.  Radiance colors $\rho_{i,j,k} \in \mathbb{R}^d$ [S1]
- **Sparsity:** The sparsity is defined as $1 - \frac{M}{H^3}$, where $M$ is the number of non-empty voxels [S1]. Typical sparsity is around 99% [S1].

**Parameters:**
- **Resolution ($H$):** The grid size in each dimension. Typical values used in the SPARF dataset are $H \in \{32, 128, 512\}$ [S1].
- **Radiance Dimension ($d$):**
  - For view-dependent SRF using 4 spherical harmonics (SH) factors per RGB channel: $d = 3 \times 4 = 12$ [S1].
  - For fixed RGB colors: $d = 3 \times 1 = 3$ [S1].
  - Note: Using 4 SH factors reduces optimization space by 40% and time by 10% compared to 9 SH factors, while maintaining similar PSNR [S1]. Using raw RGB reduces PSNR by ~1 dB but saves 80% space and 20% time [S1].

**Rendering Integration:**
During rendering, rays are sampled at points $\mathbf{x}_i$. The SRF is queried at these points. If a point falls within a non-empty voxel $(i,j,k)$, the density $\alpha$ and radiance $\rho$ are retrieved. The pixel color $C(\mathbf{r})$ is computed via standard volumetric rendering:
$$
C(\mathbf{r}) = \sum_{i=1}^N T_i \alpha_i \mathbf{c}_i, \quad \text{where } \alpha_i = 1 - \exp(-\sigma_i \delta_i), \quad T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)
$$
where $\sigma_i$ corresponds to the density $\alpha$ from the SRF, $\mathbf{c}_i$ corresponds to the radiance $\rho$ from the SRF, and $\delta_i$ is the distance between samples [S1].

**Pitfalls and Edge Cases:**
1.  **Low-Density Noise:** Many non-empty voxels have low densities that do not significantly affect volume rendering but contain random color information. This noise can confuse the network during optimization [S1].
2.  **Vanishing Gradients:** High sparsity (~99%) and misalignment between input and output SRF coordinates (often caused by sparse convolutional strides and pruning) can lead to vanishing gradients, harming the learning process [S1].
3.  **Small Errors:** Small errors in predicting densities or radiance colors can result in large distortions in rendered images [S1].

## Operating characteristics
- **Memory:** Memory usage is proportional to the number of non-empty voxels $M$ rather than the total grid volume $H^3$. Using raw RGB ($d=3$) reduces space by 80% compared to SH ($d=12$) [S1].
- **Speed:** Optimization time is reduced by using fewer SH factors (4 vs 9) and fewer Plenoxels iterations (3x12K) [S1].
- **Quality:** Using raw RGB instead of SH factors reduces PSNR by ~1 dB [S1].
- **Mesh Extraction:** Meshes can be extracted directly from the SRF using a single pass of Marching Cubes, as the representation lies on a 3D voxel space [S1].

## Background
The Sparse Radiance Field (SRF) representation was introduced in the context of the SPARF (Sparse Radiance Field) dataset, which collected one million SRFs for 3D object tasks [S1]. The approach adapts Plenoxels [S1] to scale up optimization for large numbers of shapes. It is designed to support multi-resolution representations and efficient optimization for novel view synthesis and 3D generation tasks [S1].

## Sources
[S1] SPARF: Large-Scale Learning of 3D Sparse Radiance Fields from Few Input Images, https://doi.org/10.1109/iccvw60793.2023.00315, fetched 2026-10-26
