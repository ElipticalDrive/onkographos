# Neural Relighting

Also called: Neural Inverse Rendering, Neural Relighting Fields

## Overview
Neural Relighting methods solve the problem of synthesizing novel views of a scene under arbitrary lighting conditions by learning implicit neural representations of scene appearance and light transport [S1, S2]. Unlike traditional volumetric methods that explicitly sample density and scattering coefficients, these methods decouple geometry, material reflectance, and visibility into distinct neural components, enabling efficient relighting without environment maps or explicit shadow ray casting [S1, S2]. They are distinguished from sibling methods like [Direct Lighting and Shading](3953-direct-lighting-and-shading.md) by replacing analytical visibility tests with learned neural predictors, and from [Light Propagation Volumes](4332-light-propagation-volumes.md) by their ability to handle complex, view-dependent shadows and non-Lambertian materials through end-to-end differentiable training [S1, S2].

## Shared principles
All methods in this family share the core mechanism of evaluating visibility or illumination at a specific point $\mathbf{p}$ within or on the boundary of a volume, based on a light source direction $\mathbf{l}$ or environment [G].

1.  **Visibility/Transmittance Evaluation:** The fundamental operation is determining how much light reaches $\mathbf{p}$ from a source [G]. This involves integrating the extinction coefficient $\sigma_t(\mathbf{x})$ along a path from the source to $\mathbf{p}$ (or vice versa) [G]. The transmittance $T$ is calculated as:
    $$ T = \exp\left(-\int_{\mathbf{p}_{source}}^{\mathbf{p}} \sigma_t(\mathbf{x}) \, d\mathbf{x}\right) $$
    In discrete implementations, this integral is approximated by summing $\sigma_t \cdot \Delta s$ over samples along the ray [G].

2.  **Local Shading Models:** For surface-like appearance in volumes, normals are approximated from the volume gradient $\mathbf{g} = \nabla V$ [G]. Usually, a simple absorption and emission model is used because it is relatively cheap to evaluate and gives good enough results [S1].

3.  **Shadowing Approximation:** Shadows are computed by testing occlusion along the direction to the light source [G].

4.  **Ambient Occlusion:** Indirect illumination is approximated by integrating over the hemisphere or using simplified models (e.g., upward ray for sky light) [G]. The ambient contribution $L_{ambient}$ is often modulated by a shadowing factor based on local density:
    $$ L_{ambient} = L_{sky} \cdot \exp(-D_{up}) \cdot AO_{factor} $$
    where $D_{up}$ is the integrated density along an upward ray [G].

## Subtopics

### [Sun-aligned Relighting TensoRF](12275-sun-aligned-relighting-tensorf.md)
Sun-aligned Relighting TensoRF (SR-TensoRF) is a method for outdoor scene relighting and novel view synthesis that eliminates the need for environment maps by using the sun direction as a direct input for shadow generation. It builds on [Local illumination, shadows and ambient occlusion](34-local-illumination-shadows-and-ambient-occlusion.md) by decomposing the synthesis process into albedo, shadow, and tone components, leveraging TensoRF's efficiency. Unlike NeRF-OSR, which relies on environment maps, SR-TensoRF uses sun direction input, simplifying inference and allowing dynamic shadow generation based on time and location.

### [Neural Reflectance and Visibility Fields](22509-neural-reflectance-and-visibility-fields.md)
Neural Reflectance and Visibility Fields (NRF) is a method for novel view synthesis and relighting that decouples scene geometry, material reflectance, and light transport into distinct neural representations. It solves the problem of recovering a relighting model from images with pronounced shadows and self-occlusion by training a Multi-Layer Perceptron (MLP) to approximate light source visibility for any 3D position and incident light direction. Unlike standard NeRF which renders radiance directly, NRF separates the visibility component, enabling efficient recovery of scene appearance under varying lighting conditions.

### RelightNet Baseline
- [S1] A single-image based relighting framework used to produce intrinsic decomposition results, including shading and albedo.

### Neural Lighting Estimation for AR
- [S2] A technique for AR that infers ambient lighting from monocular images in real-time to render synthetic content with photometric accuracy relative to the physical world.
- [S2] Infer ambient lighting from monocular images in real-time, typically by predicting Spherical Harmonics (SH) coefficients.
- [S2] Apply the estimated ambient lighting to render synthetic content.
- [S2] Render cast shadows and specular highlights of virtual objects with perfect photometric accuracy relative to the physical world.
- [S2] Lighting Representation: Spherical Harmonics (SH) coefficients.
- [S2] Referenced in [12, 16].

### Neural Direct-Illumination Renderer
- [S3, unofficial] Renderer that renders direct illumination images of any geometric shape with opaque materials under distant lighting.
- [S3, unofficial] 1. Input screen-space buffer (material, geometric shape, and lighting) into CNN network. 2. CNN learns mapping from pixel buffer to rendered image. 3. NDR predicts shadow map. 4. Combine shadow map with irradiance map to render image.
- [S3, unofficial] Screen-space buffer inputs: Material, Geometric shape, Lighting. Outputs: Shadow map, Irradiance map, Rendered image.

### Multi-Light Reconstruction Benefits
- [S4] Multi-light input enables more accurate reconstructions of lighting compared to single-light input.
- [S4] Single-light novel view synthesis results are slightly better than multi-light results because the single-light model is specifically trained on and can overfit to the same single lighting.
- [S4] Multi-light models achieve better reconstruction and rendering quality under novel lighting conditions compared to single-light models.
- [S4] Multi-light reconstruction recovers more details in normal maps and better shading- and artifact-free albedo maps.
- [S4] Multi-light settings help solve color ambiguity between environmental lighting and object materials.

### General Multi-Light Capture
- [S4] Create three lighting conditions by rendering objects with three unrelated environment maps.
- [S4] Optimize the three lighting conditions separately in the later training process.

## Background
Neural relighting emerged from the intersection of inverse rendering and neural radiance fields (NeRF). Early work focused on decomposing single images into intrinsic components like albedo and shading [S1]. The field advanced with methods like TensoIR, which demonstrated that multi-light capture significantly improves the accuracy of reconstructed environment maps and resolves color ambiguity between materials and lighting [S4]. Subsequent developments, such as SR-TensoRF and Neural Reflectance and Visibility Fields, moved towards implicit representations that separate geometry, material, and visibility, allowing for dynamic relighting without explicit environment maps [S1, S2]. These methods are widely used in augmented reality for photorealistic virtual object integration [S2] and in large-scale urban scene reconstruction [S1].

## Sources
[S1] UrbanIR: Large-Scale Urban Scene Inverse Rendering from a Single Video, https://doi.org/10.1109/3dv66043.2025.00052, fetched 2026-10-30
[S2] Review of Key Technologies and Application Scenarios of Neural Rendering, https://doi.org/10.54254/2755-2721/2026.36116, fetched 2026-10-26
[S3] Neural Rendering and Its Hardware Acceleration: A Review, https://doi.org/10.48550/arxiv.2402.00028, fetched 2026-10-26 (unofficial)
[S4] TensoIR: Tensorial Inverse Rendering, https://doi.org/10.1109/cvpr52729.2023.00024, fetched 2026-10-26
