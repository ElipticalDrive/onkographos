# Shadow Ray Casting


## Overview
Shadow Ray Casting is a family of local illumination techniques that determine visibility between a point in a volume and a light source by tracing rays through the participating medium. It solves the problem of self-shadowing and occlusion in volumetric data, which standard direct illumination models often ignore, resulting in flat or physically incorrect appearances. Unlike [Variance Shadow Mapping for NeRFs](3561-variance-shadow-mapping-for-nerfs.md) which uses probabilistic depth buffers, or [Lookup Table Shadow Rendering](3588-lookup-table-shadow-rendering.md) which relies on precomputed spatial indices, this method directly samples the medium along the path to light, offering a balance between the physical accuracy of [Recursive Ray Tracing](5600-recursive-ray-tracing.md) and the speed of screen-space approximations [S3, unofficial].

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
Ambient Occlusion via Shadow Rays approximates the lack of secondary (reflected) light rays by evaluating the geometric obscurance of a point $\mathbf{p}$ within or on the boundary of a volume. It solves the problem of flat, unshaded volume appearances by darkening crevices and areas shielded from the environment, providing a view-independent shading term that can be combined with arbitrary direct illumination. Unlike [Variance Shadow Mapping for NeRFs](3561-variance-shadow-mapping-for-nerfs.md) which uses probabilistic depth buffers, or [Lookup Table Shadow Rendering](3588-lookup-table-shadow-rendering.md) which relies on precomputed spatial indices, this method directly samples the hemisphere around $\mathbf{p}$ using shadow rays to determine visibility, offering a balance between the physical accuracy of [Recursive Ray Tracing](5600-recursive-ray-tracing.md) and the speed of screen-space approximations.

### [Shadow Receiving for Volumes](956-shadow-receiving-for-volumes.md)
Shadow Receiving for Volumes is a technique that allows volumetric data to receive shadows cast by polygonal geometry, enhancing depth perception and realism without requiring the volume to be rendered twice. Unlike standard shadow pipelines that render shadow-casting objects separately, this method integrates shadow occlusion checks directly into the volume ray traversal process, significantly reducing computational overhead.

### [Variance Shadow Mapping for NeRFs](3561-variance-shadow-mapping-for-nerfs.md)
Variance Shadow Mapping for NeRFs is a shadow rendering technique that adapts the variance shadow mapping algorithm from traditional computer graphics to neural radiance fields. It solves the problem of computing soft shadows in NeRFs without performing expensive cumulative transmittance integrations along shadow rays. By comparing depth inconsistencies between the camera view and a virtual light view, it produces continuous visibility values in the range $[0, 1]$, avoiding the hard-edged artifacts typical of naive shadow mapping.

### [Lookup Table Shadow Rendering](3588-lookup-table-shadow-rendering.md)
Lookup Table Shadow Rendering is a fast, one-pass shadowing technique implemented within the VolPack volume rendering library. It solves the problem of computing shadows for participating media by precomputing visibility information into lookup tables, avoiding the per-ray integration costs of standard shadow casting. Unlike [Shadow Ray Casting](2796-shadow-ray-casting.md), which evaluates transmittance along rays during rendering, this method accelerates shadow determination by mapping density or depth values to pre-calculated shadow factors.

### [Soft Shadows via Ray Tracing](5136-soft-shadows-via-ray-tracing.md)
This method computes soft shadows in participating media by generating coherent packets of secondary rays from primary ray hit points toward an area light source. It distinguishes itself from hard shadow techniques by sampling a spatial distribution of directions to approximate the penumbra, leveraging the coherence of these secondary rays to maintain interactive performance.

### [Recursive Ray Tracing](5600-recursive-ray-tracing.md)
Recursive Ray Tracing extends basic ray casting by handling diffuse reflection, specular reflection, and transmission (refraction) through the recursive tracing of rays upon intersection with objects. It solves the problem of local illumination and shadowing in participating media by evaluating visibility to light sources and simulating surface interactions that influence the final pixel color. It is distinguished from its sibling [Shadow Ray Casting](2796-shadow-ray-casting.md) by explicitly modeling reflection and refraction via physical laws (Snell's law, Fresnel equation) rather than relying solely on shadow tests and ambient approximations.

### [Soft Shadows via Area Lights](7320-soft-shadows-via-area-lights.md)
This method computes soft shadows by modeling light sources with spatial extent, allowing points in space to be partially occluded rather than fully lit or fully in shade. Unlike point source approximations which cast sharp shadows, this approach samples the light source area to generate natural penumbra effects. It is distinguished from sibling methods like [Variance Shadow Mapping for NeRFs](3561-variance-shadow-mapping-for-nerfs.md) by its direct sampling of the light geometry rather than statistical variance estimation, and from [Soft Shadows via Ray Tracing](5136-soft-shadows-via-ray-tracing.md) by its specific focus on area light integration within the volume ray marching context.

### Thin-Walled Glass with Airy Film
- [S1, unofficial] A rendering mode for thin-walled glass (window panes, bulbs, soap bubbles) that skips refraction offset and interior medium, using an optional Airy film as the wall itself.
- [S1, unofficial] Skip refraction offset and interior medium calculation.
- [S1, unofficial] Pass shadow rays through with colored transmittance.
- [S1, unofficial] Use the straight-through BSDF event to keep the previous vertex's MIS state, preventing double counting with other strategies.
- [S1, unofficial] Must ensure MIS consistency between the straight-through BSDF and other sampling strategies to avoid double counting.

### Interactive Directional Ambient Occlusion and Shadow for Volume Ray Casting
- [S2] A strategy to compute directional ambient occlusion and shadows for volume ray casting of structured datasets by evaluating Gaussian integrals at discrete samples along the cone axis.
- [S2] Given the extinction coefficient data volume, create one extra volume computing representative amplitudes of Gaussian functions.
- [S2] Use mipmapping to effectively evaluate Gaussian integrals with different sizes placed along the cone axis, adapting a circle packing approach.
- [S2] Use the computed occlusion to add directional ambient occlusion effects and generate shadows.
- [S2] Ray tracing implementation uses GLSL compute shader.
- [S2] Delivers a better balance between quality results and performance compared to previous specialized procedures, combining directional ambient occlusion and shadow generation under the same framework.

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

## Background
The concept of shadow rays in volume rendering originates from the need to enhance visual shape and depth perception in 3D scalar data visualization. Early volume rendering techniques often ignored light transport occlusion, leading to flat appearances. The integration of shadow casting into volume ray casting was explored to add realistic lighting effects. Specific implementations, such as those using GPU-based ray casting with fragment shaders, compute shaders, OpenCL, and CUDA, have been compared to optimize performance for large-scale aerosols and structured datasets [S2]. Techniques like Interactive Directional Ambient Occlusion and Shadow for Volume Ray Casting further refined this by evaluating Gaussian integrals to balance quality and performance [S2]. Variations such as Mean Intensity Projection also exist, focusing on projecting scalar values rather than just intensity [S3, unofficial].

## Sources
[S1] Add volumetric rendering, advanced materials, and imager effects, https://github.com/MihrimatriX/photon-engine/pull/3, fetched 2026-10-28 (unofficial)
[S2] (PDF) A Comparison between GPU-based Volume Ray Casting ..., https://www.researchgate.net/publication/321454253_A_Comparison_between_GPU-based_Volume_Ray_Casting_Implementations_Fragment_Shader_Compute_Shader_OpenCL_and_CUDA, fetched 2026-10-26
[S3] Direct Volume Rendering Direct Volume Rendering. What is volume rendering?. Accumulate information along 1 dimension line through volume. Volume rendering vs. isosurfaces. No intermediate geometry No , https://www.slideserve.com/lenci/direct-volume-rendering, fetched 2026-10-25 (unofficial)
[S4] Multidimensional Transfer Functions for Interactive Volume ..., https://www.academia.edu/5403340/Multidimensional_Transfer_Functions_for_Interactive_Volume_Rendering, fetched 2026-11-01
