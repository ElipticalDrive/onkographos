# Temporal and Procedural Techniques

Also called: Procedural Volume Rendering, Temporal Accumulation (in context of noise generation)

## Overview
This family encompasses methods that generate volume density or optical properties algorithmically during rendering, rather than retrieving them from pre-computed scalar fields. It addresses the need for infinite detail, seamless animation, and reduced memory bandwidth by replacing texture lookups with on-the-fly noise evaluation (e.g., Worley noise, FBM) or temporal accumulation of stochastic samples. It differs from standard direct volume ray casting by decoupling the density definition from static voxel data, allowing for dynamic effects like smoke and clouds that are defined by mathematical functions [S1].

## Shared principles
All methods in this family replace the **Sampling and Interpolation** stage of the standard ray casting loop with a procedural evaluation function. Instead of $d_i = \text{TextureLookup}(\mathbf{p}_i)$, the density is computed as $d_i = \mathcal{N}(\mathbf{p}_i, t)$, where $\mathcal{N}$ is a noise function (e.g., Worley, Perlin) and $t$ is a time parameter for animation [S1].

The core computational engine remains the direct volume ray casting loop defined in [Direct volume ray casting](19-direct-volume-ray-casting.md), but with the following modifications:
1.  **Procedural Density Evaluation**: At each sample point $\mathbf{p}_i$, the density $\rho(\mathbf{p}_i)$ is calculated using a Fractal Brownian Motion (FBM) sum of noise octaves:
    $$
    \rho(\mathbf{p}_i) = \sum_{k=0}^{K-1} A_k \cdot \mathcal{N}_k(\mathbf{p}_i \cdot f_k)
    $$
    where $A_k$ is the amplitude, $f_k$ is the frequency, and $K$ is the number of octaves [S1].
2.  **Temporal Animation**: The noise coordinates are offset by a time-dependent vector $\mathbf{v}(t)$ to simulate flow:
    $$
    \mathbf{p}'_i = \mathbf{p}_i + \mathbf{v}(t)
    $$
    Commonly, $\mathbf{v}(t) = \mathbf{d}_{wind} \cdot t \cdot \text{speed}$ [S1].
3.  **Lighting Approximation**: Due to the lack of pre-computed light transport data, scattering is often approximated using analytic phase functions (e.g., Henyey-Greenstein) and simplified transmittance calculations from a light source [S1].

## Subtopics

### [Volumetric Smoke Rendering with Worley Noise and Light Maps](2447-volumetric-smoke-rendering-with-worley-noise-and-light-maps.md)
A real-time technique for smoke using Unity Compute Shaders for texture generation and fragment shaders for raymarching. It uses Worley noise for density detail and a separate light map for scattering. It is chosen when high-performance, animated smoke with god-ray effects is required without external asset dependencies. It differs from standard DVR by using procedural noise for density and a pre-computed light map for scattering, rather than volume density alone for shading.

### [Procedural Volumetric Cloud Rendering (GPU Shader)](8240-procedural-volumetric-cloud-rendering-gpu-shader.md)
This method renders clouds by evaluating a procedural noise field (Worley noise via FBM) directly within a GPU fragment shader. It replaces texture-based density lookups with on-the-fly noise generation and approximates light scattering using the Henyey-Greenstein phase function. It is distinguished by its ability to generate infinite detail and seamless animation, though it incurs higher computational cost per sample due to noise evaluation and secondary light-marching passes.

#### GPU-accelerated ray casting with volumetric lenses
A variant that integrates ray casting with lens effects (magnifier, angular, feature-based) for focus+context visualization. It calculates ray directions using specific lens rules, computes intersections with the volume bounding box, and traverses the volume using Cg fragment programs. It requires GPU features such as loops, early termination, and branches. The layout typically involves four passes: RayDirection, RayTfrontback, RayCasting, and Rendering. A key pitfall is that standard OpenGL front/back face drawing cannot be used because ray directions are not always orthogonal or perspective; custom intersection calculation is required [S1].

## Background
Procedural volume rendering emerged with the advent of programmable GPUs, which allowed complex noise functions to be evaluated in fragment shaders. Early implementations used simple noise for texture synthesis, but later techniques integrated FBM and Worley noise for realistic smoke and clouds. The "Magic Volume Lens" paper introduced GPU-accelerated ray casting with lens effects, demonstrating that complex ray traversal and intersection calculations could be performed efficiently on hardware like the NVIDIA GeForce 6800 [S1].

## Sources
[S1] The Magic Volume Lens: An Interactive Focus+Context Technique for ..., https://www3.cs.stonybrook.edu/~mueller/papers/lens_highres.pdf, fetched 2026-10-27
