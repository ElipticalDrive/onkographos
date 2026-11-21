# Relighting and Shading

Also called: Irradiance Fields, Shading Replacement

## Overview
Relighting and Shading techniques address the problem of lighting inconsistency when compositing objects into neural radiance field (NeRF) scenes or synthesizing novel views under varying illumination conditions. Unlike standard radiance field methods that treat radiance as a static view-dependent signal, this family explicitly models lighting interactions, inter-reflections, or shading correlations to enable physically plausible relighting and HDR synthesis. It sets itself apart from direct relighting methods by either replacing object shading based on scene correlations [Shading Replacement for Relighting](6346-shading-replacement-for-relighting.md) or integrating surface rendering principles into the volume rendering framework to model irradiance [Pano-NeRF](15013-pano-nerf.md).

## Shared principles
The core mechanism involves decoupling or explicitly modeling the illumination component of the radiance field. In standard NeRF, radiance $c(x, \mathbf{d})$ is a direct output of the network. In this family, the representation is extended to account for how light interacts with geometry.

1.  **Irradiance-Radiance Coupling**: Radiance is treated as both a signal conveying lighting information and a light source illuminating other pixels. This increases observation counts for geometry and leverages irradiance-radiance attenuation for High Dynamic Range (HDR) reconstruction [S1].
2.  **Surface-Volume Integration**: Methods may integrate surface rendering principles (BRDF, normals) into the volumetric framework. For instance, outgoing radiance $C_i(x)$ is computed by integrating irradiance from incoming light directions $\omega_i$ over a sphere $\Omega$ centered at $x$ [Pano-NeRF](15013-pano-nerf.md):
    $$
    C_i(x) = \int_{\omega_i \in \Omega} f_r(x, \omega_o, \omega_i) \cdot \max(\omega_i \cdot \mathbf{n}, 0) \cdot c_i(x, \omega_i) \, d\omega_i
    $$
    where $f_r$ is the BRDF, $\mathbf{n}$ is the surface normal, and $c_i(x, \omega_i)$ is the incoming radiance computed via volume rendering [Pano-NeRF](15013-pano-nerf.md).
3.  **Shading Correlation**: In object insertion tasks, shading is inferred by correlating surface normals and shading values between the inserted object and the existing scene, avoiding explicit light source estimation [Shading Replacement for Relighting](6346-shading-replacement-for-relighting.md).

## Subtopics

### [Shading Replacement for Relighting](6346-shading-replacement-for-relighting.md)
Shading Replacement for Relighting is a technique for object insertion that avoids inferring scene illumination by replacing the inserted object's shading with shading from existing scene points [Shading Replacement for Relighting](6346-shading-replacement-for-relighting.md). It solves the problem of lighting inconsistency when compositing objects into NeRF scenes, where naive composition leads to unrealistic results due to varying illumination [Shading Replacement for Relighting](6346-shading-replacement-for-relighting.md). It sets itself apart from direct relighting methods by not requiring explicit light source estimation, instead relying on the correlation between surface normals and shading in Lambertian scenes [Shading Replacement for Relighting](6346-shading-replacement-for-relighting.md). Choose this method when inserting objects into a scene where the lighting environment is complex and unknown, but surface normals are available or estimable.

### [Pano-NeRF](15013-pano-nerf.md)
Pano-NeRF is a method for geometry recovery and High Dynamic Range (HDR) novel view synthesis from sparse Low Dynamic Range (LDR) panoramic images [Pano-NeRF](15013-pano-nerf.md). It models the scene using irradiance fields, which treat radiance as both a signal conveying lighting information and a light source illuminating other pixels, thereby increasing observation counts for geometry and leveraging irradiance-radiance attenuation for HDR reconstruction [Pano-NeRF](15013-pano-nerf.md). It differs from standard radiance field methods by integrating surface rendering principles into the volume rendering framework to explicitly model inter-reflection and lighting interactions [Pano-NeRF](15013-pano-nerf.md). Choose this method when working with sparse panoramic inputs and requiring HDR output or improved geometry recovery through inter-reflection modeling.

### CI-NeRF
CI-NeRF utilizes NeRF for volumetric rendering to deduce essential scene characteristics like diffuse albedo, surface normals, shadows, and lighting settings [S1]. It employs a CNN for forecasting interpretable and modifiable lighting parameters from an individual image [S1]. This allows for the physical interpretation of outdoor scene relighting through a unified image inference process, enhancing both the quality and realism of the relighting [S1].

## Background
The concept of integrating lighting and shading into neural radiance fields has evolved to address limitations in standard NeRF regarding physical plausibility and dynamic lighting. CI-NeRF (2023) was introduced to allow physical interpretation of outdoor scene relighting through a unified image inference process, surpassing current methods in inherent image breakdown and real-world landmark scene relighting [S1]. Pano-NeRF and Shading Replacement for Relighting further extend this by explicitly handling irradiance fields and shading correlations, respectively, to solve issues in sparse view synthesis and object compositing [S1]. These methods build on the foundational NeRF architecture but modify the query and rendering process to include lighting-aware representations.

## Sources
[S1] Neural Radiance Field-based Visual Rendering: a Comprehensive Review, https://ieeexplore.ieee.org/abstract/document/11455380, fetched 2026-10-30
