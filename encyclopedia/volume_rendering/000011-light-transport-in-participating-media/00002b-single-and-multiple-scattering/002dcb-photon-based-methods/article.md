# Photon-based Methods

Also called: Photon Mapping for Volumes, Photon Beams, Instant Radiosity for Volumes

## Overview
Photon-based methods solve the problem of estimating radiance in participating media by decoupling light transport into a preprocessing phase (photon tracing) and a rendering phase (radiance estimation via density estimation or virtual light sources) [S1, S2]. Unlike direct integration methods that trace paths per pixel, these methods store photon interactions in spatial structures and estimate incident radiance at query points using kernel density estimation or explicit summation [S2]. What sets this family apart from its siblings is the specific use of stored photon data (points, beams, or surfaces) to approximate the integral of the radiance transfer equation, differing from the explicit path tracing of [Compute Shader Photon Integration](2111-compute-shader-photon-integration.md) and the boundary-segment sampling of [Grid-based Importance Sampling for Boundary Segments](7643-grid-based-importance-sampling-for-boundary-segments.md).

## Shared principles
All methods in this family are derived from the conservation of energy for a differential cylinder of light passing through a medium [G]. They share the following core definitions and mechanisms:

**1. Fundamental Interactions**
Any volumetric effect is composed of three fundamental characteristics [G]:
*   **Absorption:** The loss of radiant energy along a ray due to energy conversion (e.g., to heat). It is described by the absorption coefficient $\sigma_a$ (units: $m^{-1}$) [G].
*   **Emission:** The addition of radiant energy where the medium is luminous (e.g., flames). It is described by the emissive term $L_e(\mathbf{p}, \boldsymbol{\omega})$, representing radiance emitted per unit length [G].
*   **Scattering:** The redirection of light rays upon collision with particles. It is split into:
    *   *In-scattering:* Light from other directions reflected into the view ray [G].
    *   *Out-scattering:* Light from the view ray reflected away [G].
In most lighting calculations, out-scattering is folded into the **extinction** term because its net effect on the view ray is identical to absorption [G].

**2. Notation**
*   $\mathbf{p}$: Position of interaction.
*   $\boldsymbol{\omega}$: Ray direction (unit vector).
*   $L$: Radiance quantity.
*   $L_i$: Incoming radiance.
*   $L_o$: Outgoing radiance.
*   $\sigma_t$: Total extinction coefficient, $\sigma_t = \sigma_a + \sigma_s$ [G].

**3. Differential Forms**
The change in radiance $dL$ over a differential path length $dt$ is governed by the balance of loss and gain terms [G]:
*   **Absorption Model:** $dL = -\sigma_a L_i dt$ [G].
*   **Emission Model:** $dL = \sigma_e dt$ (where $\sigma_e$ is the emission coefficient) [G].
*   **General RTE:** Combines extinction (loss) and source terms (gain from emission and in-scattering) [G].

**4. Transmittance**
The fraction of light surviving transmission through a medium is defined by the Beer-Lambert law [S3]:
$$ T = e^{-\tau} $$
where $\tau$ is the optical depth (integrated extinction) [S3].

## Subtopics

### [Photon-based Estimation](2521-photon-based-estimation.md)
Photon-based Estimation methods solve the problem of estimating radiance in participating media by decoupling light transport into a preprocessing phase (photon tracing) and a rendering phase (radiance estimation via density estimation). Unlike direct integration methods that trace paths per pixel, these methods store photon interactions in spatial structures and estimate incident radiance at query points using kernel density estimation. What sets this family apart from its siblings is the specific use of stored photon data (points, beams, or surfaces) to approximate the integral of the radiance transfer equation, differing from the explicit path tracing of [Compute Shader Photon Integration](2111-compute-shader-photon-integration.md) and the boundary-segment sampling of [Grid-based Importance Sampling for Boundary Segments](7643-grid-based-importance-sampling-for-boundary-segments.md).

### [Photon Beam Methods](4976-photon-beam-methods.md)
Photon Beam Methods are a family of density estimation techniques for light transport in participating media that replace discrete photon points with continuous, directional beam segments. Unlike standard photon tracing which deposits energy at discrete interaction points, these methods model light as extended cylinders, allowing for analytical integration of radiance over beam-ray intersections. This approach reduces variance compared to point-based density estimation and eliminates the quantization errors associated with grid-based diffusion methods.

### [Grid-based Importance Sampling for Boundary Segments](7643-grid-based-importance-sampling-for-boundary-segments.md)
This method provides a preprocessing-based importance sampling strategy for boundary path integrals in differentiable rendering. It addresses the difficulty of analytically computing the probability density $P(\mathbf{x}_B, \boldsymbol{\omega}_B)$ by discretizing it into a regular 3D grid. Unlike standard path guiding which accelerates unidirectional path tracing, this technique is specifically designed for the boundary integrals required for gradient computation. It builds on [Photon-based Methods](5620-photon-based-methods.md) by applying photon/importon map concepts to the low-dimensional boundary space.

### [Instant Radiosity / Virtual Point Lights (VPLs)](9319-instant-radiosity-virtual-point-lights-vpls.md)
Instant Radiosity, also known as Virtual Point Lights (VPLs), is a method for computing indirect illumination that approximates the global illumination solution by treating stored photons as discrete light sources rather than using density estimation. It solves the problem of efficiently calculating diffuse interreflection and caustics on surfaces by replacing the complex integral of incoming radiance with a summation of contributions from a smaller set of virtual point lights. Unlike sibling methods such as Photon-based Estimation, which rely on kernel density estimation to smooth photon contributions, VPLs require explicit visibility computation for each light, resulting in significantly fewer samples but introducing singularity artifacts at close distances.

### Multiple Scattering with Photon Marching
Photon marching extends photon beam tracing to handle multiple scattering events by spawning new discrete beams from existing ones [S1]. This technique allows for the simulation of complex light transport paths within participating media without requiring full path tracing for every pixel.

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

## Background
Photon mapping for participating media was introduced by Jensen et al. in 2006, extending the original photon mapping technique for surfaces to volumetric effects [S1]. The method was developed to address the high variance and computational cost of direct integration methods for multiple scattering in complex media. It has since been extended to photon beams and various density estimation techniques to improve efficiency and accuracy in rendering participating media.

## Sources
[S1] A comprehensive theory of volumetric radiance estimation using photon points and beams, https://doi.org/10.1145/1899404.1899409, fetched 2026-10-26
[S2] Photon mapping for participating media, https://doi.org/10.1145/1179298.1141920, fetched 2026-10-26
[S3] Physically Based Rendering: From Theory to Implementation, https://www.pbr-book.org/, fetched 2026-10-26
