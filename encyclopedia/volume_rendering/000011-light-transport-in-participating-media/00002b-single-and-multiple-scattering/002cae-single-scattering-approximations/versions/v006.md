# Single Scattering Approximations

Also called: Single-scattering, First-order scattering, Direct in-scattering

## Overview
Single scattering approximations solve the Radiative Transfer Equation (RTE) by assuming that light undergoes exactly one scattering event within the participating medium before reaching the camera [S2, book][S3]. This simplification ignores multiple scattering events and self-illumination (glow), making it applicable when the volume has low albedo or low density [S2, 5264-single-scattering-approximation.md, book][S3]. It is distinguished from multiple scattering methods by its computational efficiency and from homogeneous-only models by its ability to handle heterogeneous media in some variants [5566-single-scattering.md, 5264-single-scattering-approximation.md].

## Shared principles
All methods in this family assume the in-scattered radiance at a point $\mathbf{x}$ originates directly from light sources, attenuated by the transmittance from the source to $\mathbf{x}$, and travels directly to the camera, attenuated by the transmittance from $\mathbf{x}$ to the camera [S2, book][S3]. The general integral form for a ray $\mathbf{r}(t) = \mathbf{o} + t\boldsymbol{\omega}$ is:
$$ L_o(\mathbf{o}, \boldsymbol{\omega}) = T(\mathbf{o}, \mathbf{p}_{far}) L_{bg} + \int_{\mathbf{o}}^{\mathbf{p}_{far}} T(\mathbf{o}, \mathbf{x}) \sigma_s(\mathbf{x}) L_{in}(\mathbf{x}, \boldsymbol{\omega}_i) d\mathbf{x} $$
where $T(\mathbf{o}, \mathbf{x}) = \exp\left(-\int_{\mathbf{o}}^{\mathbf{x}} \sigma_t(\mathbf{u}) d\mathbf{u}\right)$ is the eye transmittance, $\sigma_s$ is the scattering coefficient, and $L_{in}$ is the incident radiance from light sources [5566-single-scattering.md]. The term $L_{in}$ typically includes the source intensity, geometric attenuation (e.g., inverse square law), and the phase function $p(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o)$ [5566-single-scattering.md].

## Subtopics

### [Single-scattering homogeneous volume rendering](1135-single-scattering-homogeneous-volume-rendering.md)
This method renders a participating medium by solving the RTE under the assumption that light scatters exactly once within a homogeneous volume before reaching the camera. It extends a basic absorption-only model by adding in-scattering from surface-emitting geometry, assuming the surfaces do not reflect or transmit light. It is distinguished from [Single-scattering atmosphere rendering](3289-single-scattering-atmosphere-rendering.md) by its general applicability to arbitrary scene manifolds rather than specific atmospheric layers, and from [Volume Single Scattering (Monte Carlo with Point Light)](2293-volume-single-scattering-monte-carlo-with-point-light.md) by its integration over the entire scene manifold $\mathcal{M}$ rather than discrete point lights.

### [Volume Single Scattering (Monte Carlo with Point Light)](2293-volume-single-scattering-monte-carlo-with-point-light.md)
This method computes the radiance contribution of a single scattering event in a participating medium by integrating the incident light from a point source along a view ray using Monte Carlo integration. It solves the problem of rendering volumetric effects (such as fog, smoke, or clouds) where light from a specific source is scattered into the camera, attenuated by the medium's extinction. Unlike deterministic uniform ray marching variants, this approach uses stochastic sampling of the ray distance to estimate the integral, which allows for efficient handling of complex lighting scenarios by decoupling the spatial integration from the light evaluation. It builds on [Single Scattering Approximations](5340-single-scattering-approximations.md) by specifically implementing the Monte Carlo estimator for a point light source.

### [Single-Scattering Atmosphere Rendering](3289-single-scattering-atmosphere-rendering.md)
Renders the sky and atmospheric scattering by numerically integrating single scattering along a ray through a spherical atmosphere shell, accumulating Rayleigh and Mie contributions weighted by view and light transmittance. It solves the Radiative Transfer Equation (RTE) for a single distant light source (the sun), accounting for Rayleigh scattering (gas molecules), Mie scattering (aerosols), and Ozone absorption. Unlike [Single-scattering homogeneous volume rendering](1135-single-scattering-homogeneous-volume-rendering.md) which assumes constant density, this method models the exponential density decay of a planetary atmosphere and the resulting wavelength-dependent color shifts (e.g., red sunsets).

### [Microflake Model](4739-microflake-model.md)
The Microflake Model is a physically-based participating medium model that represents a medium as a homogeneous cloud of microscopic flakes, where orientations are statistically described by a Normal Distribution Function (NDF) $D(\boldsymbol{\omega})$. It solves the problem of rendering anisotropic media (such as fur, hair, or fibrous materials) by deriving direction-dependent attenuation coefficients and phase functions that satisfy Helmholtz's reciprocity principle by construction. It differs from isotropic single-scattering approximations [Single Scattering](5566-single-scattering.md) by accounting for the geometric alignment of scattering elements, and it differs from the Microscopic Self-Shadowing variant [Microscopic Self-Shadowing Microflake Model](4746-microscopic-self-shadowing-microflake-model.md) by assuming uncorrelated microflake positions (no microscopic self-shadowing).

### [Heterogeneous In-scattering with Multiple Scattering](4934-heterogeneous-in-scattering-with-multiple-scattering.md)
This method computes the radiance contribution from light scattering within a heterogeneous participating medium by combining direct in-scattering with recursively computed indirect lighting (multiple scattering) up to a fixed maximum depth. It distinguishes itself from its siblings, such as [Single Scattering Approximations](5340-single-scattering-approximations.md), by explicitly modeling the bounce of light between volume elements and surfaces, thereby capturing effects like volumetric shadows and color bleeding that single-scattering models miss.

### [Single Scattering Approximation](5264-single-scattering-approximation.md)
The Single Scattering Approximation simplifies the volume rendering integral by assuming that light scatters only once before reaching the viewer, ignoring multiple scattering events and self-illumination (glow). This method is applicable when the volume has low albedo or low density, making the probability of secondary scattering negligible. It differs from multiple scattering methods by precomputing shadowing effects in a separate pass or combined pass, and from homogeneous approximations by handling spatially varying extinction coefficients.

### [Single Scattering](5566-single-scattering.md)
Single scattering approximates the in-scattered radiance at a point $\mathbf{x}$ by assuming it originates directly from light sources, attenuated by the transmittance from the source to $\mathbf{x}$, and travels directly to the camera, attenuated by the transmittance from $\mathbf{x}$ to the camera. This method solves the problem of computing volumetric illumination without accounting for light that scatters more than once, effectively treating the volume as a collection of direct-lighting surfaces. It is distinguished from [Single-scattering homogeneous volume rendering](1135-single-scattering-homogeneous-volume-rendering.md) by its general applicability to heterogeneous media and from [Volume Single Scattering (Monte Carlo with Point Light)](2293-volume-single-scattering-monte-carlo-with-point-light.md) by its formulation as a deterministic integral over the spherical domain rather than a stochastic estimator for specific light types.

### [Real-Time Volumetric Shadows using 1D Min-Max Mipmaps](9426-real-time-volumetric-shadows-using-1d-min-max-mipmaps.md)
This method computes real-time volumetric shadows in single-scattering participating media on the GPU by evaluating the scattering integral over the intersections of camera rays with a shadow map treated as a 2D height field. It distinguishes itself from sibling methods by using epipolar rectification to reduce the problem to 1D height fields and employing 1D min-max mipmaps to find lit segments in parallel, allowing all camera rays to be processed simultaneously using only standard pixel shader features without requiring GPGPU APIs or camera rectification.

#### Multiple Scattering (Max/Kajiya)
**Definition:** Extension of Blinn's model to arbitrary shapes via analytical functions, accounting for multiple reflections [S1, unofficial].
**Step:** Kajiya's two-pass method: Pass 1 pre-calculates reflected light proportion per voxel considering attenuation. Pass 2 renders image via ray casting using pre-calculated intensity info [S1, unofficial].
**History:** First scattering rendering model by Kajiya [KH84]. Extended by Max [Max94] to include anisotropic light dispersion (e.g., halo effect) [S1, unofficial].

#### Multiple Forward/Backward Scattering Approximation
**Step:** Kniss et al. [30] generalized single-pass shadow algorithms to multiple forward scattering by gathering accumulated light from several sampled directions in the previous sampling plane [S2, book]. Zhang et al. [62] added backward scattering from the next couple of sampling planes [S2, book].

#### Analytical Single Scattering without Boundaries
**Definition:** Proposes an analytical solution for single scattering by participating media without boundaries under a point light [S3, fetched 2026-10-26].
**Number:** Can achieve a real-time frame rate [S3, fetched 2026-10-26].

## Background
The concept of single scattering was first introduced by Kajiya [KH84] as the first model allowing for scattering rendering, using a two-pass method [S1, unofficial]. Max [Max86] extended Blinn's layer-based model to arbitrary shapes using analytical functions, and later [Max94] added anisotropic light dispersion [S1, unofficial]. Blinn's model itself is limited to layer-shaped media [S1, unofficial]. Modern implementations often use ray marching or Monte Carlo integration to solve the single-scattering integral for heterogeneous media [5566-single-scattering.md, 2293-volume-single-scattering-monte-carlo-with-point-light.md].

## Sources
[S1] Représentation et algorithmes pour l'exploration interactive de volumes procéduraux étendus et détaillés, https://inria.hal.science/inria-00598406v1, fetched 2026-10-26 (unofficial)
[S2] Local and Global Illumination in the Volume Rendering Integral, https://avishek.net/assets/papers/nerfs/local-and-global-illumination-rendering-integral.pdf, fetched 2026-10-27 (book)
[S3] A survey on rendering homogeneous participating media, https://doi.org/10.1007/s41095-021-0249-1, fetched 2026-10-26
