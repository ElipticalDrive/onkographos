# Sparse Radiance Field (SRF) Representation


## Overview
A Sparse Radiance Field (SRF) is a voxel-based representation of a 3D scene or object that explicitly stores volume density and view-dependent radiance colors in a sparse grid structure [S1]. It serves as a discrete, memory-efficient alternative to dense neural radiance fields (NeRFs) by encoding the continuous function $f_\theta(\mathbf{x}, \mathbf{d})$ into a set of non-empty voxels, enabling efficient optimization and direct mesh extraction via algorithms like Marching Cubes [S1]. Unlike dense grid methods, SRFs address the computational bottleneck of large-scale training by storing only occupied spatial regions, though they introduce challenges related to gradient vanishing and noise in low-density voxels [S1].

## How it works
**Inputs and Outputs**
The SRF representation takes a 3D scene defined by posed images as input and outputs a sparse set of voxel features. The primary output is the radiance field function, which, given a query point $\mathbf{x}$ and direction $\mathbf{d}$, returns density $\sigma$ and color $\mathbf{c}$. In the SRF context, these values are retrieved directly from the voxel grid rather than computed by a neural network at query time.

**Data Structures**
The SRF is defined as a voxel grid of dimension $1+d$, where $d$ is the dimension of the radiance color features and 1 dimension is reserved for density [S1].
*   **Grid Dimensions**: The full grid size is $H \times H \times H$, where $H$ is the resolution in each dimension [S1].
*   **Feature Vector**: Each voxel $(i, j, k)$ stores a feature vector $f \in \mathbb{R}^{1+d}$. The first component is the density $\alpha_{i,j,k} \in \mathbb{R}$, and the remaining $d$ components are the radiance colors $\rho_{i,j,k} \in \mathbb{R}^d$ [S1].
*   **Sparse Format**: The grid is stored in Coordinate Format (COO) as a set of $M$ tuples:
    $$
    X_{\text{non-empty}} = \{(c_m, f_m)\}_{m=1}^M
    $$
    where $c_m = (i, j, k)$ are positive integer coordinates ($c \in \mathbb{Z}^+$) and $f_m$ is the corresponding feature vector [S1]. The ordering of tuples is arbitrary [S1].
*   **Sparsity**: The sparsity is defined as $1 - \frac{M}{H^3}$ [S1]. Typical sparsity levels are approximately 99% [S1].

**Parameters**
*   **Resolution ($H$)**: The grid size in each dimension. Typical values used in the SPARF dataset are $H \in \{32, 128, 512\}$ [S1].
*   **Radiance Dimension ($d$)**:
    *   For view-dependent SRFs using 4 spherical harmonics (SH) factors per RGB channel: $d = 3 \times 4 = 12$ [S1].
    *   For fixed RGB colors: $d = 3 \times 1 = 3$ [S1].
    *   Using 4 SH factors reduces optimization space by 40% and time by 10% compared to 9 SH factors, while maintaining PSNR [S1]. Using fixed RGB reduces PSNR by ~1 dB but saves 80% space and 20% time [S1].

**Algorithmic Steps**
1.  **Initialization**: Define a 3D grid of size $H \times H \times H \times (1+d)$. Initialize all voxels as empty.
2.  **Optimization/Training**: Optimize the SRF from posed images. This typically involves minimizing a rendering loss (comparing rendered images to ground truth) and specialized losses such as a density loss (binary cross-entropy to create dense surfaces) [S1].
3.  **Sparsification**: During or after optimization, prune voxels that do not contribute significantly to the rendering. Only voxels with non-negligible density or color information are retained in the COO list [S1].
4.  **Rendering**: To render a ray, sample points along the ray. For each point, check if it falls within a non-empty voxel. If yes, retrieve $\alpha$ and $\rho$ from the COO structure. If no, assume zero density (transparent) [S1].
5.  **Volume Rendering**: Compute the pixel color $\mathbf{C}(\mathbf{r})$ using the standard volume rendering integral:
    $$
    \mathbf{C}(\mathbf{r}) = \sum_{i=1}^{N} T_i \alpha_i \mathbf{c}_i
    $$
    where $\alpha_i = 1 - \exp(-\sigma_i \delta_i)$ and $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$ [S1]. Note that in SRF, $\sigma_i$ is directly $\alpha_i$ from the voxel, and $\mathbf{c}_i$ is derived from $\rho_i$ and the view direction $\mathbf{d}$ using spherical harmonics if $d=12$ [S1].
6.  **Mesh Extraction**: To extract a mesh, apply the Marching Cubes algorithm to the density field $\alpha_{i,j,k}$ of the SRF [S1].

**Pitfalls and Edge Cases**
*   **Low-Density Noise**: Non-empty voxels may have low densities that do not affect volume rendering but contain random colors. This noise can confuse the network during training [S1].
*   **Vanishing Gradients**: High sparsity (~99%) and misalignment between input and output SRF coordinates (caused by sparse convolutional strides and pruning) can lead to vanishing gradients, harming the learning process [S1].
*   **Precision**: Small errors in predicting densities or radiance colors can result in large distortions in rendered images [S1].

## Operating characteristics
*   **Memory Complexity**: Memory usage is proportional to the number of non-empty voxels $M$, rather than the total grid size $H^3$. This allows for high-resolution representations ($H=512$) to be stored efficiently if the scene is sparse [S1].
*   **Time Complexity**: Rendering is fast as it involves direct lookup in the COO structure rather than neural network inference. Training time is reduced by using fewer spherical harmonics components and fewer optimization iterations compared to dense NeRFs [S1].
*   **Strengths**: Enables direct mesh extraction via Marching Cubes; efficient storage for large-scale datasets; faster optimization than dense MLP-based NeRFs when using appropriate resolutions and SH factors [S1].
*   **Limitations**: Struggles with high sparsity leading to gradient issues; sensitive to noise in low-density voxels; requires careful tuning of resolution $H$ and sparsity thresholds [S1].
*   **When to Use**: Suitable for large-scale 3D object datasets (e.g., ShapeNet) where multiple resolutions are needed and mesh extraction is desired [S1].
*   **When Not to Use**: Not ideal for highly complex, dense scenes where sparsity is low, or when real-time view-dependent rendering with high-frequency details is required without sufficient SH factors [S1].

## Background
The Sparse Radiance Field (SRF) representation was introduced in the context of the SPARF (Sparse Radiance Fields) dataset and framework [S1]. It builds upon the concept of Plenoxels, which proposed using spherical harmonics for view-dependent radiance in voxel grids [S1, S14]. The SPARF project collected one million SRFs from multiple resolutions (32, 128, 512) to facilitate large-scale learning of 3D representations from few input images [S1]. The representation addresses the scalability issues of NeRFs by discretizing the radiance field into sparse voxels, allowing for efficient optimization and dataset creation [S1].

## Sources
[S1] SPARF: Large-Scale Learning of 3D Sparse Radiance Fields from Few Input Images, https://doi.org/10.1109/iccvw60793.2023.00315, fetched 2026-10-26
