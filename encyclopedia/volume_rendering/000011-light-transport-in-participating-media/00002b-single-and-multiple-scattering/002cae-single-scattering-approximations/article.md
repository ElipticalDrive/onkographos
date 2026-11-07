# Single Scattering Approximations

Also called: Single-scattering, First-order scattering, Direct in-scattering

## Overview
Single scattering approximations compute the radiance contribution of light that undergoes exactly one scattering event within a participating medium before reaching the camera [S2, book][S3]. This method solves the problem of rendering volumetric effects like fog, smoke, or clouds with direct illumination by neglecting multiple scattering events, which are assumed to be negligible due to low albedo or density [S1, unofficial]. It is distinguished from [Multiple Importance Sampling for Volumetric Path Tracing](1167-multiple-importance-sampling-for-path-tracing.md) and [Unified Points, Beams, and Paths (UPBP)](1402-unified-points-beams-and-paths-upbp.md) by ignoring the glow and higher-order interactions that characterize dense media, and from [Homogeneous Medium Sampling](980-homogeneous-medium-sampling.md) by explicitly handling the integration of in-scattered light from scene geometry or light sources rather than just distance sampling.

## Shared principles
All methods in this family rely on the simplification of the Radiative Transfer Equation (RTE) by truncating the scattering series after the first term [S2, book][S3]. The outgoing radiance $L_o(\mathbf{o}, \boldsymbol{\omega})$ along a ray from origin $\mathbf{o}$ in direction $\boldsymbol{\omega}$ is computed as the sum of the attenuated background radiance and the in-scattered radiance from the first scattering event [S2, book][S3]:

$$ L_o(\mathbf{o}, \boldsymbol{\omega}) = T(\mathbf{o}, \mathbf{p}_{far}) L_{bg} + \int_{\mathbf{o}}^{\mathbf{p}_{far}} T(\mathbf{o}, \mathbf{x}) \sigma_s(\mathbf{x}) L_{in}(\mathbf{x}, \boldsymbol{\omega}) d\mathbf{x} $$

where:
*   $T(\mathbf{o}, \mathbf{x}) = \exp\left(-\int_{\mathbf{o}}^{\mathbf{x}} \sigma_t(\mathbf{u}) d\mathbf{u}\right)$ is the transmittance from the eye to point $\mathbf{x}$ [S3].
*   $\sigma_s(\mathbf{x})$ is the scattering coefficient [G].
*   $L_{in}(\mathbf{x}, \boldsymbol{\omega})$ is the incident radiance at $\mathbf{x}$ from the light source(s), attenuated by the path from the light to $\mathbf{x}$ [S1, unofficial].
*   $L_{bg}$ is the background radiance [S1, unofficial].

The incident radiance $L_{in}$ typically involves the phase function $p(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o)$ and the geometric attenuation from the light source [S1, unofficial].

## Subtopics

### [Single-scattering homogeneous volume rendering](1135-single-scattering-homogeneous-volume-rendering.md)
This method renders a participating medium by solving the RTE under the assumption that light scatters exactly once within a homogeneous volume before reaching the camera. It extends a basic absorption-only model by adding in-scattering from surface-emitting geometry, assuming the surfaces do not reflect or transmit light. It is distinguished from [Single-scattering atmosphere rendering](3289-single-scattering-atmosphere-rendering.md) by its general applicability to arbitrary scene manifolds rather than specific atmospheric layers, and from [Volume Single Scattering (Monte Carlo with Point Light)](2293-volume-single-scattering-monte-carlo-with-point-light.md) by its integration over the entire scene manifold $\mathcal{M}$ rather than discrete point lights.

### [Volume Single Scattering (Monte Carlo with Point Light)](2293-volume-single-scattering-monte-carlo-with-point-light.md)
This method computes the radiance contribution of a single scattering event in a participating medium by integrating the volume rendering integral using Monte Carlo techniques with uniform distance sampling along the view ray. It specifically handles a point light source for illumination and a background surface for the unscattered component. Unlike deterministic ray marching approaches that sum contributions over fixed steps, this method uses stochastic sampling to estimate the integral, allowing for flexible handling of complex lighting and visibility, though it requires multiple samples to converge.

### [Single-Scattering Atmosphere Rendering](3289-single-scattering-atmosphere-rendering.md)
Renders the sky and atmospheric scattering by numerically integrating single scattering along a ray through a spherical atmosphere shell, accumulating Rayleigh and Mie contributions weighted by view and light transmittance. It solves the RTE for a single distant light source (the sun), accounting for Rayleigh scattering (gas molecules), Mie scattering (aerosols), and Ozone absorption. Unlike [Single-scattering homogeneous volume rendering](1135-single-scattering-homogeneous-volume-rendering.md) which assumes constant density, this method models the exponential density decay of a planetary atmosphere and the resulting wavelength-dependent color shifts (e.g., red sunsets).

### [Microflake Model](4739-microflake-model.md)
The Microflake Model is a physically-based participating medium model that represents a medium as a homogeneous cloud of microscopic flakes, where orientations are statistically described by a Normal Distribution Function (NDF) $D(\boldsymbol{\omega})$. It solves the problem of accurately simulating anisotropic light transport in media like fur, hair, or paper, where standard isotropic models fail to capture direction-dependent attenuation and scattering. Unlike isotropic single-scattering approximations, the Microflake Model produces anisotropic attenuation coefficients and direction-dependent phase functions that satisfy Helmholtz's reciprocity principle by construction.

### [Heterogeneous In-scattering with Multiple Scattering](4934-heterogeneous-in-scattering-with-multiple-scattering.md)
This method computes the radiance contribution from light scattering within a heterogeneous participating medium, handling both direct lighting from light sources and recursively computing indirect lighting (multiple scattering) up to a maximum depth. It differs from its siblings, such as [Single-scattering homogeneous volume rendering](1135-single-scattering-homogeneous-volume-rendering.md) and [Volume Single Scattering (Monte Carlo with Point Light)](2293-volume-single-scattering-monte-carlo-with-point-light.md), by explicitly supporting spatially varying density and coefficients (heterogeneity) and approximating multiple scattering events through recursive ray casting, rather than relying solely on single-scattering approximations or homogeneous assumptions.

### [Single Scattering Approximation](5264-single-scattering-approximation.md)
The Single Scattering Approximation simplifies the volume rendering integral by assuming that the medium has low albedo and/or low density, making multiple scattering events negligible. It solves the problem of computing in-scattered light by considering only the first scattering event, ignoring glow and higher-order interactions. It differs from [Single-scattering homogeneous volume rendering](1135-single-scattering-homogeneous-volume-rendering.md) by handling heterogeneous media via precomputed shadowing, and from [Single Scattering](5566-single-scattering.md) by explicitly detailing the precomputation of shadowing effects using texture mapping hardware for gridded data.

### [Single Scattering](5566-single-scattering.md)
Single Scattering is an approximation of light transport in participating media where in-scattered radiance is assumed to originate directly from light sources, ignoring light that has scattered more than once. It solves the problem of rendering volumetric effects like fog or smoke with direct illumination by integrating the single-scattering term of the volume rendering integral over the spherical domain of incoming light directions. Unlike [Single Scattering Approximation](5264-single-scattering-approximation.md) which may use broader approximations for the integral, this method explicitly handles the spherical integration via Next Event Estimation (NEE) to compute direct lighting contributions.

### [Real-Time Volumetric Shadows using 1D Min-Max Mipmaps](9426-real-time-volumetric-shadows-using-1d-min-max-mipmaps.md)
This method computes volumetric shadows for single-scattering participating media in real-time on the GPU by treating the shadow map as a 2D height field and reducing the integration problem to 1D via epipolar rectification. It differs from incremental integration methods by allowing parallel processing of all camera rays using only DirectX 9 pixel shader features, and from shadow volume methods by looping over pixels rather than geometry, thereby avoiding large vertex transfers and supporting high-resolution shadow maps.

#### Multiple Scattering (Max/Kajiya)
**Definition:** Extension of Blinn's model to arbitrary shapes via analytical functions, accounting for multiple reflections [S1, unofficial].
**Step:** Kajiya's two-pass method: Pass 1 pre-calculates reflected light proportion per voxel considering attenuation. Pass 2 renders image via ray casting using pre-calculated intensity info [S1, unofficial].
**History:** First scattering rendering model by Kajiya [KH84]. Extended by Max [Max94] to include anisotropic light dispersion (e.g., halo effect) [S1, unofficial].

#### Multiple Forward/Backward Scattering Approximation
**Step:** Kniss et al. [30] generalized single-pass shadow algorithms to multiple forward scattering by gathering accumulated light from several sampled directions in the previous sampling plane [S2, book]. Zhang et al. [62] added backward scattering from the next couple of sampling planes [S2, book].

#### Analytical Single Scattering without Boundaries
**Definition:** Proposes an analytical solution for single scattering by participating media without boundaries under a point light [S3].
**Number:** Can achieve a real-time frame rate [S3].

## Background
The concept of single scattering was first introduced by Kajiya [KH84] as the first model allowing for scattering rendering, using a two-pass approach [S1, unofficial]. Max [Max86] extended Blinn's layer-based model to arbitrary shapes using analytical functions, and later [Max94] added anisotropic light dispersion [S1, unofficial]. Blinn's original model was limited to layer-shaped media [S1, unofficial]. Modern implementations often use Monte Carlo methods or deterministic ray marching with precomputed shadowing for real-time performance [S1, unofficial].

## Sources
[S1] Représentation et algorithmes pour l'exploration interactive de volumes procéduraux étendus et détaillés, https://inria.hal.science/inria-00598406v1, fetched 2026-10-26 (unofficial)
[S2] Local and Global Illumination in the Volume Rendering Integral, https://avishek.net/assets/papers/nerfs/local-and-global-illumination-rendering-integral.pdf, fetched 2026-10-27 (book)
[S3] A survey on rendering homogeneous participating media, https://doi.org/10.1007/s41095-021-0249-1, fetched 2026-10-26
