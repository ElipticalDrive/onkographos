# Multiple Scattering Models

Also called: Multiple Scattering, Global Illumination in Participating Media

## Overview
Multiple scattering models simulate the effect resulting from more than one scattering bounce within a participating medium [S1]. They solve the problem of rendering dense media, such as thick clouds or smoke, where single scattering approximations fail to capture the brightness and volume of light trapped inside the medium [S2, unofficial]. Unlike single scattering, which only accounts for direct light from sources, multiple scattering accounts for light that has been scattered by the medium itself, requiring the simulation of photons bouncing dozens or hundreds of times before escaping [S2, unofficial].

## Shared principles
The radiative transfer equation (RTE) describes light transport in participating media. The total radiance $L(x, \omega)$ at position $x$ in direction $\omega$ is governed by:

$$ \frac{dL(x, \omega)}{ds} = -\sigma_t(x) L(x, \omega) + \sigma_s(x) \int_{S^2} p(\omega', \omega) L(x, \omega') d\omega' + \sigma_a(x) L_e(x, \omega) $$

where $\sigma_t = \sigma_a + \sigma_s$ is the extinction coefficient, $\sigma_s$ is the scattering coefficient, $\sigma_a$ is the absorption coefficient, $p(\omega', \omega)$ is the phase function, and $L_e$ is the emissive radiance [S3, book].

Multiple scattering arises from the integral term $\int_{S^2} p(\omega', \omega) L(x, \omega') d\omega'$, where the incoming radiance $L(x, \omega')$ itself contains contributions from other directions, creating a recursive dependency. Solving this exactly is computationally prohibitive because every scattering event multiplies the work [S1].

Key shared concepts include:
1.  **Optical Depth:** The probability of a photon surviving without scattering over a distance $t$ is given by transmittance $T(t) = \exp(-\int_0^t \sigma_t(x(s)) ds)$.
2.  **Single Scattering Albedo:** The probability that a scattering event is a scatter rather than an absorption is $\omega_0 = \frac{\sigma_s}{\sigma_t}$ [S1].
3.  **Phase Function:** Describes the angular distribution of scattered light. Common approximations include isotropic ($p = \frac{1}{4\pi}$) and strongly forward-scattering functions (e.g., Henyey-Greenstein) [S1][S2, unofficial].

## Subtopics

### [Heterogeneous In-scattering with Multiple Scattering](4934-heterogeneous-in-scattering-with-multiple-scattering.md)
This method computes the radiance contribution from light scattering within a heterogeneous volume, handling direct lighting from sources and recursively computing indirect lighting (multiple scattering) up to a maximum depth. It serves as a foundational volumetric path-tracing approach for simulating realistic light transport in participating media like clouds and fog. It is distinguished from sibling methods like [Most Probable Paths (MPP) Lighting Model](12244-most-probable-paths-mpp-lighting-model.md) by its explicit recursive ray-casting for indirect bounces rather than precomputed filters, and from [3D-Filter / Concrete Directions](12253-3d-filter-concrete-directions.md) by its stochastic sampling of scattering directions. Unlike [Multiple Scattering](12474-multiple-scattering.md) which may use brute-force global illumination, this specific implementation uses a fixed-depth recursion with a hardcoded isotropic phase function.

### [Most Probable Paths (MPP) Lighting Model](12244-most-probable-paths-mpp-lighting-model.md)
The Most Probable Paths (MPP) lighting model approximates multiple scattering in general inhomogeneous participating media by summing contributions only from paths close to the "most probable path" (MPP) for a given number of scattering events, assuming strong forward scattering. It splits radiance into object, single-scattered, and multiple-scattered components, using MPPs to approximate the scattering integral without requiring the diffusion approximation or planar surface assumptions. Unlike [Kniss et al. 2002], which uses a constant effective scattering angle, this model determines the effective scattering angle locally based on the phase function and the amount of scattering that has occurred.

### [Riley et al. Angular Distribution Renderer](12249-riley-et-al-angular-distribution-renderer.md)
The Riley et al. Angular Distribution Renderer is a single-pass volume rendering method designed to approximate multiple scattering in participating media, specifically targeting large atmospheric particles like water and ice. It solves the computational complexity of the radiative transfer equation by precomputing angular distribution functions that describe how light spreads after a fixed number of scattering events, allowing the renderer to direct light into the eye ray in a single pass. Unlike recursive ray casting or low-albedo approximations, this method captures essential single-scattering effects and chromatic phenomena (rainbows, halos) while avoiding the premature darkening artifacts common in simpler models.

### [3D-Filter / Concrete Directions](12253-3d-filter-concrete-directions.md)
The 3D-Filter method approximates multiple scattering in participating media by precomputing a grid of "extended form factors" that represent the directional distribution of radiance. It solves the problem of computationally expensive recursive ray casting for indirect illumination by storing contribution ratios from neighboring voxels to a center voxel, allowing the radiance to be computed via simple multiplication and accumulation. It sets itself apart from sibling methods like Most Probable Paths by using a fixed voxel grid aligned with the viewing direction rather than stochastic path tracing, and from Angular Distribution Renderers by explicitly separating the computation of second and third-order scattering contributions into a precomputed filter pattern.

### [Multiple Scattering](12474-multiple-scattering.md)
Multiple scattering simulates the effect resulting from more than one scattering bounce within a participating medium. It solves the problem of rendering dense media, such as thick clouds or smoke, where single scattering approximations fail to capture the brightness and volume of light trapped inside the medium. Unlike single scattering, which only accounts for direct light from sources, multiple scattering accounts for light that has been scattered by the medium itself, requiring the simulation of photons bouncing dozens or hundreds of times before escaping.

### [Hybrid Hair/Fur Rendering Strategy](15774-hybrid-hair-fur-rendering-strategy.md)
This method approximates multiple scattering in dense hair and fur by combining explicit geometric ray tracing for direct illumination with a volumetric participating medium model for higher-order scattering. It solves the computational bottleneck of simulating dozens of light bounces among fibers by treating the aggregate fiber distribution as an anisotropic medium, preserving high-frequency strand details while efficiently capturing soft, low-frequency volumetric glow. Unlike microflake models that assume opaque particles or dual scattering methods that rely on local uniformity assumptions, this approach normalizes the hair BCSDF to define the phase function and uses spectral reweighting to maintain accurate color shifts.

### Multiple Scattering (Max/Kajiya)
Extension of Blinn's model to arbitrary shapes via analytical functions, accounting for multiple reflections [S2, unofficial]. Kajiya's two-pass method: Pass 1 pre-calculates reflected light proportion per voxel considering attenuation. Pass 2 renders image via ray casting using pre-calculated intensity info [S2, unofficial]. First scattering rendering model by Kajiya [KH84]. Extended by Max [Max94] to include anisotropic light dispersion (e.g., halo effect) [S2, unofficial].

### Multiple Forward/Backward Scattering Approximation
Kniss et al. [30] generalized single-pass shadow algorithms to multiple forward scattering by gathering accumulated light from several sampled directions in the previous sampling plane [S3, book]. Zhang et al. [62] added backward scattering from the next couple of sampling planes [S3, book].

### Ambient Term Approximation for Multiple Scattering
Approximate multiple scattering by an ambient term to obtain images of typical shafts of light caused by sunlight [S4].

## Background
The first scattering rendering model allowing multiple scattering was presented by Kajiya [KH84], a two-pass method where the first pass pre-calculates reflected light proportion per voxel considering attenuation, and the second pass renders the image via ray casting using pre-calculated intensity information [S2, unofficial]. Max extended this model [Max94] to include anisotropic light dispersion, such as the halo effect, allowing for arbitrary shapes via analytical functions [S2, unofficial].

Modern engines soften the cost of multiple scattering further with denoising, although volumes are notoriously harder to denoise than surfaces because the helper data that guides a denoiser, such as surface normals, simply does not exist inside a cloud [S1].

## Sources
[S1] Introduction to Volumetric Rendering, https://help.maxon.net/c4d/2027/en-us/Content/html/What_is_Volumetric_Rendering.html, fetched 2026-10-29
[S2] Représentation et algorithmes pour l'exploration interactive de volumes procéduraux étendus et détaillés, https://inria.hal.science/inria-00598406v1, fetched 2026-10-26 (unofficial)
[S3] Local and Global Illumination in the Volume Rendering Integral, https://avishek.net/assets/papers/nerfs/local-and-global-illumination-rendering-integral.pdf, fetched 2026-10-27 (book)
[S4] A survey on participating media rendering techniques, https://doi.org/10.1007/s00371-005-0287-1, fetched 2026-10-26
