# Custom Shading Models

Also called: Custom Shading Routines, Per-voxel Shading Callbacks

## Overview
Custom Shading Models provide a mechanism for users to define arbitrary illumination calculations for surface points or voxels identified during ray marching, rather than relying on built-in lighting primitives. This family solves the problem of visual fidelity and material expressiveness when standard models (like simple diffuse or Blinn-Phong) are insufficient for specific artistic or scientific requirements. It distinguishes itself from sibling methods like [Distance Color Blending](791-distance-color-blending.md) or [Ray-marched Shadows](1087-ray-marched-shadows.md) by allowing the user to inject custom logic (code or lookup tables) into the shading step, effectively decoupling the geometric traversal from the radiance computation [S1].

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
Models specular reflection on rough surfaces using a Gaussian normal distribution, providing physically plausible results for anisotropic or isotropic roughness. It is chosen when high-fidelity metallic or dielectric specular highlights are required, distinguishing itself from simpler models by its longer specular tails and microfacet geometry function.

### [Light Evaluation for SDF Faces](3751-light-evaluation-for-sdf-faces.md)
Applies standard lighting (diffuse, specular, ambient) to a face color based on the surface normal and light direction. It is a basic custom shading approach for discrete faces.

### [Voxel Shading Pipeline](3898-voxel-shading-pipeline.md)
A per-voxel lighting model that computes illumination for surface points identified by ray marching through a voxel grid. It combines Lambertian diffuse, rim lighting, indirect ambient, and microfacet specular terms, modulated by hard shadows. It is chosen for voxel-based engines (like Minecraft-style games) and differs from continuous surface shading by operating on discrete voxel coordinates and using hard shadows via secondary ray occlusion.

### [Blinn-Phong with Metallic Reflections](4252-blinn-phong-with-metallic-reflections.md)
Extends the classic Blinn-Phong approximation with environment reflections and emissive contributions to simulate urban night scenes. It is chosen for stylized or real-time applications where full microfacet models are too expensive. It distinguishes itself by modulating specular intensity and reflection weight based on a material's metallic property, using a simplified Schlick-like Fresnel term.

### [Fast Shading for Voxel Engines](5606-fast-shading-for-voxel-engines.md)
A CPU-based lighting algorithm for voxel engines that assumes vertical sun rays. It modifies voxel palette indices to simulate lighting by darkening voxels based on neighbor occupancy. It is chosen for performance-critical voxel engines where GPU shading is unavailable or too costly. It sets itself apart by operating directly on discrete palette indices rather than computing continuous radiance.

### [Gradient-Based Shading](6896-gradient-based-shading.md)
A direct volume rendering (DVR) technique that shades sampled voxels via a pseudo-normal estimated from the volume gradient. It is chosen for medical and scientific data visualization to add surface-like shading cues to scalar fields. Unlike pure volumetric scattering, it treats the volume as semi-transparent surfaces.

### [Multidirectional Occlusion Shading](7667-multidirectional-occlusion-shading.md)
Simulates directional light scattering to provide realistic soft-shadowing and depth cues in direct volume rendering. It is chosen when soft shadows and directional lighting are needed without precomputation. It allows arbitrary light positioning within the view hemisphere using elliptical footprints.

### [Single-Atomic High-Quality Robust Variant (HQS1R)](10218-single-atomic-high-quality-robust-variant-hqs1r.md)
A high-quality point cloud rendering method that reduces aliasing by blending overlapping points using atomic operations. It is chosen for high-performance point cloud rendering where anti-aliasing is critical. It operates on point primitives and uses robust overflow protection.

### [YUV Color Space Shading with Glow](11187-yuv-color-space-shading-with-glow.md)
Performs volume raycasting illumination in the YUV color space to decouple luminance from chrominance. It is chosen when independent control of brightness and color is needed, or when capturing indirect illumination independently of the transfer function.

#### Procedural Road Surface
- **Definition:** A procedural surface shader for roads that differentiates between asphalt, curb/shoulder, and terrain based on lateral distance to the road centerline, including emissive lane markings [S2, unofficial].
- **Steps:**
    1. Calculate $d = p.x - \text{roadX}(p.z)$ and $ad = |d|$ [S2, unofficial].
    2. If $ad < \text{ROAD\_HALF\_WIDTH}$: set albedo to dark gray, metal to 0.35, emissive to 0.0. Calculate lane lines using `smoothstep` for edge lines, center line, and dashed separators. Add emissive cyan color for lines [S2, unofficial].
    3. Else if $ad < \text{ROAD\_HALF\_WIDTH} + \text{SHOULDER}$: set albedo to dark gray, metal to 0.1, emissive to magenta strip at road edge [S2, unofficial].
    4. Else: set albedo to very dark, metal to 0.0, emissive to 0.0 [S2, unofficial].
- **Parameters:** Asphalt metal: 0.35 (wet-look) [S2, unofficial]. Lane line color: $\text{vec3}(0.15, 1.0, 1.1) \times 1.3$ [S2, unofficial]. Edge line offset: $\text{ROAD\_HALF\_WIDTH} - 0.3$ [S2, unofficial]. Center line offset: 0.18 [S2, unofficial]. Dashed line frequency: $p.z \times 0.18$ [S2, unofficial].

#### Triangle Subdivision Shading
- **Steps:**
    1. Divide each triangle into four subtriangles by splitting each edge in the middle [S3].
    2. Apply either Gouraud or Phong shading algorithm to the sub-triangles [S3].
    3. This process can be iterated several times. The `smooth` argument specifies one plus the number of times to divide the triangles [S3].
- **Parameters:** `smooth`: integer specifying one plus the number of subdivision iterations. `smooth=1` means no subdivision [S3].
- **Pitfall:** Using too high a level of `smooth` can cause rendering quality to deteriorate due to artifacts from polygon filling, exacerbated on devices using anti-aliasing [S3].

## Background
The concept of custom shading in volume rendering originated with early volume rendering libraries like VolPack, which provided mechanisms for users to define custom shading lookup tables or callback routines to compute per-voxel colors [S1]. VolPack allowed users to initialize their own shading tables or define callback functions (e.g., `vpSetCallback`) to override default shading behavior, enabling complex material definitions and depth cueing [S1]. This approach was necessary because built-in models were often insufficient for specific scientific or artistic needs, and callback functions, while flexible, could lead to significant performance degradation [S1]. In modern ray marching and voxel rendering, custom shading has evolved into shader-based implementations (GLSL/HLSL) that allow for real-time computation of complex BRDFs, shadows, and atmospheric effects directly on the GPU, maintaining the flexibility of the original callback concept while leveraging hardware acceleration.

## Sources
[S1] VolPack User's Guide, http://www-graphics.stanford.edu/software/volpack/vp_userguide.html, fetched 2026-10-28
[S2] PIGJET/neon-raymarcher, https://github.com/PIGJET/neon-raymarcher, fetched 2026-10-27 (unofficial)
[S3] Computing and Displaying Isosurfaces inR, https://doi.org/10.18637/jss.v028.i01, fetched 2026-10-27
