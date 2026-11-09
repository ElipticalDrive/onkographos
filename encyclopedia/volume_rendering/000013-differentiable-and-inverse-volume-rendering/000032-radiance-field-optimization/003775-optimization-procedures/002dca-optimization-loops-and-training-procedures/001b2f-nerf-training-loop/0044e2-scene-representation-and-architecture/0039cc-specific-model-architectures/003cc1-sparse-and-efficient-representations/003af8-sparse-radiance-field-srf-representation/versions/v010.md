# Sparse Radiance Field (SRF) Representation


## Overview
A Sparse Radiance Field (SRF) is a voxel-based scene representation that explicitly stores density and view-dependent radiance colors in a sparse 3D grid, serving as a direct, differentiable alternative to the implicit Multi-Layer Perceptron (MLP) used in standard NeRFs [S1]. Unlike dense voxel grids or hierarchical structures, SRFs achieve high efficiency by storing features only for occupied regions in Coordinate (COO) format, enabling fast volume rendering and straightforward mesh extraction via Marching Cubes [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A set of posed 2D images of a scene or object.
*   **Output:** A Sparse Radiance Field $X_{non-empty}$, defined as a set of tuples containing 3D coordinates and associated feature vectors.

**Data Structures**
The SRF is represented in COO (Coordinate) format. It consists of $M$ tuples, where each tuple $m$ contains:
1.  **Coordinates** $c_m = (i, j, k)$: Positive integers indexing the voxel location in a 3D grid of size $H \times H \times H$.
2.  **Features** $f_m \in \mathbb{R}^{d+1}$: A vector containing:
    *   Density $\alpha_{i,j,k} \in \mathbb{R}$ (1 dimension).
    *   Radiance colors $\rho_{i,j,k} \in \mathbb{R}^d$ ($d$ dimensions).

The total grid dimension is $1+d$. The sparsity is defined as $1 - \frac{M}{H^3}$, where $M$ is the number of non-empty voxels [S1].

**Parameters**
*   **Resolution $H$:** The grid size in each dimension. Typical values are $H \in \{32, 128, 512\}$ [S1].
*   **Radiance Dimension $d$:**
    *   For view-dependent rendering using 4 spherical harmonics (SH) factors per RGB channel: $d = 3 \times 4 = 12$ [S1].
    *   For fixed RGB colors: $d = 3 \times 1 = 3$ [S1].
    *   Note: Using fewer SH components (e.g., 4 per channel vs 9) reduces optimization space by 40% and time by 10% with negligible PSNR loss [S1]. Using raw RGB reduces PSNR by ~1 dB but saves 80% space and 20% time [S1].

**Algorithm: SRF Construction and Rendering**
The SRF is typically optimized from images (e.g., using an adaptation of Plenoxels) [S1]. Once constructed, rendering proceeds as follows:

1.  **Ray Casting:** For each pixel, cast a ray through the scene bounding volume.
2.  **Voxel Sampling:** Identify all non-empty voxels intersected by the ray. Since the data is in COO format, this requires a spatial lookup or pre-computed ray-voxel intersection list.
3.  **Feature Retrieval:** For each intersected voxel $m$ with coordinate $c_m$, retrieve $f_m = (\alpha_m, \rho_m)$.
4.  **Volume Rendering:** Apply standard volumetric rendering integration along the ray segment within the voxel. The transmittance $T$ and color $C$ are updated:
    $$ T_{next} = T_{curr} \cdot \exp(-\sigma_m \cdot \Delta s) $$
    $$ C_{next} = C_{curr} + T_{curr} \cdot (1 - \exp(-\sigma_m \cdot \Delta s)) \cdot \rho_m $$
    Where $\sigma_m$ is derived from $\alpha_m$ (often $\sigma_m = -\ln(1-\alpha_m)/\Delta s$ or similar depending on discretization), and $\Delta s$ is the step size within the voxel [S1, G].
5.  **Output:** The final color $C_{next}$ is assigned to the pixel.

**Pitfalls and Edge Cases**
*   **Low-Density Noise:** Non-empty voxels may have very low densities that do not contribute to the final image but contain random color values. These can confuse learning networks if the SRF is used as input to another model [S1].
*   **Vanishing Gradients:** Due to high sparsity (~99%), gradients may vanish. Misalignment between input and output SRF coordinates (e.g., due to sparse convolutional strides) exacerbates this [S1].
*   **Precision:** Small errors in predicting density or radiance can cause large distortions in rendered images [S1].

## Operating characteristics
*   **Memory Complexity:** $O(M \cdot d)$, where $M$ is the number of non-empty voxels. This is significantly lower than dense grids $O(H^3 \cdot d)$ when $M \ll H^3$ [S1].
*   **Rendering Speed:** Faster than MLP-based NeRFs because it avoids per-point MLP inference; rendering is a direct lookup and integration [S1].
*   **Mesh Extraction:** Straightforward and efficient. Since SRFs lie on a 3D voxel space, meshes can be extracted with a single pass of Marching Cubes [S1].
*   **Limitations:** The irregular nature of the sparse grid can complicate standard 3D deep learning operations (like convolutions) due to coordinate misalignment and vanishing gradients [S1].

## Background
The Sparse Radiance Field (SRF) was introduced in the context of the SPARF (Sparse Radiance Fields) dataset and framework, which collected one million SRFs across multiple resolutions (32, 128, 512) for 13 object classes [S1]. The representation builds on the concept of Plenoxels, which replaced NeRF's MLP with spherical harmonics in a voxel grid, but extends it by explicitly managing sparsity via COO format to scale to large datasets and facilitate learning of 3D structures from few images [S1].

## Sources
[S1] SPARF: Large-Scale Learning of 3D Sparse Radiance Fields from Few Input Images, https://doi.org/10.1109/iccvw60793.2023.00315, fetched 2026-10-26
