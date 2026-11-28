# Temporal and Procedural Techniques

Also called: Procedural Volume Rendering, Temporal Accumulation (General)

## Overview
This family encompasses techniques that generate volume data or animation dynamically during rendering, rather than sampling pre-computed scalar fields. It solves the problem of rendering infinite-resolution, dynamically animated, or computationally expensive volumetric effects (such as smoke, clouds, or procedural noise) without the memory overhead of storing large 3D textures. It differs from standard direct volume rendering by replacing the static volume lookup with procedural noise evaluation (e.g., Worley noise, FBM) and often integrates temporal components for animation or accumulation, as seen in [Volumetric Smoke Rendering with Worley Noise and Light Maps](2447-volumetric-smoke-rendering-with-worley-noise-and-light-maps.md) and [Procedural Volumetric Cloud Rendering (GPU Shader)](8240-procedural-volumetric-cloud-rendering-gpu-shader.md).

## Shared principles
The core mechanism replaces the scalar field $V(\mathbf{p})$ with a procedural function $P(\mathbf{p}, t)$, where $\mathbf{p}$ is the sample position and $t$ is time. The rendering pipeline remains identical to [Direct volume ray casting](19-direct-volume-ray-casting.md): ray generation, intersection, and compositing. However, the sampling step invokes a noise generator (e.g., Worley noise, Perlin noise) instead of a texture sampler.

For procedural density $\rho(\mathbf{p})$, the value is often constructed using Fractal Brownian Motion (FBM):
$$
\rho(\mathbf{p}) = \sum_{i=0}^{N-1} A_i \cdot \mathcal{N}(f_i \cdot \mathbf{p})
$$
where $\mathcal{N}$ is the base noise function, $A_i$ is the amplitude, and $f_i$ is the frequency for octave $i$ [S1].

Temporal animation is achieved by offsetting the noise coordinates with a time variable $t$:
$$
\mathbf{p}' = \mathbf{p} + \mathbf{v} \cdot t
$$
where $\mathbf{v}$ is a wind or flow vector [S1].

## Subtopics

### [Volumetric Smoke Rendering with Worley Noise and Light Maps](2447-volumetric-smoke-rendering-with-worley-noise-and-light-maps.md)
A real-time technique using Unity Compute Shaders to generate Worley noise textures and a vertex/fragment shader for raymarching. It distinguishes itself by combining Worley noise for density with a separate light map to simulate scattering and god rays, rather than relying solely on density for shading. It uses a merged texture containing noise, detail, light, and mask channels. Choose this for real-time smoke effects requiring dynamic lighting interaction.

### [Procedural Volumetric Cloud Rendering (GPU Shader)](8240-procedural-volumetric-cloud-rendering-gpu-shader.md)
This method ray-marches a procedural noise field (Worley noise combined with FBM) within a camera-space bounding box on the GPU. It differs from texture-based rendering by generating density entirely from 3D noise functions, allowing infinite resolution and dynamic wind animation without texture streaming. It implements a two-pass march: one for sun transmittance (light marching) and one for camera view (density marching), using the Henyey-Greenstein phase function for scattering. Choose this for high-quality, infinite-resolution cloud rendering.

### GPU-accelerated ray casting with volumetric lenses
A variant that applies lens effects (magnifier, angular, feature-based) to volume rendering by altering ray directions.
**Step:** Calculate ray directions using lens rules (feature-based, sampling-rate-based, or angular). Compute intersection points with the volumetric object's bounding box. Implement a ray traversal algorithm for a given step size, accessing volume data (density, gradient, or color) stored in 3D textures. Translate algorithms into Cg fragment programs [S1].
**Parameter:** Requires GPU features: loop, early termination, and branches [S1].
**Layout:** Standard four-pass fragment program structure for magnifier/angular lenses: Pass 1 (RayDirection), Pass 2 (RayTfrontback), Pass 3 (RayCasting), Pass 4 (Rendering). Feature-based lens adds Pass 1+ (RayLensBorder) before Pass 2. Sampling-rate-based lens replaces Pass 1 with Pass 1* (RayStartPoints) [S1].
**Pitfall:** Standard OpenGL front/back face drawing for ray start/end points cannot be used because ray directions are not always orthogonal or perspective; custom intersection calculation is required [S1].

## Background
The concept of procedural volume rendering emerged with the need to render complex natural phenomena like clouds and smoke without storing massive 3D texture arrays. Early work by Reeves et al. introduced particle systems for smoke, while later GPU advancements enabled real-time ray marching of procedural noise fields. The "Magic Volume Lens" paper [S1] specifically addresses interactive focus+context techniques using GPU-accelerated ray casting with volumetric lenses, leveraging early GPU shader capabilities (loops, branches) to implement complex ray traversal and lens effects in Cg.

## Sources
[S1] The Magic Volume Lens: An Interactive Focus+Context Technique for ..., https://www3.cs.stonybrook.edu/~mueller/papers/lens_highres.pdf, fetched 2026-10-27
