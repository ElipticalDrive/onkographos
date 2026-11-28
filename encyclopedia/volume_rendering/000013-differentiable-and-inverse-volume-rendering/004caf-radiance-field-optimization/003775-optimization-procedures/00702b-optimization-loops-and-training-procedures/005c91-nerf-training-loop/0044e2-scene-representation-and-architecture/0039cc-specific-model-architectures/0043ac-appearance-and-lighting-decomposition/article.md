# Appearance and Lighting Decomposition

Also called: Neural Radiance Field Decomposition, Inverse Rendering in NeRF, Material-Lighting Separation

## Overview
Appearance and Lighting Decomposition refers to a family of neural radiance field methods that decouple view-independent geometric and material properties from view-dependent lighting effects to enable physically plausible relighting, appearance editing, and robust reconstruction of specular surfaces [G]. Unlike standard NeRF, which maps 6D coordinates directly to raw color and density, these methods predict intermediate physical attributes (such as normals, roughness, albedo, or reflection directions) and combine them with lighting models or environment maps during the rendering process [G]. This separation resolves the shape-radiance ambiguity inherent in standard NeRF and allows for the accurate modeling of glossy and reflective surfaces that standard architectures fail to capture [G].

## Shared principles
All methods in this family build on [Specific Model Architectures](8585-specific-model-architectures.md). While the specific decomposition varies, the core principle is the decoupling of view-independent geometric/material features from view-dependent lighting effects [G].

**Input Representation:**
Raw coordinates $\mathbf{x}$ and $\mathbf{d}$ are mapped to a higher-dimensional space using positional encoding $\gamma(\cdot)$ to mitigate spectral bias [G].
$$
\gamma(\mathbf{x}) = (\sin(2^0 \pi \mathbf{x}), \cos(2^0 \pi \mathbf{x}), \dots, \sin(2^{L-1} \pi \mathbf{x}), \cos(2^{L-1} \pi \mathbf{x}))
$$
where $L$ is the number of frequency bands [S1, S3].

**Network Topology:**
The canonical implementation uses coupled Multi-Layer Perceptrons (MLPs) modified to output intermediate physical properties rather than raw color [G].
1.  **Density/Geometry Network:** Takes $\gamma(\mathbf{x})$ as input. It consists of $L$ hidden layers with ReLU activations and skip connections from the input to every 4th layer [S1, S3]. It outputs density $\sigma$ and latent features representing material properties (e.g., normals, roughness, albedo) [S1, S2, S3].
2.  **Appearance/Lighting Network:** Takes the latent features, $\gamma(\mathbf{x})$, and $\mathbf{d}$ (or derived directions like reflection vectors) as input. It computes the final color $\mathbf{c}$ by combining material properties with lighting models (e.g., environment maps, specular models) [S1, S2, S3].

**Rendering Integration:**
During training, rays are sampled at points $\mathbf{x}_i$ [G]. The MLPs predict $\sigma_i$ and material properties [G]. The pixel color $C(\mathbf{r})$ is computed via volumetric rendering:
$$
C(\mathbf{r}) = \sum_{i=1}^N T_i \alpha_i \mathbf{c}_i, \quad \text{where } \alpha_i = 1 - \exp(-\sigma_i \delta_i), \quad T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)
$$
where $\delta_i$ is the distance between samples [S1, S3]. The color $\mathbf{c}_i$ is derived from the decomposed components specific to the child method [G].

## Subtopics

### [Neural Light Field Representations](9273-neural-light-field-representations.md)
Neural Light Field Representations map camera rays directly to RGB color values, bypassing the volumetric ray-marching and alpha-composition steps required by Neural Radiance Fields (NeRF). Unlike NeRF, which requires multiple forward passes per ray to aggregate radiance, these methods achieve novel view synthesis with a single forward pass per ray, enabling rendering speeds that are orders of magnitude faster.

### [Anisotropic Neural Representation Learning](9475-anisotropic-neural-representation-learning.md)
Anisotropic Neural Representation Learning (ANRL) is a method for neural volume rendering that models volumetric density and latent features as view-dependent functions using spherical harmonics (SH) to eliminate directional ambiguity. Unlike standard NeRF which assumes isotropic point-sampled features, ANRL replaces the initial MLP output with SH coefficients, allowing the representation to capture anisotropy directly without shape-based sampling parameterization. This approach improves reconstruction quality in scenes with complex lighting or reflective surfaces while maintaining rendering efficiency.

### [Ref-NeRF](10843-ref-nerf.md)
Ref-NeRF is a neural radiance field method that improves the representation of specular reflections by replacing NeRF’s parameterization of view-dependent outgoing radiance with a representation of reflected radiance, structured using spatially-varying scene properties. It builds on [Specific Model Architectures](8585-specific-model-architectures.md) by decoupling view-independent geometric/material features from view-dependent lighting effects, specifically targeting the failure of standard NeRF to accurately capture glossy surfaces. Unlike its siblings, Ref-NeRF explicitly separates outgoing radiance into diffuse and specular components and reparameterizes the directional input using the reflection of the viewing vector about the local normal vector, enabling smoother interpolation of reflected radiance across scenes with varying materials.

### [Convex Combination Color Decoder](11505-convex-combination-color-decoder.md)
The Convex Combination Color Decoder is a method for decomposing view-dependent appearance into a diffuse-like base component and a glossy reflection component, weighted by a geometry-derived coefficient. It solves the problem of modeling specular highlights and environment reflections in neural radiance fields by ensuring that reflection features are utilized at every sample point along a ray, rather than just in image space, which aids convergence when geometry is initially fuzzy. It differs from standard NeRF color decoders by explicitly separating the view-dependent term from a reflection term that utilizes a pre-computed reflection feature $\bar{\mathbf{f}}$.

### [Unified Decompositional and Compositional NeRF Framework](14397-unified-decompositional-and-compositional-nerf-framework.md)
This method solves the problem of disentangling foreground objects from background environments in a radiance field to enable editable novel view synthesis, where foreground objects can be removed or replaced without corrupting the background. It sets itself apart from sibling methods like [Ref-NeRF](10843-ref-nerf.md) by explicitly modeling separate background and object radiance fields that are composited via a differentiable one-hot gating mechanism, rather than relying solely on view-dependent lighting models or anisotropic scattering. Unlike [Convex Combination Color Decoder](11505-convex-combination-color-decoder.md) which focuses on color mixing, this framework enforces spatial separation through density regularization and Gumbel-Softmax activation.

### DE-NeRF
DE-NeRF decomposes lighting, geometry, and appearance to allow relighting of high-frequency environments and appearance editing [S1]. It trains a signed distance field (SDF) function using a set of 2D images to reconstruct a display grid as the input scene for the MLP [S1]. The scene is decomposed into geometric features and appearance features (diffuse reflection, roughness, specular tint) [S1]. Geometric features, appearance features, and distance $h$ (from sampling point to grid) are input into four corresponding MLPs to predict diffuse albedo, roughness value, and specular tint [S1]. Diffuse reflectance is integrated with the environment map to get diffuse color $\mathbf{c}_d$ [S1]. A specular lighting MLP $F_s$ is trained to get specular lighting $\mathbf{c}_l$ [S1]. The specular color $\mathbf{c}_s$ is calculated by multiplying $\mathbf{c}_l$ with specular hue [S1]. The final color $\mathbf{c}$ is calculated as $\mathbf{c}_s + \mathbf{c}_d$ [S1]. On NeRF Synthetic and Shiny Blender data sets, PSNR is 2 points higher than latest papers [S1].

### [NeRFLight](9113-nerflight.md)
NeRFLight is a grid-based neural radiance field architecture designed to achieve real-time view synthesis with a significantly reduced memory footprint compared to standard grid-based methods like Instant-NGP or KiloNeRF. It solves the trade-off between rendering speed and model size by splitting the 3D scene volume into $N$ distinct spatial regions, each utilizing a separate density decoder but sharing a single, compact feature grid. Unlike Instant-NGP, which uses hash grids and multiresolution schemes that can suffer from rendering speed losses due to multiple linear interpolations, NeRFLight avoids hash grids entirely and instead uses multiple density decoders to handle feature collisions, ensuring high frame rates.

### NeRF-A
NeRF-A is a NeRF variant using appearance embeddings (NeRF-A) without transient embeddings for handling varying camera conditions [S2]. Train the model with the left half side of the images in the training set and test novel view synthesis on the right half side [S2]. Interpolate between appearance embeddings assigned to brighter and darker images for controllable rendering [S2]. Quantitative results on classroom image: PSNR 23.52, SSIM 0.907, LPIPS 0.247 [S2].

### MGNet Training Scheme
Train the material-geometry module first to predict albedo, normal, roughness, metallic, and depth, because the lighting network depends on these properties (e.g., SSRT depends on depth, MLP decoder depends on GBuffers) [S3]. Train the lighting module second, using the re-rendering loss [S3]. The total loss for MGNet is a weighted combination: $L_{MGNet} = \lambda_a L_{albedo} + \lambda_n L_{normal} + \lambda_m L_{material} + \lambda_d L_{depth}$ [S3]. The total loss for LightNet is: $L_{LightNet} = L_{light} + \lambda_r L_{re-render}$ [S3]. $L_{light}$ is the HDR supervision loss function proposed by [Mildenhall et al. 2021] [S3]. $L_{re-render}$ is an $L_2$ loss between the re-rendered image and the original image [S3]. Li et al. [2020]'s rendering layer causes significant artifacts on glossy surfaces [S3]. Unlike Li et al. [2020] which discretizes the incident hemisphere to approximate integration, this method leverages differentiable Monte Carlo [S3].

### NeROIC
NeROIC is a technique using a multi-stage pipeline to recover a relightable NeRF-like model from images of an object captured under multiple unconstrained lighting environments [S4, unofficial]. A method that recovers NeRF-like relightable models of large-scale buildings and historical sites [S4, unofficial]. First stage: recovers geometry while explaining appearance variations due to lighting with latent appearance embeddings [S4, unofficial]. Second stage: extracts normal vectors from this recovered geometry [S4, unofficial]. Third stage: estimates BRDF properties and a spherical harmonic representation of lighting [S4, unofficial]. Decomposes scenes into diffuse albedo, surface normals, a spherical harmonics representation of lighting, and shadows, which can be combined to relight the scene under novel environment illumination [S4, unofficial]. Assumes a Lambertian model [S4, unofficial].

## Background
The concept of decomposing appearance and lighting in neural radiance fields emerged as a response to the limitations of standard NeRF in handling specular reflections and complex lighting [G]. Standard NeRF assumes isotropic scattering and maps viewing direction directly to color, which leads to artifacts on glossy surfaces and an inability to separate geometry from material properties [S2, S3]. Early works like Ref-NeRF [S2] and DE-NeRF [S1] introduced explicit material parameters (normals, roughness) and lighting models to address these issues. Subsequent methods like NeRFLight [S1] and MGNet [S3] explored grid-based decompositions and inverse rendering techniques to estimate lighting and materials from images. The family continues to evolve with methods addressing anisotropy [S1], background decomposition [S1], and efficient rendering [S2].

## Sources
[S1] Methods and Strategies for Improving the Novel View Synthesis Quality of Neural Radiation Field, https://doi.org/10.1109/access.2024.3382997, fetched 2026-10-30
[S2] HDR-Plenoxels: Self-Calibrating High Dynamic Range Radiance Fields, https://doi.org/10.1007/978-3-031-19824-3_23, fetched 2026-10-26
[S3] Learning-based Inverse Rendering of Complex Indoor Scenes with Differentiable Monte Carlo Raytracing, https://doi.org/10.1145/3550469.3555407, fetched 2026-10-26
[S4] Advances in Neural Rendering, https://doi.org/10.48550/arxiv.2111.05849, fetched 2026-10-26 (unofficial)
