# Texture and Appearance Representation


## Overview
Texture and Appearance Representation covers methods that extend 3D Gaussian primitives with appearance attributes to determine color during rendering [G]. It solves the limitation of static color storage by enabling high-frequency details or view-dependent effects without excessively increasing the number of primitives [S1][S2, unofficial]. It sets itself apart from sibling methods like [GaussianSet Data Structure](2074-gaussianset-data-structure.md) by focusing on the semantic definition of color rather than memory layout, and from [Primitive Geometry and Distribution](6228-primitive-geometry-and-distribution.md) by addressing radiance rather than shape [G].

## Shared principles
All child methods extend the standard 3D Gaussian primitive $i$ with additional appearance attributes $\mathbf{a}_i$ beyond the geometric parameters and opacity $\alpha_i$ [G]. The rendering pipeline retains the front-to-back alpha-blending equation, but the color contribution $c_i$ of each Gaussian is no longer a static vector. Instead, $c_i$ is computed by evaluating an appearance function that queries $\mathbf{a}_i$ using the viewing direction $\mathbf{v}$ for view-dependent methods [S1][S2, unofficial], or sampled from a 2D texture using local UV coordinates $\mathbf{u}_i$ for textured methods [S2, unofficial]. The final pixel color is calculated as:

$$
C(p) = \sum_{i \in \text{sorted}} c_i(\mathbf{v}, \mathbf{a}_i) \alpha_i \prod_{j=1}^{i-1} (1 - \alpha_j)
$$

[G]. This approach involves a trade-off between storage bandwidth and computational cost during decoding [S1][S2, unofficial].

## Subtopics

### [Textured Gaussians](8061-textured-gaussians.md)
Textured Gaussians replace the standard spherical harmonic (SH) coefficients used for view-dependent color in 3D Gaussian Splatting with explicit 2D textures mapped onto each primitive. This method captures high-frequency appearance details without requiring an increase in the number of primitives. It sets itself apart from sibling methods like View-Dependent Appearance Representations by utilizing spatial UV sampling rather than directional basis functions. When to choose it: Not given in the sources. [Textured Gaussians](8061-textured-gaussians.md)

### [View-Dependent Appearance Representations](14602-view-dependent-appearance-representations.md)
This family of methods extends the standard 3D Gaussian primitive with appearance attributes $\mathbf{a}_i$ to model view-dependent color $c_i(\mathbf{v}, \mathbf{a}_i)$, where $\mathbf{v}$ is the viewing direction. Unlike static color storage, these representations decode color dynamically based on the viewing angle, enabling the modeling of specular highlights, anisotropy, and complex lighting effects. The rendering pipeline retains front-to-back alpha-blending, but the color contribution of each Gaussian is computed via an appearance function that queries $\mathbf{a}_i$ using the viewing direction $\mathbf{v}$. This approach trades off storage bandwidth and decoding computational cost. When to choose it: Not given in the sources. [View-Dependent Appearance Representations](14602-view-dependent-appearance-representations.md)

#### View-Dependent Gaussian Splatting
Models 3DGS to represent 3D shapes and predicts attributes like view-dependent color and opacity using a NeRF-like neural network, replacing the Spherical Harmonic (SH) coefficients used in original 3DGS [S1].

#### Neural BRDF for 3D Gaussian Splatting
Simplifies the BRDF to a neural shader that predicts specular and diffuse components directly from the view direction and a learnable material feature [S2, unofficial].

#### Neural Texture Splatting
Encodes textures as small neural networks evaluated at Gaussian coordinates, which can be deformed to model dynamic appearances and fine geometric displacements [S2, unofficial].

## Background
The standard 3D Gaussian Splatting (3DGS) method originally used Spherical Harmonics (SH) to represent view-dependent color [S1]. SH has limitations in capturing high-frequency details and complex lighting effects [S1]. Recent advances introduced explicit 2D textures and neural appearance models to overcome these limitations [S1][S2, unofficial]. View-Dependent Gaussian Splatting (VDGS) proposed using NeRF-like networks to predict view-dependent attributes [S1]. Neural Texture Splatting extended this by encoding textures as neural networks [S2, unofficial]. These methods aim to improve visual fidelity while maintaining the real-time performance of Gaussian splatting [S1][S2, unofficial].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
