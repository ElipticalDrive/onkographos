# Atmospheric scattering models
Also called: Atmospheric scattering, Planetary atmosphere rendering, Volumetric atmospheric rendering

## Overview
Atmospheric scattering models render the light interaction within a planetary atmosphere or volumetric cloud layer by solving the Radiative Transfer Equation (RTE) along view rays [S1, unofficial]. This family is distinguished from generic volume rendering by its reliance on analytical density profiles (e.g., exponential falloff) and specific phase functions (Rayleigh, Henyey-Greenstein) tailored to gaseous or particulate media, rather than arbitrary voxel densities [S1, unofficial]. It differs from billboard-based methods by providing true volumetric depth and internal shadowing .

## Shared principles
The family operates on the principle of discretizing the Radiative Transfer Equation (RTE) along a ray $r(t) = \mathbf{o} + t\mathbf{d}$, where $\mathbf{o}$ is the camera origin and $\mathbf{d}$ is the normalized direction vector [G]. The integration computes the accumulated color $C$ and transmittance $T$ by marching through a medium defined by density $\rho(\mathbf{p})$, scattering coefficient $\sigma_s$, and absorption coefficient $\sigma_a$ [G].

The core integration step for a segment of length $\Delta t$ is:
1.  **Density Evaluation**: Sample $\rho$ at the current position $\mathbf{p}$ [G].
2.  **Extinction**: Compute the extinction coefficient $\sigma_t = \sigma_s + \sigma_a$ [G].
3.  **Transmittance Update**: Update the ray's transmittance $T$ using Beer-Lambert law:
    $$ T \leftarrow T \cdot e^{-\sigma_t \rho \Delta t} $$ [G].
4.  **Inscattering**: Compute the light contributed by the segment. For single scattering, this involves the phase function $P(\theta)$ and the light source radiance $L_{source}$ attenuated by the optical depth to the light [G].
5.  **Accumulation**: Add the inscattered light to the accumulated color, weighted by the current transmittance:
    $$ C \leftarrow C + T \cdot \sigma_s \rho \Delta t \cdot L_{scattered} $$ [G].

The process terminates when $T$ falls below a threshold (e.g., $0.01$) or the ray exits the volume bounds [G].

## Subtopics

### [Ray Marching with Single Scattering](2057-ray-marching-with-single-scattering.md)
A real-time technique for rendering volumetric clouds by discretizing the RTE along view rays and approximating light transport with a single scattering event. It combines procedural noise for density generation with a physically-based lighting model to produce dynamic, three-dimensional cloudscapes. It is distinguished from multi-scattering methods by its computational efficiency, trading physical accuracy for performance, and from billboard-based methods by its ability to render volumetric depth and internal structure. See [Ray Marching with Single Scattering](2057-ray-marching-with-single-scattering.md).

### [Atmospheric Ray Marching](4874-atmospheric-ray-marching.md)
A technique for rendering planetary atmospheres using single scattering, exponential density falloff, and the Rayleigh phase function. It solves the RTE for a spherical atmosphere by marching along view rays and light rays to compute inscattered sunlight and shadowing. It is distinguished from [Ray Marching with Single Scattering](2057-ray-marching-with-single-scattering.md) by its specific application to planetary spheres with analytical density profiles and its handling of shadow rays for occlusion. See [Atmospheric Ray Marching](4874-atmospheric-ray-marching.md).

### Atmosphere and Cloud Rendering
A unified rendering implementation for volumetric clouds and atmosphere using raymarching as the main method [S1, unofficial]. It renders the atmosphere using an approach with multiple precomputed Look-up tables (LUTs) proposed by Sébastien Hillaire, and volumetric clouds using a pre-computed three-dimensional texture setup storing procedurally generated noise described by Andrew Schneider [S1, unofficial]. Images are rendered in high dynamic range before applying post-processing effects and using adaptive luminance to transform the image into LDR for presentation [S1, unofficial].

### Octree-Based LOD Lightning Scattering
A method where atmospheric scattering effects are precomputed and stored, using an octree-based LOD to provide cloud density for lightning scattering [S2].

## Background
The unified approach of combining atmosphere LUTs and cloud noise textures was presented by Matěj Sakmary in a 2022 bachelor thesis [S1, unofficial]. The underlying survey of cloud modeling and rendering techniques, including precomputation methods, was published in a 2020 survey [S2].

## Sources
[S1] Atmosphere and Cloud Rendering in Real-time | DCGI, https://dcgi.fel.cvut.cz/en/theses/2022/sakmamat/, fetched 2026-10-30 (unofficial)
[S2] A survey of modeling, rendering and animation of clouds in computer graphics, https://doi.org/10.1007/s00371-020-01953-y, fetched 2026-10-26
