# Appearance and Physical Modeling

Also called: Relightable Gaussians, Inverse Rendering Gaussians, Physically-Based Gaussian Splatting

## Overview
Appearance and Physical Modeling extends the standard 3D Gaussian Splatting (3DGS) representation to support physically based rendering (PBR), relighting, and volumetric light transport, solving the limitations of base 3DGS which relies on fixed Spherical Harmonic (SH) coefficients that cannot accurately represent complex specularities, dynamic lighting, or non-vacuum media. It sets itself apart from sibling methods like [Gaussian Representation and Truncation](12855-gaussian-representation-and-truncation.md) by focusing on the radiance and material representation rather than geometric simplification, and distinguishes itself from [Physics and Optical Modeling](15148-physics-and-optical-modeling.md) by prioritizing surface material decomposition and relighting over volumetric attenuation equations, although both families modify the core Gaussian properties beyond simple color.

## Shared principles
All methods in this family operate on the standard 3D Gaussian Splatting representation, where the scene is defined by a set of $N$ 3D Gaussians $\{G_i\}_{i=1}^N$. Each Gaussian $G_i$ is defined by its mean $\boldsymbol{\mu}_i$, covariance matrix $\boldsymbol{\Sigma}_i$ (derived from rotation $\mathbf{R}_i$ and scale $\mathbf{S}_i$), opacity $\alpha_i$, and view-dependent color $\mathbf{c}_i(\omega)$.

The rendering process evaluates the Gaussian density along a ray $\mathbf{r}(t) = \mathbf{o} + t\omega$ and integrates contributions. The Gaussian density function is:
$$ G(\mathbf{x}) = \exp\left(-\frac{1}{2}(\mathbf{x} - \boldsymbol{\mu})^\top \boldsymbol{\Sigma}^{-1} (\mathbf{x} - \boldsymbol{\mu})\right) $$

The core variation across methods lies in the definition, storage, and optimization of $\mathbf{c}_i(\omega)$ and potentially $\alpha_i$, moving beyond the fixed SH coefficients used in base 3DGS. Common strategies include:
1.  **Post-Processing Transformation:** Applying a learned affine transformation to the rendered image based on 3D position and view embedding [14740-color-and-appearance-modeling.md].
2.  **Factorized Illumination:** Separating scene lighting (via a tensor grid) from material properties (via BRDF features) to model specular effects [14740-color-and-appearance-modeling.md].
3.  **Image-Conditioned Modulation:** Using an MLP and per-image embeddings to modulate Gaussian color and opacity, enabling transient object removal and appearance synthesis [14740-color-and-appearance-modeling.md].
4.  **Neural Field Substitution:** Replacing per-Gaussian SH with a shared grid-based neural field to reduce parameters and improve generalization [14740-color-and-appearance-modeling.md].
5.  **Volumetric Attenuation:** Modifying the alpha-blending equation to incorporate medium properties, such as direct attenuation $B_d$ and backscatter parameters $B_\infty, B_b$, to model light transport in non-vacuum media [15148-physics-and-optical-modeling.md].

## Subtopics

### [Color and Appearance Modeling](14740-color-and-appearance-modeling.md)
Color and Appearance Modeling extends the standard 3D Gaussian Splatting (3DGS) representation by replacing or augmenting the fixed Spherical Harmonic (SH) coefficients with more flexible mechanisms to represent view-dependent color, lighting, and material properties. It solves the limitations of base 3DGS, which struggles with complex specularities, dynamic lighting variations, and transient objects in unstructured datasets. It sets itself apart from sibling methods like [Physics and Optical Modeling](15148-physics-and-optical-modeling.md) by focusing on the radiance and material representation rather than volumetric light transport equations.

### [Physics and Optical Modeling](15148-physics-and-optical-modeling.md)
Physics and Optical Modeling extends the standard 3D Gaussian Splatting (3DGS) representation to handle non-vacuum media and physically consistent surface extraction by modifying the rendering equation or the underlying density field. It solves the problem of light transport in scattering media (e.g., underwater fog) and the geometric inconsistency of discrete alpha-blending for surface reconstruction. Unlike [Color and Appearance Modeling](14740-color-and-appearance-modeling.md), which focuses on view-dependent shading and material properties in clear space, this family explicitly models volumetric attenuation and backscatter or reformulates the Gaussian primitive as a continuous attenuation field.

### Relightable 3D Gaussians
**Definition**
Relightable 3D Gaussians is a differentiable point-based rendering framework that enhances vanilla 3D Gaussians by associating extra properties (normal vectors, BRDF parameters, and incident lighting) to achieve photo-realistic relighting and BRDF decomposition from multi-view images [S1].

**Implementation Steps**
1.  Assign a normal vector $\mathbf{n}$, BRDF values, and incident light information to each 3D Gaussian point to model per-point light reflectance [S1].
2.  Apply physically based rendering (PBR) to get a PBR color for each 3D Gaussian point, which is then alpha-composited to obtain the rendered color for the corresponding image pixel [S1].
3.  Split the incident light into a global environment map and an indirect incident light field for robust material and lighting estimation [S1].
4.  Introduce proper regularizations to enhance geometry and mitigate material-lighting ambiguity, including constraints on depth distribution, smoothness priors, and a lighting regularization [S1].

### GeoSplatting Loss Function
**Definition**
GeoSplatting Loss Function is a composite loss function for end-to-end training of GeoSplatting, combining photometric loss, entropy loss, smoothness regularization, and light regularization [S2].

**Implementation Steps**
1.  Compute L1 loss and SSIM loss between rendered image and ground truth [S2].
2.  Compute object mask loss $L_{mask}$ (MSE between rendered alpha map and ground truth object mask) [S2].
3.  Add entropy loss $L_{entropy}$ to constrain the shape, following DMTet and FlexiCubes [S2].
4.  Apply smoothness regularization on $k_d$ and $k_s$, and light regularization, following NVdiffrecmc and R3DG [S2].

**Formula**
The photometric loss is computed as:
$$ L_{img} = L_1 + \lambda_{ssim} L_{SSIM} + \lambda_{mask} L_{mask} $$
where $L_{mask}$ is MSE between rendered alpha map and ground truth object mask [S2].

**Comparison**
Unlike vanilla 3DGS or other methods requiring dist loss or pseudo depth normal loss for geometric constraints, GeoSplatting does not require these regularization terms due to explicit geometry guidance [S2].

### Split-Sum Warm-up
**Definition**
Split-Sum Warm-up is a training initialization technique that uses the split-sum approximation to pre-compute lighting integrals during early training stages to mitigate optimization instability caused by noisy geometry [S2].

**Implementation Steps**
1.  During early training, use the split-sum approximation (Eq. 6) which enables fast pre-computation and enhances evaluation speed [S2].
2.  After the warm-up stage, transition to Monte Carlo sampling of the full rendering equation (Eq. 2) to incorporate shadow effects and inter-reflection [S2].

**Formula**
The split-sum approximation assumes no self-occlusion and is defined as:
$$ L_o(\omega_o) \approx \left[ \int_{\mathcal{H}} f_r(\omega_i, \omega_o) |\mathbf{n} \cdot \omega_i| \, d\omega_i \right] \cdot \left[ \int_{\mathcal{H}} L_{dir}(\omega_i) D |\mathbf{n} \cdot \omega_i| \, d\omega_i \right] $$
[S2]

**Pitfall**
Directly evaluating the full integral in Eq. 2 is inefficient and may lead to optimization instability during initial training when geometry is noisy. Split-sum avoids this but lacks self-occlusion modeling, potentially baking shadows into albedo [S2].

## Background
Relightable 3D Gaussians was introduced by Gao et al. in 2024, presenting a framework for photo-realistic relighting using BRDF decomposition and ray tracing [S1]. GeoSplatting was introduced by its authors in 2025, focusing on geometry-guided physically-based inverse rendering [S2]. These methods address the need for editable, physically plausible scene representations in computer graphics and vision applications, moving beyond static appearance capture.

## Sources
[S1] Relightable 3D Gaussians: Realistic Point Cloud Relighting with BRDF Decomposition and Ray Tracing, https://doi.org/10.1007/978-3-031-72995-9_5, fetched 2026-10-26
[S2] GeoSplatting: Towards Geometry Guided Gaussian Splatting for Physically-Based Inverse Rendering, https://doi.org/10.1109/ICCV51701.2025.02692, fetched 2026-10-30
