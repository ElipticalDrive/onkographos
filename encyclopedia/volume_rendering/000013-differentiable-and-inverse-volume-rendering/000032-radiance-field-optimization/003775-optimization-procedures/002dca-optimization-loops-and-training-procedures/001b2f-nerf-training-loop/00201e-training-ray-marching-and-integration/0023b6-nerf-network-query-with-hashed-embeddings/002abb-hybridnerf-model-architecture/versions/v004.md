# HybridNeRF Model Architecture

Also called: HybridNeRF

## Overview
HybridNeRF is a neural rendering architecture that combines dense multi-resolution 3D feature grids with multi-resolution triplanes to featurize 3D sample locations, enabling efficient training and rendering of large-scale scenes [S1]. It solves the problem of balancing high-frequency detail capture with memory efficiency by using a hybrid encoding scheme that sums features across resolutions rather than concatenating them [S1]. It sets itself apart from pure grid-based methods like [Simple-ZipNeRF](7424-simple-zipnerf.md) by incorporating triplanes for view-dependent effects and from pure triplane methods by retaining 3D grids for geometry, while differing from [VoxNeRF](8865-voxnerf.md) by its specific adaptive surfaceness formulation [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** 3D spatial coordinates $\mathbf{x}$, viewing direction $\mathbf{v}$.
*   **Outputs:** Density $\sigma$, signed distance $f$, and color $\mathbf{c}$.

**Data Structures:**
1.  **3D Feature Grids:** Three resolution levels with dimensions $128^3$, $256^3$, and $512^3$. Each cell stores a feature vector of size $K=4$.
2.  **Triplanes:** Seven resolution levels ranging from $128^2$ to $8,192^2$. Each plane stores a feature vector of size $K=4$.
3.  **Surfaceness Field:** A $512^3$ voxel grid storing scalar values $\beta(\mathbf{x})$, queried via nearest-neighbor interpolation [S1].
4.  **MLPs:** Separate small MLPs for predicting signed distance $f$ and color $\mathbf{c}$ [S1].

**Algorithm:**

1.  **Feature Fetching (3D Grids):** For the input point $\mathbf{x}$, fetch $K=4$ features from each of the 3 resolution levels of the 3D feature grids using trilinear interpolation. Let the fetched features at level $l$ be $\mathbf{g}_l(\mathbf{x})$.
2.  **Feature Fetching (Triplanes):** For the input point $\mathbf{x}$, project $\mathbf{x}$ onto the three orthogonal planes (XY, YZ, XZ) for each of the 7 resolution levels. Fetch $K=4$ features from each level using bilinear interpolation. Let the fetched features at level $l$ be $\mathbf{t}_l(\mathbf{x})$.
3.  **Feature Aggregation:**
    *   Sum the features across all 3 resolution levels of the 3D grids: $\mathbf{G}(\mathbf{x}) = \sum_{l=1}^{3} \mathbf{g}_l(\mathbf{x})$.
    *   Sum the features across all 7 resolution levels of the triplanes: $\mathbf{T}(\mathbf{x}) = \sum_{l=1}^{7} \mathbf{t}_l(\mathbf{x})$.
    *   *Note:* This summation strategy differs from concatenation used in other methods [S1].
4.  **MLP Input Construction:** Concatenate the summed 3D grid features and triplane features: $\mathbf{h}(\mathbf{x}) = [\mathbf{G}(\mathbf{x}); \mathbf{T}(\mathbf{x})]$. The resulting vector has dimension $4K + 4K = 16$ (since $K=4$) [S1].
5.  **Direction Encoding:** Encode the viewing direction $\mathbf{v}$ using spherical harmonics up to the 4th degree. Let this encoded vector be $\mathbf{e}_{SH}(\mathbf{v})$ [S1].
6.  **Signed Distance Prediction:** Pass $\mathbf{h}(\mathbf{x})$ through the distance MLP to obtain the raw signed distance $f(\mathbf{x})$ [S1].
7.  **Density Calculation:** Compute the density $\sigma_{\text{SDF}}(\mathbf{x})$ using the spatially adaptive surfaceness field $\beta(\mathbf{x})$ and the Laplace CDF $\Psi$:
    $$ \sigma_{\text{SDF}}(\mathbf{x}) = \beta(\mathbf{x})\Psi(f(\mathbf{x})\beta(\mathbf{x})) $$
    where $\Psi(s)$ is defined as:
    $$ \Psi(s) = \begin{cases} \frac{1}{2} \exp(-s) & \text{if } s > 0 \\ 1 - \frac{1}{2} \exp(s) & \text{if } s \leq 0. \end{cases} $$
    Here, $s = f(\mathbf{x})\beta(\mathbf{x})$ [S1].
8.  **Color Prediction:** Concatenate the feature vector $\mathbf{h}(\mathbf{x})$ and the direction encoding $\mathbf{e}_{SH}(\mathbf{v})$, then pass through the color MLP to obtain color $\mathbf{c}$ [S1].

**Parameters:**
*   $K=4$: Number of features fetched per level [S1].
*   3D Grid Levels: 3 levels ($128^3, 256^3, 512^3$) [S1].
*   Triplane Levels: 7 levels ($128^2$ to $8,192^2$) [S1].
*   Surfaceness Grid Resolution: $512^3$ [S1].
*   Spherical Harmonics Degree: 4 [S1].

**Edge Cases and Pitfalls:**
*   **Aliasing:** Due to the multi-resolution feature grid, aliasing artifacts can occur. HybridNeRF handles this using the technique described in VR-NeRF [S1].
*   **Surfaceness Initialization:** The surfaceness parameter $\beta(\mathbf{x})$ is initially constrained to be globally uniform and allowed to diverge spatially during the finetuning stage to prevent degradation of fine objects and transparent structures [S1].

## Subtopics

### [Shading Replacement for Relighting](6346-shading-replacement-for-relighting.md)
Shading Replacement for Relighting is a technique for object insertion that avoids inferring scene illumination by replacing the inserted object's shading with shading from existing scene points. It solves the problem of lighting inconsistency when compositing objects into neural radiance field (NeRF) scenes, where naive composition leads to unrealistic results due to varying illumination. It sets itself apart from direct relighting methods by not requiring explicit light source estimation, instead relying on the correlation between surface normals and shading in Lambertian scenes. See [Shading Replacement for Relighting](6346-shading-replacement-for-relighting.md).

### [Hybrid Multi-Space Module](11712-hybrid-multi-space-module.md)
The Hybrid Multi-Space Module extends grid-based NeRF methods (such as iNGP and TensoRF) to handle multiple sub-spaces (e.g., reflections and refractions) by decomposing the multi-space composition information into a small MLP branch, while the main branch models multiple radiance fields using explicit grids. Unlike pure feature-field approaches, this module integrates discrete learnable grids with small MLPs to output densities and colors for each sub-space, enabling efficient rendering of complex light paths in grid-based architectures. It builds on [HybridNeRF Model Architecture](4842-hybridnerf-model-architecture.md). See [Hybrid Multi-Space Module](11712-hybrid-multi-space-module.md).

### Geometry Composition by Segmented Sampling
Geometry Composition by Segmented Sampling handles the integration of external objects into a scene by transforming sampling points from scene coordinates to object coordinates using the inserting position (translation and rotation) [S2]. It composites density as the normalized sum of densities of corresponding points from the scene and object at the same position, and composites reflectance as a weighted sum of reflectance values, where weights are the densities of the voxel particles [S2]. The final image is rendered using the composited density, reflectance, and shading [S2]. This method addresses issues where direct composition leads to missing areas of the inserted object [S2].

### HybridNeRF SDF Density Formulation
The HybridNeRF SDF Density Formulation defines density $\sigma_{\text{SDF}}(\mathbf{x})$ as $\beta(\mathbf{x})\Psi(f(\mathbf{x})\beta(\mathbf{x}))$, where $f(\mathbf{x})$ is the signed distance field output, $\beta(\mathbf{x})$ is the surfaceness parameter, and $\Psi$ is the CDF of a standard Laplace distribution [S1]. The Laplace CDF $\Psi(s)$ is defined piecewise: $\frac{1}{2} \exp(-s)$ if $s > 0$, and $1 - \frac{1}{2} \exp(s)$ if $s \leq 0$, with $s = f(\mathbf{x})\beta(\mathbf{x})$ [S1]. The surfaceness field $\beta(\mathbf{x})$ is stored in a $512^3$ voxel grid and queried via nearest-neighbor interpolation [S1]. The parameter $\beta(\mathbf{x}) > 0$ determines the surfaceness of point $\mathbf{x}$, controlling how concentrated the density is around the zero-level set of $f$ [S1].

## Operating characteristics
*   **Memory Complexity:** The memory usage is dominated by the 3D feature grids ($128^3, 256^3, 512^3$) and the triplanes (7 levels up to $8,192^2$), plus the $512^3$ surfaceness grid. The summation of features across levels reduces the MLP input dimension compared to concatenation, potentially reducing MLP memory footprint [S1].
*   **Time Complexity:** Feature fetching involves trilinear interpolation for 3D grids and bilinear interpolation for triplanes, followed by MLP inference. The multi-resolution approach allows efficient detail capture without requiring extremely high-resolution single grids [S1].
*   **Strengths:** Efficient handling of large-scale scenes; ability to capture fine details via multi-resolution grids; adaptive surfaceness allows for sharp surface reconstruction [S1].
*   **Limitations:** Requires careful tuning of surfaceness parameters to avoid artifacts on fine objects or transparent structures; aliasing requires specific handling [S1].
*   **When to use:** Suitable for large-scale scene rendering where both geometry and view-dependent effects need to be captured efficiently [S1].
*   **When not to use:** Not ideal for scenes with extreme view-dependent effects that cannot be captured by triplanes alone, or when memory is extremely constrained [S1].

## Background
HybridNeRF was introduced in the paper "HybridNeRF: Efficient Neural Rendering via Adaptive Volumetric Surfaces" [S1]. It builds upon previous work in neural radiance fields, specifically incorporating ideas from VolSDF for SDF-based density representation and VR-NeRF for aliasing handling [S1]. The method was developed to address the trade-off between rendering quality and computational efficiency in large-scale neural rendering tasks [S1].

## Sources
[S1] HybridNeRF: Efficient Neural Rendering via Adaptive Volumetric Surfaces, https://doi.org/10.1109/cvpr52733.2024.01858, fetched 2026-10-26
[S2] Relighting Scenes With Object Insertions in Neural Radiance Fields, https://doi.org/10.1109/TCSVT.2025.3535599, fetched 2026-10-29
