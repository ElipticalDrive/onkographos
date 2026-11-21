# Appearance and Lighting Decomposition

Also called: Decomposition-based NeRF, Relighting-aware NeRF

## Overview
Appearance and Lighting Decomposition methods extend standard Neural Radiance Fields by explicitly separating scene geometry, material properties, and lighting conditions. This separation enables high-fidelity relighting of high-frequency environments and independent editing of appearance [S1]. Unlike sibling methods that focus on efficient sampling or specific architectural priors (e.g., [AdaNeRF](6195-adanerf.md), [SinNeRF](6199-sinnerf.md)), these models restructure the radiance field to output physical parameters (such as albedo, roughness, or specular coefficients) rather than raw color, allowing for physically plausible synthesis under varying illumination [S1].

## Shared principles
The core mechanism involves decomposing the volumetric rendering process into distinct physical components. While standard NeRF maps $(\mathbf{x}, \mathbf{d}) \to (\sigma, \mathbf{c})$, decomposition methods typically map inputs to intermediate physical attributes.

1.  **Geometric Representation:** Geometry is often represented via a Signed Distance Field (SDF) or a density field that is regularized to be surface-concentrated. This allows for the stable estimation of surface normals $\mathbf{n}$, which are critical for reflection-based lighting models [S1].
2.  **Material Decomposition:** The appearance function is split into diffuse and specular components. Diffuse color is often derived from albedo and environment map integration, while specular color is derived from a specular lighting function and material properties like roughness and specular tint [S1].
3.  **Lighting Integration:** The final color $\mathbf{c}$ is computed by combining the diffuse and specular contributions. This may involve integrating diffuse reflectance with an environment map and multiplying specular lighting by specular hue [S1].

## Subtopics

### [Anisotropic Neural Representation Learning](9475-anisotropic-neural-representation-learning.md)
[Anisotropic Neural Representation Learning](9475-anisotropic-neural-representation-learning.md) models volumetric density and latent features as view-dependent functions using spherical harmonics (SH) to eliminate directional ambiguity. It replaces isotropic point-sampled features with SH-guided anisotropic features, allowing the network to capture directional variations in geometry and appearance without requiring shape-based sampling parameterization. Unlike Ref-NeRF, which structures appearance for glossy reflections, ANRL modifies the MLP to predict view-dependent SH coefficients directly, offering a plug-and-play solution that improves rendering quality in various NeRF-based frameworks.

### [Ref-NeRF](10843-ref-nerf.md)
[Ref-NeRF](10843-ref-nerf.md) addresses the inability of standard NeRF to reproduce glossy specular reflections by reparameterizing view-dependent outgoing radiance as reflected radiance. It structures the appearance function using spatially-varying scene properties (normals, diffuse/specular splits) to ensure the underlying radiance function remains smooth and interpolatable. Unlike ANRL, which focuses on directional anisotropy via SH, Ref-NeRF relies on normal estimation and reflection vectors to decouple material properties from lighting.

### DE-NeRF
DE-NeRF decomposes lighting, geometry, and appearance to allow relighting of high-frequency environments and appearance editing [S1]. It trains a signed distance field (SDF) function using 2D images to reconstruct a display grid as the input scene for the MLP [S1]. The scene is decomposed into geometric features and appearance features (diffuse reflection, roughness, specular tint). Geometric features, appearance features, and distance $h$ (from sampling point to grid) are input into four corresponding MLPs to predict diffuse albedo, roughness value, and specular tint [S1]. Diffuse reflectance is integrated with the environment map to get diffuse color $c_d$. A specular lighting MLP $F_s$ is trained to get specular lighting $c_l$, which is multiplied with specular hue to get specular color $c_s$. The final color is $c = c_s + c_d$ [S1]. On NeRF Synthetic and Shiny Blender data sets, PSNR is 2 points higher than latest papers [S1].

### NeRFLight
NeRFLight extends grid-based methods by dividing the scene into eight different regions to split NeRF into separate density and color fields, using a shared feature grid [S1]. It avoids increasing the number of features for seamless reconstruction while reducing stored feature values [S1]. The scene is divided into eight regions, each corresponding to an MLP that outputs density and color. A shared feature grid is used, with features symmetrically placed at the center of the scene [S1].

## Background
The concept of decomposing radiance fields for relighting and material editing emerged as NeRF applications expanded beyond novel view synthesis to interactive editing and physical simulation. Early works like DE-NeRF [25] and NeRFLight [27] (cited in [S1]) introduced architectural changes to separate geometry from appearance. DE-NeRF specifically targets high-frequency environment relighting by integrating environment maps with predicted diffuse albedo [S1]. NeRFLight utilizes a grid-based approach with region-specific MLPs to maintain seamless reconstruction while reducing storage overhead [S1]. These methods laid the groundwork for subsequent approaches like Ref-NeRF and ANRL, which further refined the handling of view-dependent effects and anisotropy.

## Sources
[S1] Methods and Strategies for Improving the Novel View Synthesis Quality of Neural Radiation Field, https://doi.org/10.1109/access.2024.3382997, fetched 2026-10-30
