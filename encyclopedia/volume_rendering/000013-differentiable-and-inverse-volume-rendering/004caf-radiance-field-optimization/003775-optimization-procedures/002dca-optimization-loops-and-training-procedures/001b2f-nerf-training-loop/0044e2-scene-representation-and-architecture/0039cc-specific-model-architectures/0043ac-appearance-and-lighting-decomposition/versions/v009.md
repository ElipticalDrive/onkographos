# Appearance and Lighting Decomposition

Also called: Neural Radiance Field Decomposition, View-Dependent Appearance Modeling

## Overview
Appearance and Lighting Decomposition refers to a family of neural radiance field (NeRF) architectures that decouple view-independent geometric and material properties from view-dependent lighting effects. This separation solves the problem of standard NeRF's inability to accurately model specular reflections, glossy surfaces, and complex lighting environments, which often result in blurry or incorrect highlights due to the entanglement of geometry and appearance in a single color prediction head [S2, S3]. Unlike standard NeRF methods that predict raw outgoing radiance, these methods predict physical parameters (such as normals, roughness, albedo, or reflection directions) to enable physically plausible rendering, relighting, and material editing [S1, S2, S3].

## Shared principles
All methods in this family build on [Specific Model Architectures](8585-specific-model-architectures.md). While the specific decomposition varies, the core principle is the decoupling of view-independent geometric/material features from view-dependent lighting effects.

**Input Representation:**
Raw coordinates $\mathbf{x}$ and $\mathbf{d}$ are mapped to a higher-dimensional space using positional encoding $\gamma(\cdot)$ to mitigate spectral bias [S1, S3].
$$
\gamma(\mathbf{x}) = (\sin(2^0 \pi \mathbf{x}), \cos(2^0 \pi \mathbf{x}), \dots, \sin(2^{L-1} \pi \mathbf{x}), \cos(2^{L-1} \pi \mathbf{x}))
$$
where $L$ is the number of frequency bands [S1, S3].

**Network Topology:**
The canonical implementation uses coupled Multi-Layer Perceptrons (MLPs). The density network $F_\sigma$ and color network $F_c$ are modified to output intermediate physical properties rather than raw color [S1, S2, S3].
1.  **Density/Geometry Network:** Takes $\gamma(\mathbf{x})$ as input. It consists of $L$ hidden layers with ReLU activations and skip connections from the input to every 4th layer [S1, S3]. It outputs density $\sigma$ and latent features representing material properties (e.g., normals, roughness, albedo) [S1, S2, S3].
2.  **Appearance/Lighting Network:** Takes the latent features, $\gamma(\mathbf{x})$, and $\mathbf{d}$ (or derived directions like reflection vectors) as input. It computes the final color $\mathbf{c}$ by combining material properties with lighting models (e.g., environment maps, specular models) [S1, S2, S3].

**Rendering Integration:**
During training, rays are sampled at points $\mathbf{x}_i$. The MLPs predict $\sigma_i$ and material properties. The pixel color $C(\mathbf{r})$ is computed via volumetric rendering:
$$
C(\mathbf{r}) = \sum_{i=1}^N T_i \alpha_i \mathbf{c}_i, \quad \text{where } \alpha_i = 1 - \exp(-\sigma_i \delta_i), \quad T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)
$$
where $\delta_i$ is the distance between samples [S1, S3]. The color $\mathbf{c}_i$ is derived from the decomposed components specific to the child method.

## Subtopics

### [Neural Light Field Representations](9273-neural-light-field-representations.md)
Neural Light Field Representations map camera ray parameters directly to final pixel color, bypassing the volumetric ray-marching and alpha-composition steps required by Neural Radiance Fields (NeRF). Unlike NeRF, which predicts density and radiance at sampled points along a ray requiring multiple forward passes per pixel, NeLF performs a single forward pass per ray to output the integrated RGB value directly. This approach enables orders-of-magnitude faster rendering speeds and reduced computational complexity (FLOPs) while maintaining or exceeding the visual quality of NeRF.

### [Anisotropic Neural Representation Learning](9475-anisotropic-neural-representation-learning.md)
Anisotropic Neural Representation Learning (ANRL) is a method for neural volume rendering that models volumetric density and latent features as view-dependent functions using spherical harmonics (SH) to eliminate directional ambiguity. Unlike standard NeRF which assumes isotropic point-sampled features, ANRL replaces the initial MLP output with SH coefficients, allowing the representation to capture anisotropy directly without shape-based sampling parameterization. This approach improves reconstruction quality in scenes with complex lighting or reflective surfaces while maintaining rendering efficiency.

### [Ref-NeRF](10843-ref-nerf.md)
Ref-NeRF is a neural radiance field method that improves the representation of specular reflections by replacing NeRF’s parameterization of view-dependent outgoing radiance with a representation of reflected radiance, structured using spatially-varying scene properties. It builds on [Specific Model Architectures](8585-specific-model-architectures.md) by decoupling view-independent geometric/material features from view-dependent lighting effects, specifically targeting the failure of standard NeRF to accurately capture glossy surfaces. Unlike its siblings, Ref-NeRF explicitly separates outgoing radiance into diffuse and specular components and reparameterizes the directional input using the reflection of the viewing vector about the local normal vector, enabling smoother interpolation of reflected radiance across scenes with varying materials.

### [Convex Combination Color Decoder](11505-convex-combination-color-decoder.md)
The Convex Combination Color Decoder is a method for decomposing view-dependent appearance into a diffuse-like base component and a glossy reflection component, weighted by a geometry-derived coefficient. It solves the problem of modeling specular highlights and environment reflections in neural radiance fields by ensuring that reflection features are utilized at every sample point along a ray, rather than just in image space, which aids convergence when geometry is initially fuzzy. It differs from standard NeRF color decoders by explicitly separating the view-dependent term from a reflection term that utilizes a pre-computed reflection feature $\bar{\mathbf{f}}$.

### DE-NeRF
- [S1] Decomposes lighting, geometry, and appearance to allow relighting of high-frequency environments and appearance editing.
- [S1] Train a signed distance field (SDF) function using a set of 2D images to reconstruct a display grid as the input scene for the MLP.
- [S1] Decompose the scene into geometric features and appearance features (diffuse reflection, roughness, specular tint).
- [S1] Input geometric features, appearance features, and distance h (from sampling point to grid) into four corresponding MLPs to predict diffuse albedo, roughness value, and specular tint.
- [S1] Integrate diffuse reflectance with the environment map to get diffuse color cd.
- [S1] Train a specular lighting MLP Fs to get specular lighting cl.
- [S1] Multiply specular lighting cl with specular hue to get specular color cs.
- [S1] Calculate final color c as cs + cd.
- [S1] On NeRF Synthetic and Shiny Blender data sets, PSNR is 2 points higher than latest papers.

### NeRFLight
- [S1] Extends grid-based methods by dividing the scene into eight different regions to split NeRF into separate density and color fields, using a shared feature grid.
- [S1] Avoids increasing the number of features for seamless reconstruction while reducing stored feature values.
- [S1] Divide the scene into eight different regions, each corresponding to an MLP that outputs density and color.
- [S1] Use a shared feature grid with features symmetrically placed at the center of the scene.

### NeRF-A
- [S2] A NeRF variant using appearance embeddings (NeRF-A) without transient embeddings for handling varying camera conditions.
- [S2] Train the model with the left half side of the images in the training set and test novel view synthesis on the right half side.
- [S2] Interpolate between appearance embeddings assigned to brighter and darker images for controllable rendering.
- [S2] Quantitative results on classroom image: PSNR 23.52, SSIM 0.907, LPIPS 0.247.

### MGNet Training Scheme
- [S3] Train the material-geometry module first to predict albedo, normal, roughness, metallic, and depth, because the lighting network depends on these properties (e.g., SSRT depends on depth, MLP decoder depends on GBuffers).
- [S3] Train the lighting module second, using the re-rendering loss.
- [S3] The total loss for MGNet is a weighted combination: $L_{MGNet} = λ_a L_{albedo} + λ_n L_{normal} + λ_m L_{material} + λ_d L_{depth}$.
- [S3] The total loss for LightNet is: $L_{LightNet} = L_{light} + λ_r L_{re-render}$.
- [S3] $L_{light}$ is the HDR supervision loss function proposed by [Mildenhall et al. 2021].
- [S3] $L_{re-render}$ is an $L_2$ loss between the re-rendered image and the original image.
- [S3] Li et al. [2020]'s rendering layer causes significant artifacts on glossy surfaces.
- [S3] Unlike Li et al. [2020] which discretizes the incident hemisphere to approximate integration, this method leverages differentiable Monte Carlo.

## Background
The decomposition of appearance and lighting in neural radiance fields emerged as a response to the limitations of standard NeRF in handling specularities and complex lighting. Standard NeRF predicts outgoing radiance directly, which entangles geometry and lighting, leading to blurry reflections and an inability to relight scenes [S2, S3]. Early works like DE-NeRF [S1] and Ref-NeRF [S2] introduced explicit separation of diffuse and specular components, often leveraging physical priors such as normals and roughness. Subsequent methods like NeRFLight [S1] and MGNet [S3] extended these ideas to grid-based representations and inverse rendering frameworks, enabling the estimation of material properties and lighting from unstructured photographs. The field has evolved to include anisotropic modeling [S1] and convex combination decoders [S1] to better handle the ambiguity between shape and appearance.

## Sources
[S1] Methods and Strategies for Improving the Novel View Synthesis Quality of Neural Radiation Field, https://doi.org/10.1109/access.2024.3382997, fetched 2026-10-30
[S2] HDR-Plenoxels: Self-Calibrating High Dynamic Range Radiance Fields, https://doi.org/10.1007/978-3-031-19824-3_23, fetched 2026-10-26
[S3] Learning-based Inverse Rendering of Complex Indoor Scenes with Differentiable Monte Carlo Raytracing, https://doi.org/10.1145/3550469.3555407, fetched 2026-10-26
