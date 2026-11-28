# Custom Shading Models

Also called: Custom Shading Routines, Per-voxel Shading Callbacks

## Overview
Custom Shading Models provide a framework for defining arbitrary illumination calculations within volume rendering and ray-marching pipelines, allowing developers to replace standard analytic lighting with user-defined logic [G]. This family enables the implementation of specialized effects such as physically based microfacet reflections, directional occlusion, and color-space-specific glow, which are not covered by generic diffuse or specular models [G]. It distinguishes itself from sibling methods like [Distance Color Blending](791-distance-color-blending.md) by operating on explicit geometric data (normals, positions) rather than raw scalar field values, and from [Ambient Occlusion via SDF](1631-ambient-occlusion-via-sdf.md) by allowing full directional light integration rather than just ambient darkening [G].

## Shared principles
All methods in this family operate on a surface point $\mathbf{p}$ and its associated geometric data, primarily the surface normal $\mathbf{n}$ [G]. The normal is typically derived from the Signed Distance Function (SDF) $f(\mathbf{x})$ using the gradient:
$$ \mathbf{n} = \frac{\nabla f(\mathbf{p})}{\|\nabla f(\mathbf{p})\|} $$
where $\nabla f(\mathbf{p})$ is estimated via finite differences, such as solving a linear system based on values at vertices of a tetrahedron, or via back-propagation [G].

The core shading computation generally follows this structure:
1. **Geometric Setup:** Determine the view direction $\mathbf{v}$ (from $\mathbf{p}$ to camera) and light direction $\mathbf{l}$ (from $\mathbf{p}$ to light source) [G].
2. **Occlusion/Ambient Occlusion:** Calculate shadow factors or ambient occlusion values to determine how much light reaches $\mathbf{p}$ from the environment or specific lights [G].
3. **Reflectance Model:** Compute the diffuse and specular components based on the angle between $\mathbf{n}$, $\mathbf{l}$, and the reflection vector $\mathbf{r}$ [G].
4. **Color Accumulation:** Combine the base material color with the lighting contributions, often modulated by depth cueing or atmospheric effects [G].

## Subtopics

### [GGX Microfacet BRDF](3715-ggx-microfacet-brdf.md)
The GGX Microfacet BRDF is a specular reflection model that approximates surface roughness using the Gaussian distribution of microfacet normals (GGX) combined with the Schlick Fresnel approximation and a Smith geometry function. It is distinguished from its siblings, such as [Blinn-Phong with Metallic Reflections](4252-blinn-phong-with-metallic-reflections.md), by providing physically plausible energy conservation and accurate specular highlights for rough surfaces, whereas Blinn-Phong is an empirical approximation. It serves as the standard for high-quality specular shading in real-time engines and voxel renderers.

### [Light Evaluation for SDF Faces](3751-light-evaluation-for-sdf-faces.md)
This method applies lighting (diffuse, specular, and ambient) to a face color based on the surface normal and light direction. It is designed for SDF-based rendering where the "face" represents a discretized surface element. It differs from general volume shading by operating directly on the extracted face geometry's color and alpha, using a simplified analytic lighting model rather than full ray-marched integration.

### [Blinn-Phong with Metallic Reflections](4252-blinn-phong-with-metallic-reflections.md)
A custom shading model for ray-marched signed distance fields that extends the classic Blinn-Phong approximation with environment reflections and emissive contributions to simulate urban night scenes. It distinguishes itself from standard Blinn-Phong by modulating specular intensity, power, and reflection weight based on a material's metallic property, and from GGX microfacet models by using a simplified Schlick-like Fresnel term rather than a full microfacet distribution.

### [Multidirectional Occlusion Shading](7667-multidirectional-occlusion-shading.md)
Multidirectional Occlusion Shading (M-DOS) is a direct volume rendering technique that simulates directional light scattering to produce realistic soft-shadowing and depth cues. Unlike the original MS-DOS model, which constrains the light direction to the viewing direction and uses symmetric disc-shaped Gaussian kernels, M-DOS allows arbitrary light positioning within the hemisphere defined by the view vector by using elliptical footprints derived from a tilted cone. It achieves this through incremental convolution of an opacity buffer, requiring no precomputation and maintaining interactive frame rates.

### [Single-Atomic High-Quality Robust Variant (HQS1R)](10218-single-atomic-high-quality-robust-variant-hqs1r.md)
HQS1R is a high-quality point cloud rendering method that reduces aliasing by blending overlapping points together, similar to mipmapping. It is a simplified, compute-based implementation of the blending algorithm from [BHZK05], designed to be faster than aliased `GL_POINTS` results while providing anti-aliasing. Unlike surface splatting [ZPVG01], it blends overlapping points without considering oriented disks with variable radii. It sets itself apart from the [Single-Atomic High-Quality Robust Variant (HQS1R)](10218-single-atomic-high-quality-robust-variant-hqs1r.md) by using a robust overflow protection mechanism that prevents artifacts when more than 255 points contribute to a pixel.

### [YUV Color Space Shading with Glow](11187-yuv-color-space-shading-with-glow.md)
This method performs volume raycasting illumination in the YUV color space to decouple luminance from chrominance, allowing for independent control of brightness and color. It differs from standard RGB shading by computing lighting effects (such as specular highlights and glow) primarily on the luminance channel, while interpolating color channels based on a technique-derived factor. This approach is particularly useful for capturing indirect illumination independently of the transfer function.

### [Tangent Space Shading](12616-tangent-space-shading.md)
Tangent Space Shading is a gradient-less lighting technique that approximates the diffuse lighting component of the Phong shading model without computing explicit gradients for every texel. It is distinguished from sibling methods like [GGX Microfacet BRDF](3715-ggx-microfacet-brdf.md) and [Light Evaluation for SDF Faces](3751-light-evaluation-for-sdf-faces.md) by its reliance on rendering artifacts (double-pass texture coordinate offsets) rather than geometric derivatives to simulate surface orientation.

### [Voxel-Specific Shading Pipelines](17027-voxel-specific-shading-pipelines.md)
Voxel-Specific Shading Pipelines are a family of rendering techniques designed to compute illumination for data stored in discrete 3D grids (voxels). Unlike continuous surface shading which operates on smooth Signed Distance Functions (SDFs) or triangle meshes, these methods operate on the discrete topology of voxel cells, handling both surface-like boundaries and volumetric density. The family encompasses approaches ranging from per-voxel geometric shading with hard shadows to CPU-based palette index manipulation and dedicated volumetric ray-marching shaders for fluid simulations.

### Procedural Road Surface
- **Definition:** A procedural surface shader for roads that differentiates between asphalt, curb/shoulder, and terrain based on lateral distance to the road centerline, including emissive lane markings [S2, unofficial].
- **Steps:**
    1. Calculate $d = p.x - \text{roadX}(p.z)$ and $ad = |d|$ [S2, unofficial].
    2. If $ad < \text{ROAD\_HALF\_WIDTH}$: set albedo to dark gray, metal to 0.35, emissive to 0.0 [S2, unofficial]. Calculate lane lines using `smoothstep` for edge lines, center line, and dashed separators [S2, unofficial]. Add emissive cyan color for lines [S2, unofficial].
    3. Else if $ad < \text{ROAD\_HALF\_WIDTH} + \text{SHOULDER}$: set albedo to dark gray, metal to 0.1, emissive to magenta strip at road edge [S2, unofficial].
    4. Else: set albedo to very dark, metal to 0.0, emissive to 0.0 [S2, unofficial].
- **Parameters:** Asphalt metal: 0.35 (wet-look) [S2, unofficial]. Lane line color: $\text{vec3}(0.15, 1.0, 1.1) \times 1.3$ [S2, unofficial]. Edge line offset: $\text{ROAD\_HALF\_WIDTH} - 0.3$ [S2, unofficial]. Center line offset: 0.18 [S2, unofficial]. Dashed line frequency: $p.z \times 0.18$ [S2, unofficial].

### Triangle Subdivision Shading
- **Steps:**
    1. Divide each triangle into four subtriangles by splitting each edge in the middle [S3].
    2. Apply either Gouraud or Phong shading algorithm to the sub-triangles [S3].
    3. This process can be iterated several times [S3]. The `smooth` argument specifies one plus the number of times to divide the triangles [S3].
- **Parameters:** `smooth`: integer specifying one plus the number of subdivision iterations. `smooth=1` means no subdivision [S3].
- **Pitfalls:** Using too high a level of `smooth` can cause rendering quality to deteriorate due to artifacts from polygon filling, exacerbated on devices using anti-aliasing [S3].

## Background
The concept of custom shading in volume rendering originated with early libraries like VolPack, which introduced the `vpSetCallback` mechanism allowing users to define per-voxel shading routines instead of relying on fixed lookup tables [S1]. This allowed for arbitrary color transfer functions and lighting models but noted potential performance degradation [S1]. The family has since expanded to include physically based rendering models like GGX [3715-ggx-microfacet-brdf.md], specialized techniques for point clouds like HQS1R [10218-single-atomic-high-quality-robust-variant-hqs1r.md], and color-space specific methods like YUV shading [11187-yuv-color-space-shading-with-glow.md]. The family also encompasses discrete voxel-specific pipelines [17027-voxel-specific-shading-pipelines.md] and procedural surface shaders for specific domains like road rendering [S2, unofficial].

## Sources
[S1] VolPack User's Guide, http://www-graphics.stanford.edu/software/volpack/vp_userguide.html, fetched 2026-10-28
[S2] PIGJET/neon-raymarcher, https://github.com/PIGJET/neon-raymarcher, fetched 2026-10-27 (unofficial)
[S3] Computing and Displaying Isosurfaces inR, https://doi.org/10.18637/jss.v028.i01, fetched 2026-10-27
