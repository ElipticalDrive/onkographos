# Volumetric Shadow Sampling and Ray Casting


## Overview
This family encompasses methods for computing shadows and local illumination within participating media by evaluating visibility along rays cast from shading points toward light sources. It solves the problem of determining how much light is attenuated by the volume density between a point and the light, enabling realistic shadowing in fog, smoke, or particle systems. It differs from global illumination methods by focusing exclusively on direct visibility attenuation (often via stochastic sampling or slice-based accumulation) rather than simulating full light transport bounces, and from hard shadow techniques by approximating penumbrae through density-weighted sampling or spatial sorting.

## Shared principles
All methods in this family share the core mechanism of evaluating visibility or illumination at a specific point $\mathbf{p}$ within or on the boundary of a volume, based on a light source direction $\mathbf{l}$ or environment [G].

1.  **Visibility/Transmittance Evaluation:** The fundamental operation is determining how much light reaches $\mathbf{p}$ from a source [G]. This involves integrating the extinction coefficient $\sigma_t(\mathbf{x})$ along a path from the source to $\mathbf{p}$ (or vice versa) [G]. The transmittance $T$ is calculated as:
    $$ T = \exp\left(-\int_{\mathbf{p}_{source}}^{\mathbf{p}} \sigma_t(\mathbf{x}) \, d\mathbf{x}\right) $$
    In discrete implementations, this integral is approximated by summing $\sigma_t \cdot \Delta s$ over samples along the ray [G].

2.  **Local Shading Models:** For surface-like appearance in volumes, normals are approximated from the volume gradient $\mathbf{g} = \nabla V$ [G]. Usually, a simple absorption and emission model is used because it is relatively cheap to evaluate and gives good enough results [S1, unofficial].

3.  **Shadowing Approximation:** Shadows are computed by testing occlusion along the direction to the light source [G].

4.  **Ambient Occlusion:** Indirect illumination is approximated by integrating over the hemisphere or using simplified models (e.g., upward ray for sky light) [G]. The ambient contribution $L_{ambient}$ is often modulated by a shadowing factor based on local density:
    $$ L_{ambient} = L_{sky} \cdot \exp(-D_{up}) \cdot AO_{factor} $$
    where $D_{up}$ is the integrated density along an upward ray [G].

## Subtopics

### [Volumetric Soft Shadowing](4507-volumetric-soft-shadowing.md)
Estimates soft shadows by sampling random directions towards the light, weighting by local density and distance. It differs from hard shadow techniques by approximating the penumbra through stochastic sampling, and from global illumination methods by focusing solely on direct visibility attenuation without simulating indirect bounces.

### [Half-Angle Slice Rendering](9979-half-angle-slice-rendering.md)
Adds volumetric shadowing to particle systems by rendering particles as slices perpendicular to a half-angle vector (between view and light directions), allowing simultaneous accumulation of shadowing and blending using a single 2D shadow buffer. It solves the problem of efficiently computing soft volumetric shadows for large numbers of particles without the memory overhead of 3D voxelization. Unlike volume rendering approaches that require voxelization and 3D textures, this method uses only a single 2D shadow buffer and particle sorting.

### Half-Angle Shadow Opacity Sampling
**Step**
- [S1, unofficial] In the render vertex shader, compute light-space texture coordinates for opacity lookup: `lightTextureCoordinates = (uLightViewProjection * vec4(position, 1.0)) * 0.5 + 0.5`.
- [S1, unofficial] Sample opacity from `uOpacityTexture` using the light-space coordinates: `vOpacity = texture(uOpacityTexture, lightTextureCoordinates).a`.
- [S1, unofficial] In the floor fragment shader, compute light-space coordinates for the floor position and sample opacity. If coordinates are outside [0,1], set opacity to 0.

**Formula**
- [S1, unofficial] Final floor shadow color: `FragColor = vec4(0.0, 0.0, 0.0, opacity * 0.5)`.

## Background
Origin: The specific implementations described, particularly Half-Angle Slice Rendering for particle systems, originate from the GitHub repository `zk8085454/VolumeParticleShadows` [S1, unofficial]. This technique addresses the performance bottleneck of 3D voxelization for large-scale particle effects by projecting shadows onto 2D buffers aligned with the half-angle vector between view and light directions.

## Sources
[S1] GitHub - zk8085454/VolumeParticleShadows: Half angle slicing technique ..., https://github.com/zk8085454/VolumeParticleShadows/, fetched 2026-10-28 (unofficial)
