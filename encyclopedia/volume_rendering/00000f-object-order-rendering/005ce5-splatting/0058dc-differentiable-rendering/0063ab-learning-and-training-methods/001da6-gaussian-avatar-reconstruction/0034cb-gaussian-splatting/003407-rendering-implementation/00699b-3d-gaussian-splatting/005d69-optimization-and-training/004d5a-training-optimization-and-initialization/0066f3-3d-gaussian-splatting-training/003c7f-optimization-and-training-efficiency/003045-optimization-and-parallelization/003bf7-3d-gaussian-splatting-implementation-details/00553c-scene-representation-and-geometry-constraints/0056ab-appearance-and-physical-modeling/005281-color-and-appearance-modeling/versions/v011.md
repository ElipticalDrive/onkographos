# Color and Appearance Modeling

Also called: Appearance Modeling, View-Dependent Color Modeling

## Overview
Color and Appearance Modeling extends the standard 3D Gaussian Splatting (3DGS) representation by replacing or augmenting the fixed Spherical Harmonic (SH) coefficients with more flexible mechanisms to represent view-dependent color, lighting, and material properties. It solves the limitations of base 3DGS, which struggles with complex specularities, dynamic lighting variations, and transient objects in unstructured datasets. It sets itself apart from sibling methods like [Physics and Optical Modeling](15148-physics-and-optical-modeling.md) by focusing on the radiance and material representation rather than volumetric light transport equations.

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

### [DAVIGS](12021-davigs.md)
DAVIGS is a plug-and-post appearance modeling method for 3D Gaussian Splatting that mitigates floaters and color distortions caused by appearance variations (e.g., daylight changes, ISP processing, local lighting) by transforming rendered images at the image level using 3D consistent features. Unlike coupled methods that manipulate Gaussian properties and slow rendering, or decoupled methods that only handle mild global variations, DAVIGS decouples global view-dependent appearance from local spatial changes, ensuring 3D consistency while maintaining real-time rendering speeds by discarding the appearance module after optimization. See [DAVIGS](12021-davigs.md).

### [Compact View-dependent Color](12904-compact-view-dependent-color.md)
Compact View-dependent Color replaces the per-Gaussian Spherical Harmonics (SH) coefficients with a shared grid-based neural field to represent view-dependent color, significantly reducing the parameter count from 48 coefficients per Gaussian to a compact neural representation. Unlike sibling methods that use post-processing transformations [DAVIGS](12021-davigs.md), factorized illumination [3iGS](13977-3igs-factorised-tensorial-illumination-for-3d-gaussian-splat.md), or image-conditioned modulation [SWAG](14315-swag-splatting-in-the-wild-images-with-appearance-conditione.md), this method substitutes the directional encoding entirely with a spatial-hash-based MLP, addressing the disk space explosion caused by high-order SH storage. See [Compact View-dependent Color](12904-compact-view-dependent-color.md).

### [3iGS (Factorised Tensorial Illumination for 3D Gaussian Splatting)](13977-3igs-factorised-tensorial-illumination-for-3d-gaussian-splat.md)
3iGS enhances 3D Gaussian Splatting (3DGS) by modeling outgoing radiance as a function of a continuous local illumination field and individual Gaussian BRDF attributes, rather than optimizing outgoing radiance in isolation. It solves the problem of inaccurate specular view-dependent effects in reflective scenes by separating scene lighting from material properties, enabling physically plausible shading while maintaining the rapid rendering speeds of rasterization. Unlike vanilla 3DGS, which uses spherical harmonics for view-dependent color without physical interpretability, 3iGS explicitly models diffuse color and specular highlights via a neural renderer and a factorized tensor illumination grid. See [3iGS (Factorised Tensorial Illumination for 3D Gaussian Splatting)](13977-3igs-factorised-tensorial-illumination-for-3d-gaussian-splat.md).

### [Spherical Harmonics Optimization](14206-spherical-harmonics-optimization.md)
Spherical Harmonics (SH) Optimization refers to the family of methods that determine, compress, or regularize the view-dependent color coefficients $\mathbf{c}_i(\omega)$ stored per 3D Gaussian primitive in 3D Gaussian Splatting (3DGS). It solves the problem of efficiently representing complex lighting and specular effects while managing memory bandwidth, training stability, and computational cost. It differs from its siblings by focusing specifically on the SH coefficient representation itself (storage, degree, or estimation) rather than the camera pose optimization [10077-explicit-camera-jacobian-derivation.md] or the multi-view training loop structure [13982-efficient-multi-view-training-for-3d-gaussian-splatting.md]. See [Spherical Harmonics Optimization](14206-spherical-harmonics-optimization.md).

### [SWAG (Splatting in the Wild images with Appearance-conditioned Gaussians)](14315-swag-splatting-in-the-wild-images-with-appearance-conditione.md)
SWAG is an extension of 3D Gaussian Splatting (3DGS) designed to handle unstructured, in-the-wild image collections by modeling appearance variations and handling transient objects (occluders) without supervision. It sets itself apart from base 3DGS and siblings like [DAVIGS](12021-davigs.md) and [3iGS](13977-3igs-factorised-tensorial-illumination-for-3d-gaussian-splat.md) by using a per-image embedding vector and an MLP to modulate Gaussian color and opacity, enabling the disentangling of static scene geometry from dynamic occluders and lighting conditions. See [SWAG (Splatting in the Wild images with Appearance-conditioned Gaussians)](14315-swag-splatting-in-the-wild-images-with-appearance-conditione.md).

#### GeoSplatting PBR Attribute Modeling
**Definition**: Uses multi-resolution hash grids with MLP headers to model PBR attributes (albedo, roughness, metallic) as continuous fields, avoiding learnable parameters for varying numbers of Gaussian points [S1].
**Layout**: Ed HashEnc: 16 levels, $2^{19}$ max entries per level, 2 feature dimensions per entry, coarsest resolution 32, finest resolution 4096. Ed MLP: 32x32x32x6 layers, Kaiming-uniform init, Sigmoid activation. Es MLP: 32x32x3 layers, Kaiming-uniform init, None activation [S1].
**Code**: `diffAlbedo = E_d(centerPos)`, `(roughness, metalness) = E_s(centerPos)` [S1].

## Background
Originating from the limitations of base 3DGS in handling complex lighting and dynamic scenes, these methods evolved to incorporate neural fields, factorized illumination, and image-conditioned modulation. They are primarily used in novel view synthesis, inverse rendering, and augmented reality applications where high-fidelity appearance and physical plausibility are required.

## Sources
[S1] GeoSplatting: Towards Geometry Guided Gaussian Splatting for Physically-Based Inverse Rendering, https://doi.org/10.1109/ICCV51701.2025.02692, fetched 2026-10-30
