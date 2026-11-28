# BRDF and Relighting Extensions

Also called: Relightable 3D Gaussians, PBR Gaussian Splatting, Inverse Rendering with Gaussians

## Overview
BRDF and Relighting Extensions augment standard 3D Gaussian Splatting (3DGS) by replacing the baked, view-dependent color attribute with physically based rendering (PBR) parameters, enabling dynamic relighting, shadow casting, and material decomposition . Unlike standard 3DGS which captures lighting in spherical harmonics, these methods decouple geometry from material properties, allowing the scene to be illuminated by arbitrary light sources or environment maps . They distinguish themselves from siblings like [3DGS-DR](1471-3dgs-dr.md) by explicitly storing BRDF parameters (albedo, roughness, normal) per Gaussian, facilitating direct lighting reconstruction via environment maps and indirect lighting via spherical harmonics .

## Shared principles
All methods in this family share the core mechanism of projecting 3D primitives into 2D screen space and accumulating color based on depth order or spatial locality [G].

1.  **Projection:** Primitives are transformed from world space to camera space [G]. The 3D covariance matrix $\Sigma$ is transformed using the view matrix $W$ and the Jacobian $J$ of the perspective projection approximation: $\Sigma_c = J W \Sigma W^T J^T$ [G]. The 2D covariance $\Sigma_{2D}$ is extracted as the upper-left $2\times2$ block [G].
2.  **Rasterization:** Primitives are assigned to screen tiles [G]. Visibility is determined by bounding boxes or projected extents [G].
3.  **Compositing:** Color is accumulated using front-to-back alpha blending or order-independent techniques [G]. The final intensity for a pixel is $I_f = \sum_{i=1}^N c_i \tilde{\sigma}_i \tilde{\mathcal{G}}_i(\mathbf{p}) \prod_{j=1}^{i-1} (1 - \tilde{\sigma}_j \tilde{\mathcal{G}}_j(\mathbf{p}))$, where $c_i$ is color, $\tilde{\sigma}_i$ is opacity or density, and $\tilde{\mathcal{G}}_i(\mathbf{p})$ is the 2D Gaussian value at pixel $\mathbf{p}$ [G].
4.  **Sorting:** Most methods require sorting primitives by depth within tiles to ensure correct front-to-back compositing, although some variants use spatial hashing or cell-level sorting to approximate this or enable order-independent transparency [G].

## Subtopics

### [3D Gaussian Splatting with BRDF Embedding](1174-3d-gaussian-splatting-with-brdf-embedding.md)
This method extends standard 3DGS by embedding physically based rendering parameters into each Gaussian primitive, enabling per-point BRDF evaluation and relighting. It distinguishes itself from siblings like [3DGS-DR](1471-3dgs-dr.md) and [Relightable 3D Gaussian](6001-relightable-3d-gaussian.md) by explicitly storing incident lighting and BRDF parameters per Gaussian, facilitating direct lighting reconstruction via environment maps and indirect lighting via spherical harmonics.

### [3D Gaussian Splatting with Reflection and Relighting](5873-3d-gaussian-splatting-with-reflection-and-relighting.md)
This method extends standard 3DGS by incorporating classical reflection and shading concepts to address limitations with highly reflective surfaces and enable dynamic relighting. Unlike standard 3DGS, which bakes lighting into spherical harmonics via an emission-absorption model, this approach allows for physically based rendering (PBR) by augmenting Gaussian kernels with environmental information. It leverages the natural normal estimation provided by the shortest axis of the Gaussian kernel to achieve high-quality rendering of reflective objects with training efficiency.

### [Relightable 3D Gaussian](6001-relightable-3d-gaussian.md)
Relightable 3D Gaussian extends 3D Gaussian Splatting by assigning per-Gaussian BRDF parameters and precomputed visibility to enable dynamic relighting and shadow casting. It distinguishes itself from standard 3D Gaussian Splatting by evaluating material responses to arbitrary light sources rather than relying solely on captured environment maps. Builds on [BRDF and Relighting Extensions](6227-brdf-and-relighting-extensions.md).

### [Inverse Rendering with Gaussians (GS-IR / GIR)](6153-inverse-rendering-with-gaussians-gs-ir-gir.md)
Inverse Rendering with Gaussians (GS-IR / GIR) adapts 3D Gaussian Splatting to recover physically accurate material and lighting properties from images by optimizing differentiable shading models. It distinguishes itself from sibling methods by employing a split-sum approximation to handle indirect lighting and specular reflections.

### [GaussianShader Material Model](8013-gaussianshader-material-model.md)
GaussianShader enhances 3D Gaussian Splatting (3DGS) by incorporating trainable material parameters—diffuse color, specular intensity, roughness, and surface normal—along with a global environment map to achieve physically-based rendering (PBR). It distinguishes itself from standard 3DGS by disentangling geometry from material properties, allowing for accurate specular reflections and relighting capabilities.

### [BRDF Decoder Strategies](8147-brdf-decoder-strategies.md)
BRDF Decoder Strategies define how 3D Gaussian Splatting primitives encode and reconstruct material properties (such as albedo, roughness, and metalness) or direct light reflection weights during optimization. While standard 3DGS renders color directly, these strategies decouple geometry and appearance to enable physically-based rendering (PBR) and relighting. They differ from sibling methods by focusing specifically on the *decoding* mechanism—whether through explicit physical parameter regression or implicit neural mapping—rather than the rasterization or sorting pipeline itself.

## Background
The extension of 3D Gaussian Splatting to support PBR and relighting emerged as a response to the inability of standard 3DGS to separate geometry from lighting . Standard 3DGS encodes appearance into view-dependent spherical harmonics, making it difficult to relight scenes or recover material properties . Early works such as [3D Gaussian Splatting with BRDF Embedding](1174-3d-gaussian-splatting-with-brdf-embedding.md) and [GaussianShader](8013-gaussianshader-material-model.md) introduced per-Gaussian material attributes . Subsequent methods like [Inverse Rendering with Gaussians](6153-inverse-rendering-with-gaussians-gs-ir-gir.md) and [Relightable 3D Gaussian](6001-relightable-3d-gaussian.md) focused on differentiable optimization of these attributes from multi-view images . These techniques are primarily used in computer graphics for virtual production, digital twins, and inverse rendering applications where dynamic lighting and material analysis are required .

## Sources
[S1] 3D Gaussian Splatting with BRDF Embedding, https://github.com/ashawkey/3dgs-brdf, 2024-05-20
[S2] GaussianShader, https://github.com/buaacyw/GaussianShader, 2024-05-20
