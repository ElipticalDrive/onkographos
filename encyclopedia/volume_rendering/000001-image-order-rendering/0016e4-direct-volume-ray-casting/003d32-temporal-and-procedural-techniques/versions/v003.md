# Temporal and Procedural Techniques

Also called: Procedural Volume Rendering, Real-time Volumetric Effects

## Overview
This family covers methods that generate volumetric density and lighting data procedurally or via temporal accumulation, rather than relying on pre-computed voxel grids or static simulation outputs. It solves the memory and storage bottlenecks of large-scale volumetric data by evaluating noise functions or accumulating frames on the fly. It differs from standard direct volume ray casting by replacing the static volume texture lookup with on-the-fly procedural evaluation (e.g., Worley noise, FBM) or temporal history buffers, enabling real-time rendering of dynamic effects like smoke and clouds without pre-baking [S1, unofficial].

## Shared principles
All methods in this family share the core ray casting loop defined in [Direct volume ray casting](19-direct-volume-ray-casting.md) but replace the static sampling stage with procedural or temporal logic.

1.  **Procedural Density Evaluation**: Instead of sampling a 3D texture $V(\mathbf{p})$, the density $\rho(\mathbf{p})$ is computed via a mathematical function $F(\mathbf{p}, \mathbf{t})$, where $\mathbf{t}$ is time. Common functions include Worley noise for cellular structures (smoke/clouds) or FBM for fractal details [S1, unofficial].
2.  **On-the-Fly Light Transport**: Lighting is often approximated analytically or via secondary ray marches (e.g., sun transmittance) rather than pre-integrated light maps, though hybrid approaches exist [S1, unofficial].
3.  **Temporal Stability**: For dynamic scenes, methods may use temporal accumulation to stabilize noise or reuse previous frame data, though this article's children focus primarily on procedural spatial generation [G].

## Subtopics

### [Volumetric Smoke Rendering with Worley Noise and Light Maps](2447-volumetric-smoke-rendering-with-worley-noise-and-light-maps.md)
Uses Worley noise for density and optional light maps for scattering. It differs by combining procedural noise with pre-baked illumination data for god rays. Choose for real-time smoke with high visual fidelity.

### [GPU-accelerated ray casting with volumetric lenses](6767-gpu-accelerated-ray-casting-with-volumetric-lenses.md)
Not given in the sources.

### [Procedural Volumetric Cloud Rendering (GPU Shader)](8240-procedural-volumetric-cloud-rendering-gpu-shader.md)
Evaluates FBM Worley noise directly in the shader without light maps. It differs by using analytic Henyey-Greenstein scattering and simplified transmittance marches. Choose for real-time clouds without pre-computed textures.

### [Displacement Mapping for Volume Rendering](10659-displacement-mapping-for-volume-rendering.md)
Applies discontinuous geometric displacements to the volume domain. It differs by modifying the spatial coordinates of the volume data itself rather than just density values. Choose for interactive deformation (cuts, peeling) of volumetric objects.

## Background
Procedural volume rendering emerged to address the memory limitations of storing high-resolution volumetric datasets for real-time applications. Early methods relied on static voxel grids, but the need for dynamic effects like smoke and clouds led to the integration of procedural noise functions (e.g., Worley, Perlin) directly into the rendering pipeline [S1, unofficial]. The use of GPU compute shaders and fragment shaders has enabled these techniques to run in real-time, often hybridizing procedural generation with traditional ray casting [S1, unofficial].

## Sources
[S1] Unity Volumetric Smoke and Cloud Rendering Implementation Details, URL, 2023-10-27
