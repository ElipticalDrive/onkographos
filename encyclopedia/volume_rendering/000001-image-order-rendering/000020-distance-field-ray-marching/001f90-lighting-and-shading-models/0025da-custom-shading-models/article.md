# Custom Shading Models

Also called: Custom Shading Routines, Per-voxel Shading Callbacks

## Overview
Custom Shading Models provide a mechanism for users to define arbitrary lighting computations per-voxel or per-hit-point, bypassing standard lookup tables or fixed-function pipelines. This family solves the problem of applying non-standard, physically based, or highly stylized lighting effects to volume-rendered or ray-marched geometry. It differs from sibling methods like [Phong Shading with SDF Normals](1977-phong-shading-with-sdf-normals.md) by allowing complete algorithmic freedom rather than restricting the user to a specific analytic model, and from [Distance Color Blending](791-distance-color-blending.md) by operating on lighting calculations rather than color accumulation.

## Shared principles
The core mechanism for implementing custom shading in volume rendering, as defined by the VolPack API, involves replacing the default shading lookup shader with a user-defined callback. The user registers this callback using `vpSetCallback` with the `VP_GRAY_SHADE_FUNC` identifier [S1]. When invoked, the callback function computes the color for the current voxel based on the scalar field value, normal, and lighting parameters [S1].

Key implementation details:
- **Callback Registration:** The user defines a function (e.g., `myshader`) and registers it via `vpSetCallback(vpc, VP_GRAY_SHADE_FUNC, myshader)` [S1].
- **Lookup Table Alternative:** Instead of callbacks, users can create custom routines to initialize shading lookup tables of any size, allowing the use of non-encoded normal vectors or color transfer functions [S1].
- **Performance:** Using callback functions can lead to significant performance degradation during rendering compared to lookup table methods [S1].
- **Depth Cueing:** An independent feature, depth cueing can be enabled via `vpEnable(vpc, VP_DEPTH_CUE, 1)` to darken distant voxels, assisting in foreground/background distinction [S1].

## Subtopics

### [GGX Microfacet BRDF](3715-ggx-microfacet-brdf.md)
A physically based specular model using GGX NDF, Schlick Fresnel, and Smith geometry. Chosen for energy conservation and long-tail highlights.

### [Voxel Shading Pipeline](3898-voxel-shading-pipeline.md)
A GPU-based pipeline combining Lambertian diffuse, rim lighting, indirect ambient, and GGX specular for voxel worlds, integrating shadow casting and emissive handling.

### [Blinn-Phong with Metallic Reflections](4252-blinn-phong-with-metallic-reflections.md)
Combines Blinn-Phong diffuse/specular with metallic Fresnel reflections and emissive contributions, using simplified power-law specular lobes.

### [Fast Shading for Voxel Engines](5606-fast-shading-for-voxel-engines.md)
A CPU-based algorithm modifying voxel palette indices directly based on neighbor occupancy, optimized for immediate visual feedback without ray tracing.

### Light Evaluation for SDF Faces
Applies lighting (diffuse, specular, ambient) to a face color based on normal and light direction [S2, unofficial]. The normal's Z component is set to its absolute value, and the light vector is normalized from the sine and cosine of `_LightAngle` [S2, unofficial]. The color is computed as `max(faceColor.rgb, 0) + GetSpecular(n, light) * faceColor.a`, then multiplied by `(1 - dot(n, light) * _Diffuse)` and lerped between `_Ambient` and `1` using `n.z * n.z` [S2, unofficial]. The final color is set to `float4(col, faceColor.a)` [S2, unofficial].

### Procedural Road Surface
A procedural surface shader for roads that differentiates between asphalt, curb/shoulder, and terrain based on lateral distance to the road centerline, including emissive lane markings [S3, unofficial]. The lateral distance `d` is calculated as `p.x - roadX(p.z)` with absolute value `ad` [S3, unofficial]. If `ad < ROAD_HALF_WIDTH`, albedo is dark gray, metal is 0.35, and emissive is 0.0; lane lines are calculated using `smoothstep` for edge lines, center lines, and dashed separators, adding emissive cyan color [S3, unofficial]. If `ad < ROAD_HALF_WIDTH + SHOULDER`, albedo is dark gray, metal is 0.1, and emissive is a magenta strip at the road edge [S3, unofficial]. Otherwise, albedo is very dark, metal is 0.0, and emissive is 0.0 [S3, unofficial]. Parameters include asphalt metal 0.35, lane line color `vec3(0.15, 1.0, 1.1) * 1.3`, edge line offset `ROAD_HALF_WIDTH - 0.3`, center line offset 0.18, and dashed line frequency `p.z * 0.18` [S3, unofficial].

## Background
The concept of custom shading in volume rendering originated with the VolPack library, which provided a framework for shading volumetric data [S1]. VolPack allowed users to either initialize custom shading lookup tables or define callback routines for per-voxel shading, offering flexibility beyond its default encoded normal vectors [S1]. This approach laid the groundwork for modern custom shading models in ray marching and voxel engines, where users implement specific lighting equations (such as GGX or Blinn-Phong variants) directly in shader code or CPU routines [S2, S3, unofficial].

## Sources
[S1] VolPack User's Guide, http://www-graphics.stanford.edu/software/volpack/vp_userguide.html, fetched 2026-10-28
[S2] andersblomqvist/unity-nanovdb-renderer, https://github.com/andersblomqvist/unity-nanovdb-renderer, fetched 2026-10-27 (unofficial)
[S3] PIGJET/neon-raymarcher, https://github.com/PIGJET/neon-raymarcher, fetched 2026-10-27 (unofficial)
