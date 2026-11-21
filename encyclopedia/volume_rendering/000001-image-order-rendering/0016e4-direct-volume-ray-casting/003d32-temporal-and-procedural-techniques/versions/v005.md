# Temporal and Procedural Techniques

Also called: Procedural Volume Rendering, Dynamic Volume Techniques

## Overview
This family extends direct volume ray casting by replacing static, pre-computed scalar fields with procedural generation or temporal adaptation mechanisms. It solves the limitations of static data storage and fixed rendering performance by enabling dynamic effects (smoke, clouds), interactive geometric manipulation (displacement), and adaptive performance scaling (multi-frame rate) [S1]. It differs from static volume rendering by computing density or geometry at runtime rather than fetching it from a fixed 3D texture, and differs from neural methods by relying on analytical noise functions or heuristic performance models rather than learned latent representations.

## Shared principles
The method retains the core ray casting loop of [Direct volume ray casting](19-direct-volume-ray-casting.md) but modifies the sampling or traversal stages.

1.  **Procedural Density Evaluation**: Instead of a single texture lookup $V(\mathbf{p})$, density is computed via analytical functions (e.g., Worley noise, FBM) or dynamic simulation states [S1]. This allows for infinite resolution and animation without pre-computation overhead.
2.  **Dynamic Ray Modification**: Ray origins or directions may be altered based on temporal state (e.g., lens effects, warping) or geometric displacement, requiring custom intersection logic that accounts for non-perspective or non-orthogonal ray trajectories [S1].
3.  **Adaptive Sampling**: Step sizes or sample counts may vary based on procedural complexity or performance metrics, often utilizing early termination when transmittance is low or accumulated opacity is high [S1].
4.  **Temporal Consistency**: When rendering dynamic content, techniques must manage frame-to-frame coherence, either through noise animation offsets or by blending between multi-frame rate and single-frame rate rendering strategies [S1].

## Subtopics

### [Volumetric Smoke Rendering with Worley Noise and Light Maps](2447-volumetric-smoke-rendering-with-worley-noise-and-light-maps.md)
Uses 3D Worley noise for density and light maps for scattering to simulate dynamic smoke. Chosen for real-time animated smoke effects where static textures are insufficient.

### [GPU-accelerated ray casting with volumetric lenses](6767-gpu-accelerated-ray-casting-with-volumetric-lenses.md)
Modifies ray directions based on screen-space lens parameters for focus+context. Chosen when interactive magnification or distortion of specific volume regions is required.

### [Procedural Volumetric Cloud Rendering (GPU Shader)](8240-procedural-volumetric-cloud-rendering-gpu-shader.md)
Evaluates Worley noise FBM and Henyey-Greenstein scattering directly in the shader. Chosen for large-scale atmospheric effects where storage of 3D cloud textures is prohibitive.

### [Displacement Mapping for Volume Rendering](10659-displacement-mapping-for-volume-rendering.md)
Applies vector-based displacements to volumetric data via inverse mapping. Chosen for interactive cutting, peeling, or deforming of volumes without mesh remeshing.

### [Dynamic Multi-Frame Rate Rendering](11338-dynamic-multi-frame-rate-rendering.md)
Switches between image warping (MFR) and parallel raycasting (SFR) based on workload. Chosen to balance latency and fidelity in interactive applications with variable complexity.

## Background
The techniques originate from the need to render dynamic volumetric phenomena (smoke, clouds) and interactive volume manipulation in real-time graphics. Early procedural methods used analytical noise functions to approximate density fields [S1]. The integration of GPU compute shaders enabled efficient generation of these noise fields (e.g., Worley noise) and their integration into ray marching loops [S1]. The concept of volumetric lenses and displacement mapping extended ray casting to support view-dependent distortions and geometric modifications [S1]. Dynamic multi-frame rate rendering emerged to address the performance trade-offs in interactive volume visualization, leveraging empirical performance data to select optimal rendering strategies [S1].

## Sources

[S1] Unofficial notes on Temporal and Procedural Techniques, provided in the prompt context.
