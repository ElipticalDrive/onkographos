# Single Scattering Approximations

Also called: Single-scattering, Single-bounce approximation, First-order scattering

## Overview
Single scattering approximations solve the Radiative Transfer Equation (RTE) by assuming that light undergoes at most one scattering event within the participating medium before reaching the camera [S1, unofficial]. This simplification neglects multiple scattering (glow) and emission from the medium itself, reducing the problem to computing the in-scattered radiance from light sources attenuated by the medium's extinction [S1, unofficial]. It is distinguished from multiple scattering methods by its computational efficiency and lack of noise in direct illumination, but fails to capture volumetric glow and color bleeding in high-albedo or dense media [S1, unofficial].

## Shared principles
All methods in this family rely on the single-scattering term of the volume rendering integral. The outgoing radiance $L_o$ along a ray is composed of the attenuated background radiance and the in-scattered radiance from light sources [S1, unofficial]:

$$ L_o(\mathbf{o}, \boldsymbol{\omega}) = T(\mathbf{o}, \mathbf{p}_{far}) L_{bg} + \int_{\mathbf{o}}^{\mathbf{p}_{far}} T(\mathbf{o}, \mathbf{x}) \sigma_s(\mathbf{x}) L_{in}(\mathbf{x}, \boldsymbol{\omega}) d\mathbf{x} $$

where:
*   $T(\mathbf{o}, \mathbf{x}) = \exp\left(-\int_{\mathbf{o}}^{\mathbf{x}} \sigma_t(\mathbf{u}) d\mathbf{u}\right)$ is the transmittance from the eye to point $\mathbf{x}$ [S3].
*   $\sigma_s(\mathbf{x})$ is the scattering coefficient [G].
*   $L_{in}(\mathbf{x}, \boldsymbol{\omega})$ is the incident radiance at $\mathbf{x}$ from light sources, attenuated by the path from the light to $\mathbf{x}$ [S1, unofficial].

The incident radiance for a point light at $\mathbf{l}$ with intensity $I_l$ is:
$$ L_{in}(\mathbf{x}, \boldsymbol{\omega}) = \frac{I_l}{\|\mathbf{x} - \mathbf{l}\|^2} p(\boldsymbol{\omega}_l, \boldsymbol{\omega}) T(\mathbf{l}, \mathbf{x}) V(\mathbf{l}, \mathbf{x}) $$
where $p$ is the phase function, $T(\mathbf{l}, \mathbf{x})$ is the light-to-point transmittance, and $V$ is visibility [S1, unofficial].

## Subtopics

### [Single-scattering homogeneous volume rendering](1135-single-scattering-homogeneous-volume-rendering.md)
This method renders a participating medium by solving the RTE under the assumption that light scatters exactly once within a homogeneous volume before reaching the camera. It extends a basic absorption-only model by adding in-scattering from surface-emitting geometry, assuming the surfaces do not reflect or transmit light. It is distinguished from [Single-scattering atmosphere rendering](3289-single-scattering-atmosphere-rendering.md) by its general applicability to arbitrary scene manifolds rather than specific atmospheric layers, and from [Volume Single Scattering (Monte Carlo with Point Light)](2293-volume-single-scattering-monte-carlo-with-point-light.md) by its integration over the entire scene manifold $\mathcal{M}$ rather than discrete point lights.

### [Volume Single Scattering (Monte Carlo with Point Light)](2293-volume-single-scattering-monte-carlo-with-point-light.md)
Computes the radiance contribution of a single scattering event in a homogeneous participating medium by integrating in-scattered light from a point light source along a view ray. It differs from uniform ray marching variants by using Monte Carlo distance sampling to select scattering positions. It is a specific instance of single scattering approximations [5340-single-scattering-approximations.md] tailored for point lights and homogeneous media.

### [Single-Scattering Atmosphere Rendering](3289-single-scattering-atmosphere-rendering.md)
Renders the sky and atmospheric scattering by numerically integrating single scattering along a ray through a spherical atmosphere shell, accumulating Rayleigh and Mie contributions weighted by view and light transmittance. It solves the RTE for a single distant light source (the sun), accounting for Rayleigh scattering (gas molecules), Mie scattering (aerosols), and Ozone absorption. Unlike [Single-scattering homogeneous volume rendering](1135-single-scattering-homogeneous-volume-rendering.md) which assumes constant density, this method models the exponential density decay of a planetary atmosphere and the resulting wavelength-dependent color shifts (e.g., red sunsets).

### [Microflake Model](4739-microflake-model.md)
The Microflake Model is a physically-based participating medium model that represents a medium as a homogeneous cloud of microscopic, specular flakes. It solves the problem of accurately rendering anisotropic media (such as fur, hair, or certain fabrics) by deriving direction-dependent attenuation and phase functions from a statistical distribution of microflake normals. It differs from isotropic single-scattering approximations by explicitly accounting for the orientation of scattering particles, ensuring that the model satisfies Helmholtz's reciprocity principle by construction, which arbitrary anisotropic phase functions often violate.

### [Heterogeneous In-scattering with Multiple Scattering](4934-heterogeneous-in-scattering-with-multiple-scattering.md)
This method computes the total radiance contribution of a participating medium by combining direct in-scattering from light sources with recursively computed indirect in-scattering (multiple scattering) up to a fixed depth. It addresses the limitation of single-scattering approximations in heterogeneous media where light is absorbed and scattered non-uniformly, and where surface reflections (albedo) significantly influence the volumetric appearance. Unlike homogeneous approximations, it integrates extinction and scattering coefficients along the ray path to handle density variations.

### [Single Scattering Approximation](5264-single-scattering-approximation.md)
The Single Scattering Approximation is a simplification of the volume rendering integral that assumes low albedo and/or low density, such that the probability of light scattering more than once is negligible. It solves the problem of computing volumetric lighting by considering only single scattering events, ignoring glow and multiple scattering contributions. It sets itself apart from multiple scattering methods by precomputing shadowing effects to determine incident radiance, and from diffusion approximations by retaining directional scattering properties rather than assuming isotropic diffuse light.

### [Single Scattering](5566-single-scattering.md)
Single scattering is an approximation where in-scattered radiance is assumed to originate directly from light sources, treating volumetric lighting similarly to direct lighting in surface rendering. It differs from multiple scattering methods by ignoring light that has bounced off other particles, and from homogeneous approximations by allowing for arbitrary spatial variation in density and phase functions.

### [Real-Time Volumetric Shadows using 1D Min-Max Mipmaps](9426-real-time-volumetric-shadows-using-1d-min-max-mipmaps.md)
This method computes volumetric shadows for single-scattering participating media in real-time on the GPU by evaluating the scattering integral over the intersections of camera rays with a shadow map treated as a 2D height field. It distinguishes itself from sibling methods by using epipolar rectification to reduce the problem to 1D height fields and employing 1D min-max mipmaps to find lit segments in parallel, allowing all camera rays to be processed simultaneously using only standard pixel shader features without requiring GPGPU APIs or camera rectification.

#### Multiple Scattering (Max/Kajiya)
- **Definition:** Extension of Blinn's model to arbitrary shapes via analytical functions, accounting for multiple reflections [S1, unofficial].
- **Step:** Kajiya's two-pass method: Pass 1 pre-calculates reflected light proportion per voxel considering attenuation. Pass 2 renders image via ray casting using pre-calculated intensity info [S1, unofficial].
- **History:** First scattering rendering model by Kajiya [KH84]. Extended by Max [Max94] to include anisotropic light dispersion (e.g., halo effect) [S1, unofficial].

#### Multiple Forward/Backward Scattering Approximation
- **Step:** Kniss et al. [30] generalized single-pass shadow algorithms to multiple forward scattering by gathering accumulated light from several sampled directions in the previous sampling plane [S2, book]. Zhang et al. [62] added backward scattering from the next couple of sampling planes [S2, book].

#### Analytical Single Scattering without Boundaries
- **Definition:** Proposes an analytical solution for single scattering by participating media without boundaries under a point light [S3].
- **Number:** Can achieve a real-time frame rate [S3].

## Background
The concept of single scattering was first introduced by Kajiya [KH84] as a two-pass method for rendering participating media, marking the first model to account for scattering [S1, unofficial]. Max [Max86] extended Blinn's layered medium model to arbitrary shapes using analytical functions, and later [Max94] added anisotropic light dispersion to account for effects like halos [S1, unofficial]. The approximation remains standard for real-time rendering and atmospheric effects due to its efficiency, though it is limited to low-albedo media [S1, unofficial].

## Sources
[S1] Représentation et algorithmes pour l'exploration interactive de volumes procéduraux étendus et détaillés, https://inria.hal.science/inria-00598406v1, fetched 2026-10-26 (unofficial)
[S2] Local and Global Illumination in the Volume Rendering Integral, https://avishek.net/assets/papers/nerfs/local-and-global-illumination-rendering-integral.pdf, fetched 2026-10-27 (book)
[S3] A survey on rendering homogeneous participating media, https://doi.org/10.1007/s41095-021-0249-1, fetched 2026-10-26
