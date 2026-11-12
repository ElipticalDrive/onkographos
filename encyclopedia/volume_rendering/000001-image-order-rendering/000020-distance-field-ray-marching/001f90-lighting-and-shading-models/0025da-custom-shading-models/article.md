# Custom Shading Models

Also called: Custom Shading Routines, Per-voxel Shading Callbacks

## Overview
Custom Shading Models provide a mechanism for users to define arbitrary lighting and color computation logic for volume rendering, bypassing standard pre-defined shading tables or simple lookup functions. They solve the problem of limited expressiveness in standard volumetric shading by allowing per-voxel or per-surface-point callbacks that can implement complex material properties, non-physical artistic effects, or specialized lighting approximations [S1]. Unlike sibling methods that focus on specific physical models (like GGX) or specific data types (like voxel grids), this family is defined by the *interface* for customization—typically a callback function or a custom table initialization routine—rather than a single mathematical formula [S1].

## Shared principles
All methods in this family operate on a surface point $\mathbf{p}$ and its associated geometric data, primarily the surface normal $\mathbf{n}$ [G]. The normal is typically derived from the Signed Distance Function (SDF) $f(\mathbf{x})$ using the gradient:
$$ \mathbf{n} = \frac{\nabla f(\mathbf{p})}{\|\nabla f(\mathbf{p})\|} $$
where $\nabla f(\mathbf{p})$ is estimated via finite differences, such as solving a linear system based on values at vertices of a tetrahedron, or via back-propagation [S1].

The core shading computation generally follows this structure:
1. **Geometric Setup:** Determine the view direction $\mathbf{v}$ (from $\mathbf{p}$ to camera) and light direction $\mathbf{l}$ (from $\mathbf{p}$ to light source) [G].
2. **Occlusion/Ambient Occlusion:** Calculate shadow factors or ambient occlusion values to determine how much light reaches $\mathbf{p}$ from the environment or specific lights [G].
3. **Reflectance Model:** Compute the diffuse and specular components based on the angle between $\mathbf{n}$, $\mathbf{l}$, and the reflection vector $\mathbf{r}$ [G].
4. **Color Accumulation:** Combine the base material color with the lighting contributions, often modulated by depth cueing or atmospheric effects [G].

## Subtopics

### [GGX Microfacet BRDF](3715-ggx-microfacet-brdf.md)
A specular reflection model that approximates surface roughness using a Gaussian distribution of microfacet normals. It differs from simpler models like Blinn-Phong by providing physically plausible energy conservation and accurate highlight shapes for rough surfaces, making it the standard for high-quality real-time rendering.

### [Light Evaluation for SDF Faces](3751-light-evaluation-for-sdf-faces.md)
Applies lighting (diffuse, specular, and ambient) to a face color based on the surface normal and light direction. It is designed for SDF-based rendering where the "face" represents a discretized surface element. It differs from general volume shading by operating directly on the extracted face geometry's color and alpha, using a simplified analytic lighting model rather than full ray-marched integration.

### [Voxel Shading Pipeline](3898-voxel-shading-pipeline.md)
A per-voxel lighting model that computes illumination for surface points identified by ray marching through a voxel grid. It combines Lambertian diffuse reflection, rim lighting, indirect ambient contribution, and a microfacet-based specular term, modulated by hard shadows and tone mapping. It differs from continuous surface shading by operating on discrete voxel coordinates, rounding ray hit positions to the grid to generate per-voxel specular highlights, and employs hard shadows via secondary ray occlusion checks.

### [Blinn-Phong with Metallic Reflections](4252-blinn-phong-with-metallic-reflections.md)
A custom shading model for ray-marched signed distance fields that extends the classic Blinn-Phong approximation with environment reflections and emissive contributions to simulate urban night scenes. It distinguishes itself from standard Blinn-Phong by modulating specular intensity, power, and reflection weight based on a material's metallic property, and from GGX microfacet models by using a simplified Schlick-like Fresnel term rather than a full microfacet distribution.

### [Fast Shading for Voxel Engines](5606-fast-shading-for-voxel-engines.md)
A CPU-based lighting algorithm for voxel engines that simulates directional illumination (specifically vertical sun rays) by modifying voxel palette indices. It differs from its sibling `fast_shade1` by using a more complex neighborhood check involving 11 specific neighbors to determine shading intensity, offering higher visual detail at the cost of increased neighbor lookups. It operates directly on the voxel grid data structure rather than rendering a mesh or ray-marching a distance field.

### [Gradient-Based Shading](6896-gradient-based-shading.md)
A discrete volume rendering (DVR) technique that renders 3D textures using front-to-back accumulation, shading sampled voxels via a pseudo-normal estimated from the volume gradient and a directional light model. It solves the problem of flat, unlit volume appearance by introducing surface-like shading cues derived from the scalar field's spatial derivatives, allowing standard lighting models to be applied to volumetric data. Unlike pure volumetric scattering methods, it treats the volume as a collection of semi-transparent surfaces, offering a balance between performance and visual fidelity for medical and scientific data.

### [Multidirectional Occlusion Shading](7667-multidirectional-occlusion-shading.md)
A direct volume rendering technique that simulates directional light scattering to produce realistic soft-shadowing and depth cues. Unlike the original MS-DOS model, which constrains the light direction to the viewing direction and uses symmetric disc-shaped Gaussian kernels, M-DOS allows arbitrary light positioning within the hemisphere defined by the view vector by using elliptical footprints derived from a tilted cone. It achieves this through incremental convolution of an opacity buffer, requiring no precomputation and maintaining interactive frame rates.

### [Single-Atomic High-Quality Robust Variant (HQS1R)](10218-single-atomic-high-quality-robust-variant-hqs1r.md)
A high-quality point cloud rendering method that reduces aliasing by blending overlapping points together, similar to mipmapping. It is a simplified, compute-based implementation of the blending algorithm from [BHZK05], designed to be faster than aliased `GL_POINTS` results while providing anti-aliasing. Unlike surface splatting [ZPVG01], it blends overlapping points without considering oriented disks with variable radii. It sets itself apart from the [Single-Atomic High-Quality Variant (HQS1x)](#single-atomic-high-quality-variant-hqs1x) by using a robust overflow protection mechanism that prevents artifacts when more than 255 points contribute to a pixel.

### [YUV Color Space Shading with Glow](11187-yuv-color-space-shading-with-glow.md)
Performs volume raycasting illumination in the YUV color space to decouple luminance from chrominance, allowing for independent control of brightness and color. It differs from standard RGB shading by computing lighting effects (such as specular highlights and glow) primarily on the luminance channel, while interpolating color channels based on a technique-derived factor. This approach is particularly useful for capturing indirect illumination independently of the transfer function.

### Procedural Road Surface
A procedural surface shader for roads that differentiates between asphalt, curb/shoulder, and terrain based on lateral distance to the road centerline, including emissive lane markings [S2, unofficial].
1. Calculate $d = p.x - \text{roadX}(p.z)$ and $ad = |d|$.
2. If $ad < \text{ROAD\_HALF\_WIDTH}$: set albedo to dark gray, metal to 0.35, emissive to 0.0. Calculate lane lines using `smoothstep` for edge lines, center line, and dashed separators. Add emissive cyan color for lines.
3. Else if $ad < \text{ROAD\_HALF\_WIDTH} + \text{SHOULDER}$: set albedo to dark gray, metal to 0.1, emissive to magenta strip at road edge.
4. Else: set albedo to very dark, metal to 0.0, emissive to 0.0 [S2, unofficial].
Parameters: Asphalt metal: 0.35 (wet-look); Lane line color: $\text{vec3}(0.15, 1.0, 1.1) \times 1.3$; Edge line offset: $\text{ROAD\_HALF\_WIDTH} - 0.3$; Center line offset: 0.18; Dashed line frequency: $p.z \times 0.18$ [S2, unofficial].

### Triangle Subdivision Shading
Divide each triangle into four subtriangles by splitting each edge in the middle. Apply either Gouraud or Phong shading algorithm to the sub-triangles. This process can be iterated several times. The `smooth` argument specifies one plus the number of times to divide the triangles [S3].
Parameter: `smooth`: integer specifying one plus the number of subdivision iterations. `smooth=1` means no subdivision [S3].
Pitfall: Using too high a level of `smooth` can cause rendering quality to deteriorate due to artifacts from polygon filling, exacerbated on devices using anti-aliasing [S3].

### Tangent Space Shading
A gradient-less lighting shader that approximates the diffuse lighting component of the Phong shading model without computing gradients for every texel [S4].
Pitfall: Does not generate correct lighting; it creates a visual effect via a system-dependent technique. Produces seams for bricked shapes along the borders of the bricks [S4].
Comparison: Use `vzTMGradientShader` for correct volumetric lighting if bricking artifacts are a concern [S4].

## Background
The concept of custom shading in volume rendering originated with early libraries like VolPack, which provided a framework for users to define their own shading lookup tables or callback routines to compute voxel colors [S1]. VolPack allowed users to initialize custom tables of any size or define a callback function via `vpSetCallback` to be invoked for each voxel, enabling non-standard shading models such as color transfer functions or custom lighting approximations [S1]. However, this flexibility came at the cost of significant performance degradation compared to optimized lookup table methods [S1].

In the context of Signed Distance Fields (SDFs) and ray marching, custom shading became a standard technique for achieving specific artistic styles or physically based rendering (PBR) effects that were not supported by built-in shaders. This includes implementing microfacet BRDFs like GGX, metallic reflections, and complex procedural materials like road surfaces [S2, unofficial][S3]. The evolution of GPU computing and compute shaders has further expanded custom shading to include high-performance techniques like atomic blending for point clouds (HQS1R) and YUV-space illumination [S1][S2, unofficial].

## Sources
[S1] VolPack User's Guide, http://www-graphics.stanford.edu/software/volpack/vp_userguide.html, fetched 2026-10-28
[S2] PIGJET/neon-raymarcher, https://github.com/PIGJET/neon-raymarcher, fetched 2026-10-27 (unofficial)
[S3] Computing and Displaying Isosurfaces inR, https://doi.org/10.18637/jss.v028.i01, fetched 2026-10-27
[S4] Chapter 4. Texture Mapping Render Action, https://techpubs.jurassic.nl/library/manuals/4000/007-4389-010/sgi_html/ch04.html, fetched 2026-10-27
