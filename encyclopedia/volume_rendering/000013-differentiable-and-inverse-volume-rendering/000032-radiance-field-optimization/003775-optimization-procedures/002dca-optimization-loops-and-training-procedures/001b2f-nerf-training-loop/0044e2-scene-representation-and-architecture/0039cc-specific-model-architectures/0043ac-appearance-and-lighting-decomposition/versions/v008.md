# Appearance and Lighting Decomposition


## Overview
Appearance and Lighting Decomposition refers to a family of neural radiance field architectures that explicitly separate scene geometry, material properties (diffuse/specular), and lighting conditions to enable relighting, appearance editing, and accurate rendering of reflective surfaces. Unlike standard NeRF which maps position and viewing direction directly to color, these methods decompose the rendering equation into distinct components (e.g., diffuse albedo, roughness, specular tint, or reflection directions) [S1, S2, S3]. This separation resolves the ambiguity between shape and material, allowing for physically plausible interactions with environment maps and improved handling of glossy or specular highlights [S1, S2, S3].

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

### [Anisotropic Neural Representation Learning](9475-anisotropic-neural-representation-learning.md)
Anisotropic Neural Representation Learning models volumetric density and latent features as view-dependent functions using spherical harmonics (SH) to eliminate directional ambiguity, replacing the isotropic point-sampled features of standard NeRF. Unlike Mip-NeRF which samples cones to approximate pre-filtered representations, this method modifies the MLP to predict view-dependent SH coefficients directly, avoiding the need for shape-based sampling parameterization.

### [Ref-NeRF](10843-ref-nerf.md)
Ref-NeRF improves the representation of specular reflections by replacing NeRF's parameterization of view-dependent outgoing radiance with a representation of reflected radiance, structured using spatially-varying scene properties. It builds on [Mip-NeRF](52-mip-nerf.md) and modifies the standard architecture to explicitly model diffuse and specular components, enabling accurate rendering of glossy surfaces. Unlike standard NeRF, which struggles with reflections due to foggy geometry and noisy normals, Ref-NeRF introduces a density regularizer to concentrate density around surfaces and uses Integrated Directional Encoding (IDE) to smooth the interpolation of reflected radiance.

### [Convex Combination Color Decoder](11505-convex-combination-color-decoder.md)
The Convex Combination Color Decoder predicts view-dependent appearance in neural radiance fields by blending a direct view-dependent color component with a reflection-based color component. It addresses the challenge of modeling glossy or reflective surfaces where standard view-dependent MLPs fail to capture consistent reflections across varying viewpoints. Unlike the standard NeRF color decoder which maps position and direction directly to color, this method explicitly separates the appearance into a base view-dependent term and a reflection term, weighted by a geometry-derived coefficient.

#### DE-NeRF
DE-NeRF decomposes lighting, geometry, and appearance to allow relighting of high-frequency environments and appearance editing [S1]. It trains a signed distance field (SDF) function using a set of 2D images to reconstruct a display grid as the input scene for the MLP [S1]. The scene is decomposed into geometric features and appearance features (diffuse reflection, roughness, specular tint) [S1]. Geometric features, appearance features, and distance $h$ (from sampling point to grid) are input into four corresponding MLPs to predict diffuse albedo, roughness value, and specular tint [S1]. Diffuse reflectance is integrated with the environment map to get diffuse color $\mathbf{c}_d$ [S1]. A specular lighting MLP $F_s$ is trained to get specular lighting $\mathbf{c}_l$ [S1]. The specular color $\mathbf{c}_s$ is obtained by multiplying $\mathbf{c}_l$ with specular hue [S1]. The final color $\mathbf{c}$ is calculated as $\mathbf{c}_s + \mathbf{c}_d$ [S1]. On NeRF Synthetic and Shiny Blender data sets, PSNR is 2 points higher than latest papers [S1].

#### NeRFLight
NeRFLight extends grid-based methods by dividing the scene into eight different regions to split NeRF into separate density and color fields, using a shared feature grid [S1]. It avoids increasing the number of features for seamless reconstruction while reducing stored feature values [S1]. The scene is divided into eight different regions, each corresponding to an MLP that outputs density and color [S1]. A shared feature grid is used, with features symmetrically placed at the center of the scene [S1].

## Background
The concept of decomposing radiance fields into geometric and appearance components originated from the limitations of standard NeRF in handling specularities and relighting. Early works like Ref-NeRF [S2] and DE-NeRF [S1] introduced explicit modeling of normals, roughness, and environment maps to address the "foggy" geometry and lack of physical plausibility in vanilla NeRF. NeRFLight [S1] further explored grid-based decompositions for efficient relighting. These methods collectively form the basis for modern differentiable rendering pipelines that require physically-based material editing.

## Sources
[S1] Methods and Strategies for Improving the Novel View Synthesis Quality of Neural Radiation Field, https://doi.org/10.1109/access.2024.3382997, fetched 2026-10-30
