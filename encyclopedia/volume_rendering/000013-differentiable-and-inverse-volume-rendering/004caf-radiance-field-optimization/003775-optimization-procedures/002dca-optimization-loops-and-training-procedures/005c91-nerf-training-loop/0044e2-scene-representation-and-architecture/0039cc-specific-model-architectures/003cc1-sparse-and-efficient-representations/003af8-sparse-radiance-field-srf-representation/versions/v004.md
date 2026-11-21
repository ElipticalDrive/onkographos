# Sparse Radiance Field (SRF) Representation


## Overview
A Sparse Radiance Field (SRF) is a voxel-based representation that encodes the volume density and view-dependent radiance of a 3D scene or object into a sparse grid [S1]. It serves as a structured alternative to the continuous Multi-Layer Perceptron (MLP) used in standard NeRF, enabling efficient optimization and direct mesh extraction via algorithms like Marching Cubes [S1]. Unlike its sibling [Mega-NeRF Spatial Partitioning](8776-mega-nerf-spatial-partitioning.md), which manages independent sub-networks for spatial cells, the SRF stores explicit feature vectors directly in the voxel coordinates, treating the radiance field as a discrete, irregular volumetric data structure [S1].

## How it works

### Inputs and Outputs
*   **Input:** A set of posed 2D images of a scene or object.
*   **Output:** A Sparse Radiance Field $X$ defined by a set of non-empty voxels, each containing density and radiance features.

### Data Structures
The SRF is defined as a voxel grid of dimension $1+d$ [S1].
*   **Grid Dimensions:** The grid has size $H$ in each of the three spatial dimensions ($X, Y, Z$). The total potential size is $H \times H \times H \times (1+d)$ [S1].
*   **Feature Vector:** Each voxel $(i, j, k)$ stores a feature vector $f \in \mathbb{R}^{d+1}$ [S1].
    *   The first component is the volume density $\alpha_{i,j,k} \in \mathbb{R}$ [S1].
    *   The remaining $d$ components are the radiance colors $\rho_{i,j,k} \in \mathbb{R}^d$ [S1].
*   **Sparse Representation:** Due to high sparsity, the SRF is stored in Coordinate List (COO) format [S1]. It is represented as a set of $M$ tuples:
    $$X_{\text{non-empty}} = \{(c_m, f_m)\}_{m=1}^M$$
    where $c_m = (i, j, k)$ are positive integer coordinates ($c \in \mathbb{Z}^+$) and $f_m$ is the feature vector at that location [S1]. The ordering of tuples is arbitrary [S1].

### Parameters
*   **Resolution ($H$):** The grid size per dimension. Typical values used in the SPARF dataset are $H \in \{32, 128, 512\}$ [S1].
*   **Radiance Dimension ($d$):** The number of features for color [S1].
    *   For view-dependent SRF using 4 spherical harmonics factors per RGB channel: $d = 3 \times 4 = 12$ [S1].
    *   For fixed RGB colors: $d = 3 \times 1 = 3$ [S1].
    *   Note: Using RGB instead of Spherical Harmonics reduces PSNR by $\sim 1$ dB, space by 80%, and time by 20% [S1].

### Algorithm: SRF Construction and Rendering
The SRF is typically obtained by optimizing posed images into a volume, often using an adaptation of Plenoxels [S1].

1.  **Initialize Grid:** Create a sparse COO structure for a grid of size $H \times H \times H$.
2.  **Optimize Features:** For each non-empty voxel, optimize the density $\alpha$ and radiance $\rho$ to minimize the difference between rendered images and ground truth images [S1].
    *   *Note:* The specific optimization loop details are covered in the parent article [Specific Model Architectures](8585-specific-model-architectures.md).
3.  **Volume Rendering:** To render a view, cast rays through the sparse grid. For each ray, sample only the non-empty voxels $(c_m, f_m)$ intersected by the ray.
    *   Compute transmittance and color accumulation using standard volume rendering integrals, utilizing $\alpha_{i,j,k}$ for opacity and $\rho_{i,j,k}$ for color [S1].
4.  **Mesh Extraction (Optional):** Since SRFs lie on 3D voxel space, meshes can be extracted by applying Marching Cubes to the density field $\alpha_{i,j,k}$ [S1].

### Edge Cases and Pitfalls
*   **Low-Density Noise:** Many non-empty voxels have low densities that do not affect volume rendering but contain random colors. These can confuse the network during training [S1].
*   **Vanishing Gradients:** The typical sparsity is $\sim 99\%$. Combined with misalignment between input and output SRF coordinates (caused by sparse convolutional strides and pruning), this leads to vanishing gradients [S1].
*   **Precision Sensitivity:** Small errors in predicting densities or radiance colors can result in large distortions in rendered images [S1].

## Operating characteristics

*   **Memory Complexity:** Memory usage is proportional to the number of non-empty voxels $M$, not the total grid size $H^3$. This makes it highly efficient for scenes with significant empty space.
*   **Rendering Speed:** Rendering is faster than dense voxel grids because only $M$ voxels are sampled per ray. However, it may be slower than continuous MLPs for very sparse scenes due to the overhead of coordinate lookups, though it allows for parallelization on GPUs [S1].
*   **Mesh Extraction:** A key advantage is the ability to extract explicit 3D meshes directly using Marching Cubes, which is not natively supported by continuous NeRF MLPs [S1].
*   **Limitations:** The representation is sensitive to sparsity-induced gradient issues. It requires careful handling of low-density voxels to prevent color artifacts [S1].

## Background
The Sparse Radiance Field (SRF) was introduced in the context of the SPARF (Large-Scale Learning of 3D Sparse Radiance Fields from Few Input Images) project [S1]. The work aimed to scale up NeRF-like optimization to a large dataset of one million 3D shapes across multiple resolutions (32, 128, 512) [S1]. The SRF representation was chosen to facilitate efficient optimization and direct mesh extraction, leveraging the structured nature of voxel grids while maintaining sparsity for memory efficiency [S1].

## Sources
[S1] SPARF: Large-Scale Learning of 3D Sparse Radiance Fields from Few Input Images, https://doi.org/10.1109/iccvw60793.2023.00315, fetched 2026-10-26
