# Multiple Scattering Models

Also called: Multiple Scattering, Multi-Bounce Scattering, Indirect Volume Lighting

## Overview
Multiple scattering models simulate light transport in participating media where photons undergo more than one scattering event before escaping the volume [S1][S2, unofficial]. While single scattering approximations suffice for thin media like light haze, multiple scattering is required to render dense media such as clouds or smoke, which can reflect up to 90% of incident light through dozens or hundreds of bounces [S2, unofficial]. It is distinguished from single scattering approximations by its ability to capture the bright, diffuse appearance of thick volumes, though it incurs significantly higher computational costs and noise [S1][S2, unofficial].

## Shared principles
The fundamental problem addressed by multiple scattering models is the underestimation of radiance in optically thick media. In single scattering, the in-scattering term $L_{in}$ in the radiative transfer equation is approximated by direct illumination from light sources. In multiple scattering, $L_{in}$ includes contributions from light that has already scattered one or more times within the medium.

The radiative transfer equation for a participating medium is:
$$ \frac{dL(\mathbf{x}, \omega)}{ds} = -\sigma_t(\mathbf{x}) L(\mathbf{x}, \omega) + \sigma_s(\mathbf{x}) \int_{4\pi} p(\omega', \omega) L(\mathbf{x}, \omega') d\omega' + L_e(\mathbf{x}, \omega) $$
where $\sigma_t = \sigma_a + \sigma_s$ is the extinction coefficient, $\sigma_s$ is the scattering coefficient, $p$ is the phase function, and $L_e$ is emitted radiance.

Multiple scattering methods generally fall into two categories:
1.  **Stochastic/Recursive Methods:** These trace random walks or recursively evaluate the in-scattering integral. They are unbiased but suffer from high variance (noise) and high computational cost, as every bounce multiplies the work [S1][S2, unofficial].
2.  **Approximation/Deterministic Methods:** These approximate the integral term using precomputed data structures (e.g., light volumes, 3D filters) or analytical assumptions (e.g., Most Probable Paths, diffusion). They are faster but may introduce artifacts or inaccuracies in heterogeneous media [S1][S2, unofficial].

Key shared parameters include:
*   **Scattering Coefficient ($\sigma_s$):** Determines the probability of scattering per unit distance.
*   **Absorption Coefficient ($\sigma_a$):** Determines the probability of absorption per unit distance.
*   **Phase Function ($p$):** Describes the angular distribution of scattered light.
*   **Maximum Depth/Bounces:** A cutoff for the number of scattering events to simulate, balancing accuracy and performance [S1][S2, unofficial].

## Subtopics

### [Heterogeneous In-scattering with Multiple Scattering](4934-heterogeneous-in-scattering-with-multiple-scattering.md)
This method computes the radiance contribution from light scattering within a heterogeneous volume, handling direct lighting from light sources and recursively computing indirect lighting (multiple scattering) up to a maximum depth. It distinguishes itself from homogeneous approximations by evaluating local extinction and scattering coefficients at each integration step, allowing for spatially varying density and material properties.

### [Most Probable Paths (MPP) Lighting Model](12244-most-probable-paths-mpp-lighting-model.md)
The Most Probable Paths (MPP) model is a lighting approximation for participating media that computes the multiple-scattering radiance $L_{MS}$ by summing contributions only from paths close to the "Most Probable Path" (MPP) for a given number of scattering events. It splits total radiance into object, single-scattered, and multiple-scattered components, using Gaussian-weighted incident radiance to approximate the scattering integral. Unlike diffusion-based siblings, it handles general inhomogeneous media and avoids artifacts near boundaries or low-density areas where the mean free path is comparable to feature sizes.

### [3D-Filter / Concrete Directions](12253-3d-filter-concrete-directions.md)
The 3D-Filter method approximates multiple scattering in participating media by representing the directional distribution of radiance using extended form factors stored in a 3D grid. It solves the problem of computationally expensive indirect light transport in heterogeneous media (such as clouds or snow) by precomputing contribution ratios from neighboring voxels to a center voxel, thereby avoiding the calculation of form factors between all voxel pairs. It is distinguished from sibling methods like Heterogeneous In-scattering with Multiple Scattering by its specific use of a "contribution-ratio pattern" (the 3D-filter) derived from a smaller space with mean density, and from Most Probable Paths by its deterministic grid-based accumulation rather than stochastic path tracing.

### [Multiple Scattering](12474-multiple-scattering.md)
Multiple scattering simulates light transport in participating media where photons undergo more than one scattering event before escaping the volume. While single scattering suffices for thin media like light haze, multiple scattering is required to render dense media such as clouds or smoke, which can reflect up to 90% of incident light through dozens or hundreds of bounces. It is distinguished from single scattering approximations by its ability to capture the bright, diffuse appearance of thick volumes, though it incurs significantly higher computational costs and noise.

### Multiple Scattering (Max/Kajiya)
This is the first scattering rendering model, presented by Kajiya [KH84], and extended by Max [Max94] to include anisotropic light dispersion (e.g., halo effect) [S2, unofficial]. Kajiya's method is a two-pass approach: Pass 1 pre-calculates the proportion of reflected light per voxel considering attenuation; Pass 2 renders the image via ray casting using the pre-calculated intensity information [S2, unofficial]. Max extended Blinn's model to arbitrary shapes via analytical functions, accounting for multiple reflections [S2, unofficial].

### Multiple Forward/Backward Scattering Approximation
Kniss et al. [30] generalized single-pass shadow algorithms to multiple forward scattering by gathering accumulated light from several sampled directions in the previous sampling plane [S3, book]. Zhang et al. [62] added backward scattering from the next couple of sampling planes [S3, book].

### Riley et al. Angular Distribution Renderer
Riley et al. [76] proposed a multiple model system able to generate fine angularly dependent chromatic effects such as rainbows, halos, and glories [S4]. The core is an angular distribution renderer developed to handle the forward multi-scattering behavior of large water and ice atmospheric particles [S4]. It uses multiple scattering angular distribution functions to approximate the distribution of light after a certain number of scattering events [S4]. Designed for efficient single-pass rendering on programmable graphics hardware, it applies the half-angle slicing scheme for a light buffer and an eye buffer [S4]. The volume data is sliced at an angle halfway between the eye and light perspectives [S4]. Skylight and aerial perspective effects, based on a simplified single-scattering analytic model, are added to increase realism [S4].

### Ambient Term Approximation for Multiple Scattering
This approach approximates multiple scattering by an ambient term to obtain images of typical shafts of light caused by sunlight [S4].

## Background
The concept of multiple scattering in volume rendering originated with Kajiya's 1984 work, which was the first model to account for scattering in participating media [S2, unofficial]. Max extended this in 1994 to handle arbitrary shapes and anisotropic dispersion [S2, unofficial]. Early methods often relied on approximations, such as treating multiple scattering as an ambient term [S4].

Modern implementations face the challenge that simulating multiple scattering means following rays through many additional bounces, multiplying computational work [S1]. A thick cumulus cloud reflects up to 90% of the light that hits it, requiring dozens or hundreds of bounces for accuracy [S1][S2, unofficial]. This is the primary reason volume rendering has a reputation for being slow [S1].

GPU rendering and clever approximations (covered in related articles) are used to soften this cost [S1]. Denoising is often employed, although volumes are notoriously harder to denoise than surfaces because helper data such as surface normals does not exist inside the volume [S1].

## Sources
[S1] Introduction to Volumetric Rendering, https://help.maxon.net/c4d/2027/en-us/Content/html/What_is_Volumetric_Rendering.html, fetched 2026-10-29
[S2] Représentation et algorithmes pour l'exploration interactive de volumes procéduraux étendus et détaillés, https://inria.hal.science/inria-00598406v1, fetched 2026-10-26 (unofficial)
[S3] Local and Global Illumination in the Volume Rendering Integral, https://avishek.net/assets/papers/nerfs/local-and-global-illumination-rendering-integral.pdf, fetched 2026-10-27 (book)
[S4] A survey on participating media rendering techniques, https://doi.org/10.1007/s00371-005-0287-1, fetched 2026-10-26
