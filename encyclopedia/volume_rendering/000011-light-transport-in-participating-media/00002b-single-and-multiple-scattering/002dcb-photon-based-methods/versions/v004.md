# Photon-based Methods


## Overview
Photon-based methods solve the problem of estimating radiance in participating media by decoupling light transport into a preprocessing phase (photon tracing) and a rendering phase (radiance estimation via density estimation or virtual light sources) [S1]. Unlike direct integration methods that trace paths per pixel, these methods store photon interactions in spatial structures and estimate incident radiance at query points using kernel density estimation or explicit summation [G]. What sets this family apart from its siblings is the specific use of stored photon data (points, beams, or surfaces) to approximate the integral of the radiance transfer equation, differing from the explicit path tracing of [Compute Shader Photon Integration](2111-compute-shader-photon-integration.md) and the boundary-segment sampling of [Grid-based Importance Sampling for Boundary Segments](7643-grid-based-importance-sampling-for-boundary-segments.md) [G].

## Shared principles
Inherits core definitions and mechanisms from [Single and multiple scattering](35-single-and-multiple-scattering.md).

## Subtopics

### [Photon-based Estimation](2521-photon-based-estimation.md)
Solves the problem of computing realistic in-scattering and indirect illumination by decoupling light transport simulation from final image synthesis. Unlike direct integration methods, it stores photon interactions in a spatial data structure and estimates radiance via density estimation or direct contribution summation. Distinguished from [Photon Beam Mapping](7874-photon-beam-mapping.md) and [Adaptive Beam Tracing for Volume Caustics](8406-adaptive-beam-tracing-for-volume-caustics.md) by using discrete point-based representations, and from [Instant Radiosity / Virtual Point Lights (VPLs)](9319-instant-radiosity-virtual-point-lights-vpls.md) by explicitly tracking photon paths through scattering events rather than approximating them as point light sources.

### [Grid-based Importance Sampling for Boundary Segments](7643-grid-based-importance-sampling-for-boundary-segments.md)
Provides a preprocessing-based importance sampling strategy for boundary path integrals in differentiable rendering. It addresses the difficulty of analytically computing the probability density $P(\mathbf{x}_B, \boldsymbol{\omega}_B)$ by discretizing it into a regular 3D grid. Unlike standard path guiding, it is specifically designed for the boundary integrals required for gradient computation. It builds on [Photon-based Methods](5620-photon-based-methods.md) by applying photon/importon map concepts to the low-dimensional boundary space.

### [Photon Beam Mapping](7874-photon-beam-mapping.md)
Replaces discrete photon points with continuous photon beams to create a more compact representation of light transport. By storing light transport as beams rather than points, it achieves faster convergence for media such as fog compared to standard photon mapping. It operates in two phases: a lighting pass that stores light transport as photon beams, and a rendering pass that uses a beam-to-beam estimator to gather contributions.

### [Adaptive Beam Tracing for Volume Caustics](8406-adaptive-beam-tracing-for-volume-caustics.md)
A real-time algorithm for rendering surface and volume caustics in single-scattering participating media. It solves the problem of efficiently capturing high-frequency light transport patterns formed by specular reflections and refractions without the prohibitive cost of path tracing. Unlike sibling methods which trace individual rays or photons, this method traces volumetric beams of light. It differs from [Grid-based Importance Sampling for Boundary Segments](7643-grid-based-importance-sampling-for-boundary-segments.md) by generating beams adaptively from a projected grid in light space, refining resolution based on geometric discontinuities and beam contribution, thereby reducing fill-rate in defocused regions.

### [Instant Radiosity / Virtual Point Lights (VPLs)](9319-instant-radiosity-virtual-point-lights-vpls.md)
Computes indirect illumination by approximating the global illumination solution by treating stored photons as discrete light sources rather than using density estimation. It solves the problem of efficiently calculating diffuse interreflection and caustics on surfaces by replacing the complex integral of incoming radiance with a summation of contributions from a smaller set of virtual point lights. Unlike sibling methods such as Photon-based Estimation, which rely on kernel density estimation to smooth photon contributions, VPLs require explicit visibility computation for each light, resulting in significantly fewer samples but introducing singularity artifacts at close distances.

### Multiple Scattering with Photon Marching
Extends photon beam tracing to handle multiple scattering events by spawning new discrete beams from existing ones [S1].
#### step
1. For a discrete beam $b_0$, choose a random propagation distance $t_s$ for the scattering event location [S1].
2. In homogeneous media, compute $t_s = -\log(1-\xi)/\sigma_t$ [S1].
3. In heterogeneous media, use inversion method or delta-tracking [S1].
4. Initiate a new discrete beam $b_1$ with origin $x_{b1} = x_{b0} + t_s \tilde{\omega}_{b0}$, direction $\tilde{\omega}_{b1}$ sampled from $f(\theta)$, and starting power $\Phi_{b1} = (\sigma_s/\sigma_t) \Phi_{b0}$ [S1].
5. Continue photon marching of the parent beam $b_0$ to the media's boundary [S1].
#### formula
The scattering distance $t_s$ is determined by the probability density function $pdf(t_s) = \sigma_t e^{-\sigma_t t_s}$, leading to the inverse transform sampling formula:
$$ t_s = -\frac{\log(1-\xi)}{\sigma_t} $$
where $\xi$ is a uniform random number in $[0, 1)$ and $\sigma_t$ is the total extinction coefficient [S1].
#### parameter
The scattering albedo $\sigma_s/\sigma_t$ is used for Russian roulette [S1]. If the scattering event is accepted, the starting power of the new beam simplifies to $\Phi_{b1} = \Phi_{b0}$ in grey media [S1].

### Lattice-Boltzmann Diffusion
A new technique based on the lattice-Boltzmann method able to deal with diffusion processes involved in participating medium illumination problems [S2]. It uses a simple grid-based photon transport model able to handle complex boundary conditions [S2]. No interaction between light and surfaces is considered [S2].

## Background
Not given in the sources.

## Sources
[S1] A comprehensive theory of volumetric radiance estimation using photon points and beams, https://doi.org/10.1145/1899404.1899409, fetched 2026-10-26
[S2] A survey on participating media rendering techniques, https://doi.org/10.1007/s00371-005-0287-1, fetched 2026-10-26
