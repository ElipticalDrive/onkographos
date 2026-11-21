# Custom Shading Models

Also called: Custom Shading Routines, Per-voxel Shading Callbacks

## Overview
Custom Shading Models provide a mechanism for users to define arbitrary illumination calculations for volume rendering, bypassing standard lookup tables or fixed-function pipelines. They solve the problem of limited flexibility in standard shading by allowing per-voxel or per-surface-point computation of color based on user-defined logic [S1]. Unlike sibling methods that implement specific physical or heuristic models (e.g., [GGX Microfacet BRDF](3715-ggx-microfacet-brdf.md) or [Blinn-Phong with Metallic Reflections](4252-blinn-phong-with-metallic-reflections.md)), this family represents the *interface* or *paradigm* of user-defined shading, ranging from simple callback routines to complex pipeline integrations.

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
The GGX Microfacet BRDF models specular reflection on rough surfaces by assuming the surface consists of microscopic facets with Gaussian-shaped normal distributions, providing a physically plausible approximation for anisotropic or isotropic roughness. It distinguishes itself from simpler models like Blinn-Phong by using a Normal Distribution Function (NDF) that produces longer specular tails, better matching real-world metallic and dielectric materials, and incorporates a geometry function to account for microfacet shadowing and masking.

### [Voxel Shading Pipeline](3898-voxel-shading-pipeline.md)
The Voxel Shading Pipeline is a per-voxel lighting model that computes illumination for surface points identified by ray marching through a voxel grid. It combines Lambertian diffuse reflection, rim lighting, indirect ambient contribution, and a microfacet-based specular term, modulated by hard shadows and tone mapping. It differs from continuous surface shading by operating on discrete voxel coordinates, rounding ray hit positions to the grid to generate per-voxel specular highlights, and employs hard shadows via secondary ray occlusion checks.

### [Blinn-Phong with Metallic Reflections](4252-blinn-phong-with-metallic-reflections.md)
A custom shading model for ray-marched signed distance fields that extends the classic Blinn-Phong approximation with environment reflections and emissive contributions to simulate urban night scenes. It distinguishes itself from standard Blinn-Phong by modulating specular intensity, power, and reflection weight based on a material's metallic property, and from GGX microfacet models by using a simplified Schlick-like Fresnel term rather than a full microfacet distribution.

### [Fast Shading for Voxel Engines](5606-fast-shading-for-voxel-engines.md)
Fast Shading for Voxel Engines is a CPU-based lighting algorithm for voxel engines that assumes vertical sun rays. It modifies voxel palette indices to simulate lighting by darkening voxels based on the occupancy of their immediate neighbors. It sets itself apart from general-purpose shading models by operating directly on discrete palette indices rather than computing continuous radiance, and by relying on a fixed, pre-defined neighborhood kernel instead of ray-marching for occlusion.

### [Gradient-Based Shading](6896-gradient-based-shading.md)
Gradient-Based Shading is a discrete volume rendering (DVR) technique that renders 3D textures using front-to-back accumulation, shading voxels via a pseudo-normal estimated from the volume gradient and a directional light model. It differs from sibling methods like [GGX Microfacet BRDF](3715-ggx-microfacet-brdf.md) or [Blinn-Phong with Metallic Reflections](4252-blinn-phong-with-metallic-reflections.md) by relying on a simple Lambertian diffuse model driven by finite-difference normals rather than complex microfacet distributions or metallic reflections. It is distinct from [Fast Shading for Voxel Engines](5606-fast-shading-for-voxel-engines.md) by its explicit use of a transfer function LUT and continuous/discrete normalization modes within a standard raymarching loop.

### [Multidirectional Occlusion Shading](7667-multidirectional-occlusion-shading.md)
Multidirectional Occlusion Shading (M-DOS) is a direct volume rendering technique that simulates directional light scattering to provide realistic soft-shadowing and depth cues. Unlike the original MS-DOS model which constrains light to the viewing direction, M-DOS allows arbitrary light positioning within the hemisphere defined by the view vector by using elliptical footprints instead of symmetric spherical ones. It achieves this through incremental convolution of an opacity buffer with a tilted conical phase function, requiring no precomputation.

### [Single-Atomic High-Quality Robust Variant (HQS1R)](10218-single-atomic-high-quality-robust-variant-hqs1r.md)
HQS1R is a high-quality point cloud rendering method that reduces aliasing by blending overlapping points together, similar to mipmapping. It is a simplified, compute-based implementation of the blending algorithm from [BHZK05], designed to be faster than aliased `GL_POINTS` results while providing anti-aliasing. It sets itself apart from its sibling [GGX Microfacet BRDF](3715-ggx-microfacet-brdf.md) and [Blinn-Phong with Metallic Reflections](4252-blinn-phong-with-metallic-reflections.md) by operating on point primitives rather than surface geometry, and from [HQS1×](#single-atomic-high-quality-variant-hqs1x) by using a robust overflow protection mechanism that prevents artifacts when more than 255 points contribute to a pixel. Unlike surface splatting [ZPVG01], it blends overlapping points without considering oriented disks with variable radii.

### [YUV Color Space Shading with Glow](11187-yuv-color-space-shading-with-glow.md)
This method performs volume raycasting illumination in the YUV color space to decouple luminance from chrominance, allowing for independent control of brightness and color. It differs from standard RGB shading by computing lighting effects (such as specular highlights and glow) primarily on the luminance channel, while interpolating color channels based on a technique-derived factor. This approach is particularly useful for capturing indirect illumination independently of the transfer function.

### Light Evaluation for SDF Faces
Applies lighting (diffuse, specular, ambient) to a face color based on normal and light direction [S2, unofficial]. Set $n.z = \text{abs}(n.z)$; normalize light vector from $\sin/\cos$ of `_LightAngle`; compute $\text{col} = \max(\text{faceColor.rgb}, 0) + \text{GetSpecular}(n, \text{light}) \times \text{faceColor.a}$; multiply $\text{col}$ by $(1 - \text{dot}(n, \text{light}) \times \text{\_Diffuse})$; multiply $\text{col}$ by $\text{lerp}(\text{\_Ambient}, 1, n.z \times n.z)$; set $\text{color} = \text{float4}(\text{col}, \text{faceColor.a})$ [S2, unofficial].

### Procedural Road Surface
A procedural surface shader for roads that differentiates between asphalt, curb/shoulder, and terrain based on lateral distance to the road centerline, including emissive lane markings [S3, unofficial].
1. Calculate $d = p.x - \text{roadX}(p.z)$ and $ad = \text{abs}(d)$.
2. If $ad < \text{ROAD\_HALF\_WIDTH}$: set albedo to dark gray, metal to 0.35, emissive to 0.0. Calculate lane lines using `smoothstep` for edge lines, center line, and dashed separators. Add emissive cyan color for lines.
3. Else if $ad < \text{ROAD\_HALF\_WIDTH} + \text{SHOULDER}$: set albedo to dark gray, metal to 0.1, emissive to magenta strip at road edge.
4. Else: set albedo to very dark, metal to 0.0, emissive to 0.0 [S3, unofficial].
Parameters: Asphalt metal: 0.35 (wet-look); Lane line color: $\text{vec3}(0.15, 1.0, 1.1) \times 1.3$; Edge line offset: $\text{ROAD\_HALF\_WIDTH} - 0.3$; Center line offset: 0.18; Dashed line frequency: $p.z \times 0.18$ [S3, unofficial].

### Triangle Subdivision Shading
Divide each triangle into four subtriangles by splitting each edge in the middle. Apply either Gouraud or Phong shading algorithm to the sub-triangles. This process can be iterated several times. The `smooth` argument specifies one plus the number of times to divide the triangles [S4]. Parameter: `smooth`: integer specifying one plus the number of subdivision iterations. `smooth=1` means no subdivision [S4]. Pitfall: Using too high a level of `smooth` can cause rendering quality to deteriorate due to artifacts from polygon filling, exacerbated on devices using anti-aliasing [S4].

## Background
The concept of custom shading in volume rendering originated with the VolPack library, which provided users with two primary options to define non-standard shading models: creating custom lookup tables or defining callback routines [S1]. The callback approach, invoked via `vpSetCallback()`, allowed a user-defined function (e.g., `myshader()`) to be called for each voxel during rendering, enabling arbitrary color computation [S1]. This paradigm laid the groundwork for modern per-voxel shading callbacks and custom shading routines in ray marching and voxel engines.

## Sources
[S1] VolPack User's Guide, http://www-graphics.stanford.edu/software/volpack/vp_userguide.html, fetched 2026-10-28
[S2] andersblomqvist/unity-nanovdb-renderer, https://github.com/andersblomqvist/unity-nanovdb-renderer, fetched 2026-10-27 (unofficial)
[S3] PIGJET/neon-raymarcher, https://github.com/PIGJET/neon-raymarcher, fetched 2026-10-27 (unofficial)
[S4] Computing and Displaying Isosurfaces inR, https://doi.org/10.18637/jss.v028.i01, fetched 2026-10-27
