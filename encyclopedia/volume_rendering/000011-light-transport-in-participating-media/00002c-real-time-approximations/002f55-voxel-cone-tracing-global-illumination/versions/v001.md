# Voxel Cone Tracing Global Illumination

Also called: Crassin's voxel cone tracing

## Overview
Voxel Cone Tracing Global Illumination is a real-time specular global illumination technique that leverages voxel cone tracing to approximate glossy reflections [S1, unofficial]. It distinguishes itself from diffuse voxel GI applications by requiring only a single cone trace per glossy material, thereby significantly lowering performance costs [S1, unofficial].

## How it works
Inputs: Voxelized scene representation (typically a trilinearly interpolated 3D texture or sparse voxel octree), material properties (diffuse/albedo, specular color, roughness), and camera/view parameters.
Outputs: Final pixel color with specular global illumination contributions.

Data structures:
- Voxel Grid: A 3D texture or sparse structure storing radiance values. Each voxel contains color/intensity data representing the accumulated light in that volume [S1, unofficial].
- Material Buffer: Per-pixel data including roughness, which determines the cone's opening angle.

Steps:
1. For each pixel with a glossy material, determine the reflection direction based on the view vector and surface normal.
2. Calculate the cone's opening angle based on the material's roughness parameter. Higher roughness results in a wider cone [S1, unofficial].
3. Perform a single cone trace along the reflection direction through the voxel grid [S1, unofficial].
4. Sample the voxel data along the cone's path, accumulating radiance contributions weighted by the cone's intersection with the voxels [S1, unofficial].
5. Combine the accumulated specular radiance with the base color to produce the final pixel color [S1, unofficial].

Parameters:
- Roughness: Controls the cone width. Not given in the sources.
- Cone Trace Steps: Number of samples or distance intervals for the trace. Not given in the sources.

Edge cases and pitfalls:
- Highly polished reflectors: When dealing with highly polished (low roughness) reflectors, the underlying voxel representation of the scene might be revealed due to insufficient sampling or voxel grid resolution [S1, unofficial]. However, this rarely happens in practice [S1, unofficial].

## Operating characteristics
Time and memory complexity:
- Time: Lower than diffuse voxel GI because only a single cone trace is required per glossy material, compared to multiple traces or integration steps for diffuse [S1, unofficial].
- Memory: Depends on the voxel grid resolution and storage format (dense vs. sparse). Not given in the sources.

Measured performance:
- Not given in the sources.

Scaling:
- Performance scales with voxel grid resolution and the number of glossy pixels. Not given in the sources.

Strengths and limitations:
- Strengths: Lower performance cost for specular GI compared to diffuse GI approaches; suitable for real-time applications [S1, unofficial].
- Limitations: Potential visibility of voxel artifacts on highly polished surfaces [S1, unofficial].

When to use it and when not:
- Use for real-time specular global illumination in scenes with glossy materials where performance is critical [S1, unofficial].
- Avoid for highly polished, mirror-like surfaces if voxel artifacts are unacceptable, though this is rare [S1, unofficial].

## Background
Origin: The Voxel Cone Tracing global illumination technique was proposed by Cyril Crassin [S2, unofficial].
History: It was introduced as an alternative to localized environment maps, which suffer from insufficient spatial density and crude shape approximations [S1, unofficial]. Planar reflections were also considered as an alternative but are prohibitive due to the requirement of additional render passes for each mirror surface, especially with recursive reflections [S1, unofficial].
Usage: It is used in real-time rendering engines to provide efficient specular global illumination [S2, unofficial].

## Subtopics

### Voxel Cone Tracing Anisotropic Filtering
What it is: An extension that applies anisotropic filtering to cone traces to handle anisotropic materials.
What sets it apart: Handles directional roughness, unlike isotropic cone tracing.
When to choose: For materials with anisotropic reflections (e.g., brushed metal).
Link: [Voxel Cone Tracing Anisotropic Filtering](2552-voxel-cone-tracing-anisotropic-filtering.md)

### Customized Cone Tracing
What it is: A variant that allows custom cone shapes or tracing strategies.
What sets it apart: Flexibility in cone definition compared to standard isotropic cones.
When to choose: For specific material requirements not covered by standard cones.
Link: [Customized Cone Tracing](3447-customized-cone-tracing.md)

### Voxel Cone Tracing for Ambient Occlusion
What it is: Uses cone tracing to approximate ambient occlusion in voxel grids.
What sets it apart: Focuses on occlusion rather than global illumination.
When to choose: For efficient AO in voxel-based scenes.
Link: [Voxel Cone Tracing for Ambient Occlusion](3544-voxel-cone-tracing-for-ambient-occlusion.md)

### Voxel Shading via Convolution
What it is: Shading method using convolution operations on voxel data.
What sets it apart: Uses convolution instead of ray/cone tracing.
When to choose: For smooth, blurred lighting effects.
Link: [Voxel Shading via Convolution](4778-voxel-shading-via-convolution.md)

### Anisotropic Voxel Representation
What it is: Voxel representation that stores directional information.
What sets it apart: Captures anisotropy in the voxel data itself.
When to choose: For complex anisotropic lighting effects.
Link: [Anisotropic Voxel Representation](4963-anisotropic-voxel-representation.md)

### Cone Tracing
What it is: General technique of tracing cones through a volume.
What sets it apart: The foundational method for voxel cone tracing.
When to choose: As a base for various voxel-based lighting techniques.
Link: [Cone Tracing](5679-cone-tracing.md)

## Sources
[S1] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
[S2] domme/VoxelConeTracing, https://github.com/domme/VoxelConeTracing, fetched 2026-10-26 (unofficial)
