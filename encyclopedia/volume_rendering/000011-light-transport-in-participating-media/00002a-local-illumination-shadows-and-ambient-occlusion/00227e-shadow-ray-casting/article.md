# Shadow Ray Casting


## Overview
Shadow Ray Casting evaluates visibility from a light source to a point in a volume by integrating extinction along a ray, solving the problem of accurate volumetric shadowing where standard surface techniques fail due to participating media. It differs from [Two-Pass Shadow Casting (Kajiya and Von Herzen)](2196-two-pass-shadow-casting-kajiya-and-von-herzen.md) by computing shadows locally per-shading-point rather than via a global pre-pass, and from [Shadow Mapping](2934-shadow-mapping.md) by integrating physical transmittance rather than comparing discrete depth buffers.

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

### [Ambient Occlusion via Shadow Rays](878-ambient-occlusion-via-shadow-rays.md)
Approximates lack of secondary light by evaluating geometric obscurance via hemisphere sampling, offering a balance between physical accuracy and speed.

### [Direct Illumination with Shadow Rays](2011-direct-illumination-with-shadow-rays.md)
Computes direct lighting by integrating extinction to light sources, differing from path tracing by avoiding recursive bounces.

### [Lookup Table Shadow Rendering](3588-lookup-table-shadow-rendering.md)
A fast, one-pass technique in VolPack that precomputes visibility into tables, avoiding per-ray integration costs.

### [Soft Shadows via Ray Tracing](5136-soft-shadows-via-ray-tracing.md)
Computes soft shadows by generating coherent packets of secondary rays toward area lights, leveraging coherence for performance.

### [Recursive Ray Tracing](5600-recursive-ray-tracing.md)
Extends basic casting to handle reflection/refraction via physical laws, distinguishing itself by modeling surface interactions beyond simple shadow tests.

### [Soft Shadows via Area Lights](7320-soft-shadows-via-area-lights.md)
Computes soft shadows by sampling light source geometry for natural penumbra, distinguished by direct area integration.

### [Shadow Mapping](10992-shadow-mapping.md)
Evaluates visibility via depth texture comparison, solving occlusion without expensive ray-marching transmittance calculations.

### Thin-Walled Glass with Airy Film
- [S1, unofficial] A rendering mode for thin-walled glass (window panes, bulbs, soap bubbles) that skips refraction offset and interior medium, using an optional Airy film as the wall itself.
- [S1, unofficial] Skip refraction offset and interior medium calculation.
- [S1, unofficial] Pass shadow rays through with colored transmittance.
- [S1, unofficial] Use the straight-through BSDF event to keep the previous vertex's MIS state, preventing double counting with other strategies.
- [S1, unofficial] Must ensure MIS consistency between the straight-through BSDF and other sampling strategies to avoid double counting.

### Shadow Ray Marching
- [S2, unofficial] Calculates sunlight contribution by casting a shadow ray from the surface towards the sun. If the shadow ray hits an object, the point is in shadow.
- [S2, unofficial] If SUN_ENABLED, calculate sunDir from sceneData. Calculate nDotL = max(dot(normal, sunDir), 0.0). If nDotL > 0, create shadowOrigin = position + normal * 0.001. March shadow ray using March(shadowOrigin, sunDir, 64).
- [S2, unofficial] If shadowRay.status != 2 (no hit), sunContribution = SUN_COLOR * nDotL * sceneData.sunIntensity.

### Mean Intensity Projection
- [S3, unofficial] A variation of ray casting that projects the mean intensity along the ray.
- [S3, unofficial] The output C(R) is determined by the mean scalar value B along the ray: $C(R) = B$.

### Halfway Vector Slicing for Shadows
- [S4] A shadow computation method that modifies the slice axis to be the direction halfway between the view and light directions, allowing the same slice to be rendered from both eye and light points of view.
- [S4] 1. Calculate the dot product of light and view directions.
2. If dot product > 0: Slice along vector halfway between light and view directions. Render front-to-back relative to observer.
3. If dot product < 0: Slice along vector halfway between light and inverted view directions. Render back-to-front relative to observer.
4. In both cases, render front-to-back relative to the light.
- [S4] Slice spacing maintenance: Care must be taken to maintain slice spacing along view and light directions when light or eye positions change.
- [S4] Differs from previous hardware shadow work [2] by using an off-screen render buffer instead of a volumetric shadow map, and by modifying the slice axis to allow shared slicing.

### Dynamic Shadow Region Approximation
- [S5, unofficial] A technique by Fernandez et al. that subdivides the scene and keeps a list of light sources that will be fully occluded, fully visible, or partially visible to each region of space, using an approximation based on shadow rays to build the data structure at run time for dynamic scenes.

## Background
Shadow ray casting in volume rendering evolved from surface-based shadowing techniques to handle participating media. Early work by Kajiya and Von Herzen introduced two-pass methods for global illumination in volumes [2196-two-pass-shadow-casting-kajiya-and-von-herzen.md]. Local shadow casting became prevalent with the rise of real-time volume rendering, where per-pixel shadow rays provided accurate attenuation without the memory overhead of global shadow maps. The technique is widely used in medical visualization, atmospheric effects, and real-time game engines (e.g., VolPack, WebGPU voxel raymarchers) where physical accuracy of light transport through fog, smoke, or tissue is required.

## Sources
[S1] Add volumetric rendering, advanced materials, and imager effects, https://github.com/MihrimatriX/photon-engine/pull/3, fetched 2026-10-28 (unofficial)
[S2] jamestkiernan/webgpu-voxel-raymarching, https://github.com/jamestkiernan/webgpu-voxel-raymarching, fetched 2026-10-24 (unofficial)
[S3] Direct Volume Rendering Direct Volume Rendering. What is volume rendering?. Accumulate information along 1 dimension line through volume. Volume rendering vs. isosurfaces. No intermediate geometry No , https://www.slideserve.com/lenci/direct-volume-rendering, fetched 2026-10-25 (unofficial)
[S4] Multidimensional Transfer Functions for Interactive Volume ..., https://www.academia.edu/5403340/Multidimensional_Transfer_Functions_for_Interactive_Volume_Rendering, fetched 2026-11-01
[S5] Massive Data Management for Distributed Volume Visualization, http://hdl.handle.net/1951/47671, fetched 2026-10-26 (unofficial)
