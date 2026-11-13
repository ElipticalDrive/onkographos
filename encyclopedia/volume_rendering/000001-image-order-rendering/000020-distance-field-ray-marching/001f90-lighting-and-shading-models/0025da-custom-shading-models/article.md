# Custom Shading Models

Also called: Custom Shading Routines, Per-voxel Shading Callbacks

## Overview
Custom Shading Models provide a mechanism for users to define arbitrary illumination logic for volume rendering, bypassing standard predefined lighting equations. They solve the problem of rigid lighting constraints by allowing per-voxel or per-surface-point computation of color based on user-defined material properties, geometric data, and environmental factors. What sets this family apart from its siblings is the architectural approach to integration: some methods inject custom logic via API callbacks into a rendering pipeline [S1], while others implement self-contained mathematical models (e.g., GGX, YUV) or procedural rules (e.g., road surfaces) directly within the shading step.

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
A physically based specular model using Gaussian microfacet distribution. Choose for high-quality rough surface highlights.

### [Light Evaluation for SDF Faces](3751-light-evaluation-for-sdf-faces.md)
Analytic lighting applied to discretized SDF faces. Choose for simple, fast face-based shading without full ray integration.

### [Voxel Shading Pipeline](3898-voxel-shading-pipeline.md)
Per-voxel lighting combining diffuse, rim, indirect, and microfacet specular terms with hard shadows. Choose for voxel-based engines requiring discrete specular highlights.

### [Blinn-Phong with Metallic Reflections](4252-blinn-phong-with-metallic-reflections.md)
Blinn-Phong extended with metallic modulation and environment reflections. Choose for stylized urban night scenes with metallic materials.

### [Fast Shading for Voxel Engines](5606-fast-shading-for-voxel-engines.md)
CPU-based palette index modification for directional illumination. Choose for voxel engines needing fast, pre-computed shading without ray marching.

### [Multidirectional Occlusion Shading](7667-multidirectional-occlusion-shading.md)
Direct volume rendering using tilted cone phase functions for soft shadows. Choose for interactive DVR with directional light scattering.

### [Single-Atomic High-Quality Robust Variant (HQS1R)](10218-single-atomic-high-quality-robust-variant-hqs1r.md)
Point cloud rendering with atomic blending and overflow protection. Choose for high-quality, anti-aliased point cloud visualization.

### [YUV Color Space Shading with Glow](11187-yuv-color-space-shading-with-glow.md)
Illumination in YUV space decoupling luminance and chrominance. Choose for independent control of brightness and color with volumetric glow.

### Procedural Road Surface
A procedural surface shader for roads that differentiates between asphalt, curb/shoulder, and terrain based on lateral distance to the road centerline, including emissive lane markings [S2, unofficial].
1. Calculate $d = p.x - \text{roadX}(p.z)$ and $ad = |d|$ [S2, unofficial].
2. If $ad < \text{ROAD\_HALF\_WIDTH}$: set albedo to dark gray, metal to $0.35$, emissive to $0.0$. Calculate lane lines using `smoothstep` for edge lines, center line, and dashed separators. Add emissive cyan color for lines [S2, unofficial].
3. Else if $ad < \text{ROAD\_HALF\_WIDTH} + \text{SHOULDER}$: set albedo to dark gray, metal to $0.1$, emissive to magenta strip at road edge [S2, unofficial].
4. Else: set albedo to very dark, metal to $0.0$, emissive to $0.0$ [S2, unofficial].
Parameters: Asphalt metal $0.35$ (wet-look); Lane line color $\text{vec3}(0.15, 1.0, 1.1) \times 1.3$; Edge line offset $\text{ROAD\_HALF\_WIDTH} - 0.3$; Center line offset $0.18$; Dashed line frequency $p.z \times 0.18$ [S2, unofficial].

### Triangle Subdivision Shading
Divide each triangle into four subtriangles by splitting each edge in the middle [S3]. Apply either Gouraud or Phong shading algorithm to the sub-triangles [S3]. This process can be iterated several times. The `smooth` argument specifies one plus the the number of times to divide the triangles [S3].
Parameter: `smooth` integer specifying one plus the number of subdivision iterations. `smooth=1` means no subdivision [S3].
Pitfall: Using too high a level of `smooth` can cause rendering quality to deteriorate due to artifacts from polygon filling, exacerbated on devices using anti-aliasing [S3].

### Tangent Space Shading
A gradient-less lighting shader that approximates the diffuse lighting component of the Phong shading model without computing gradients for every texel [S4].
Pitfall: Does not generate correct lighting; it creates a visual effect via a system-dependent technique [S4]. Produces seams for bricked shapes along the borders of the bricks [S4].
Comparison: Use `vzTMGradientShader` for correct volumetric lighting if bricking artifacts are a concern [S4].

## Background
The concept of custom shading in volume rendering originated with early libraries like VolPack, which provided mechanisms for users to initialize custom shading lookup tables or define callback routines for per-voxel shading via `vpSetCallback` [S1]. This allowed for non-encoded normal vectors and arbitrary color transfer functions but warned of significant performance degradation [S1]. Modern implementations have evolved into self-contained mathematical models (e.g., GGX, YUV) and procedural shaders integrated directly into ray marching or voxel rendering pipelines, offering more control over material properties like metallic reflections and atmospheric attenuation [S2, unofficial][S3, S4].

## Sources
[S1] VolPack User's Guide, http://www-graphics.stanford.edu/software/volpack/vp_userguide.html, fetched 2026-10-28
[S2] PIGJET/neon-raymarcher, https://github.com/PIGJET/neon-raymarcher, fetched 2026-10-27 (unofficial)
[S3] Computing and Displaying Isosurfaces inR, https://doi.org/10.18637/jss.v028.i01, fetched 2026-10-27
[S4] Chapter 4. Texture Mapping Render Action, https://techpubs.jurassic.nl/library/manuals/4000/007-4389-010/sgi_html/ch04.html, fetched 2026-10-27
