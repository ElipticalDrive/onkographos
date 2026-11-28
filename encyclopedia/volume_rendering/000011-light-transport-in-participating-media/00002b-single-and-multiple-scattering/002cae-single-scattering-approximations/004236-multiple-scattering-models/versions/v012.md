# Multiple Scattering Models


## Overview
Multiple scattering models simulate light transport where photons undergo more than one scattering event within a participating medium [S1][S2, unofficial]. They solve the problem of rendering dense media, such as thick clouds or smoke, which appear dark and lifeless under single scattering approximations because they fail to capture the internal light bouncing that contributes significantly to their brightness and color [S2, unofficial]. Unlike single-scattering approximations, which only account for direct light from sources, multiple scattering models explicitly simulate indirect light transport within the volume, transforming dense volumes from opaque grey masses into luminous, believable objects [S1][S2, unofficial].

## Shared principles
The radiative transfer equation (RTE) describes light transport in participating media [G]. The total radiance $L(x, \omega)$ at position $x$ in direction $\omega$ is governed by:

$$ \frac{dL(x, \omega)}{ds} = -\sigma_t(x) L(x, \omega) + \sigma_s(x) \int_{S^2} p(\omega', \omega) L(x, \omega') d\omega' + \sigma_a(x) L_e(x, \omega) $$

where $\sigma_t = \sigma_a + \sigma_s$ is the extinction coefficient, $\sigma_s$ is the scattering coefficient, $\sigma_a$ is the absorption coefficient, $p(\omega', \omega)$ is the phase function, and $L_e$ is the emissive radiance [G].

Multiple scattering arises from the integral term $\int_{S^2} p(\omega', \omega) L(x, \omega') d\omega'$, where the incoming radiance $L(x, \omega')$ itself contains contributions from other directions, creating a recursive dependency [G]. Solving this exactly is computationally prohibitive because every scattering event multiplies the work [S1].

Key shared concepts include:
1. **Optical Depth:** The probability of a photon surviving without scattering over a distance $t$ is given by transmittance $T(t) = \exp(-\int_0^t \sigma_t(x(s)) ds)$ [G].
2. **Single Scattering Albedo:** The probability that a scattering event is a scatter rather than an absorption is $\omega_0 = \frac{\sigma_s}{\sigma_t}$ [G].
3. **Phase Function:** Describes the angular distribution of scattered light [G]. Common approximations include isotropic ($p = \frac{1}{4\pi}$) and strongly forward-scattering functions (e.g., Henyey-Greenstein) [G].

## Subtopics

### [Heterogeneous In-scattering with Multiple Scattering](4934-heterogeneous-in-scattering-with-multiple-scattering.md)
This method computes the radiance contribution from light scattering within a heterogeneous volume, handling both direct lighting from light sources and recursively computing indirect lighting (multiple scattering) up to a maximum depth. It is distinct from single-scattering approximations by explicitly simulating indirect light transport within the volume, transforming dense volumes from opaque grey masses into luminous, believable objects. Unlike the [Most Probable Paths (MPP) Lighting Model](12244-most-probable-paths-mpp-lighting-model.md) which focuses on specific path probabilities, or the [Riley et al. Angular Distribution Renderer](12249-riley-et-al-angular-distribution-renderer.md) which uses angular distributions, this approach uses recursive ray tracing with a fixed depth limit to approximate multiple bounces. It is computationally more expensive than heuristic shortcuts but provides physically plausible indirect illumination.

### [Most Probable Paths (MPP) Lighting Model](12244-most-probable-paths-mpp-lighting-model.md)
The Most Probable Paths (MPP) lighting model approximates multiple scattering in general inhomogeneous participating media by summing contributions only from paths close to the "Most Probable Path" (MPP). Unlike diffusion-based methods that assume uniform, optically thick media, this model handles inhomogeneous densities and avoids artifacts near boundaries by using a local effective scattering angle derived from the phase function. It splits radiance into object, single-scattered, and multiple-scattered components, allowing single and multiple scattering to be computed in a single volume sweep.

### [Riley et al. Angular Distribution Renderer](12249-riley-et-al-angular-distribution-renderer.md)
The Riley et al. Angular Distribution Renderer is a single-pass volume rendering method designed to efficiently simulate forward multiple scattering in atmospheric media, such as clouds and fog, while capturing fine angularly dependent chromatic effects like rainbows and halos. It distinguishes itself from sibling methods by using pre-computed multiple scattering angular distribution functions to approximate the light distribution after a specific number of scattering events, allowing for efficient hardware-accelerated rendering without the premature darkening artifacts common in low-albedo models.

### [3D-Filter / Concrete Directions](12253-3d-filter-concrete-directions.md)
The 3D-Filter method approximates multiple scattering in participating media by representing the directional distribution of radiance using extended form factors within a voxel grid. It solves the problem of computationally prohibitive multiple scattering integration by precomputing a "contribution-ratio pattern" (the 3D-filter) that encodes how energy from neighboring voxels contributes to the radiance of a center voxel in the viewing direction. Unlike sibling methods that may use diffusion equations or stochastic path tracing for the full angular distribution, this approach restricts the solution to specific scattering orders (typically up to third order) and exploits the forward-scattering nature of media like clouds to accelerate the calculation of these form factors.

### [Multiple Scattering](12474-multiple-scattering.md)
Multiple scattering simulates light transport where photons undergo more than one scattering event within a participating medium. It solves the problem of rendering dense media, such as thick clouds or smoke, which appear dark and lifeless under single scattering approximations because they fail to capture the internal light bouncing that contributes significantly to their brightness and color. It is distinguished from single scattering by its ability to model the recursive dependency of radiance on itself, though it incurs a multiplicative computational cost for each additional bounce.

### [Hybrid Hair/Fur Rendering Strategy](15774-hybrid-hair-fur-rendering-strategy.md)
This method approximates multiple scattering in dense hair and fur by combining explicit geometric rendering for direct illumination with a volumetric participating medium model for higher-order scattering. It solves the computational bottleneck of simulating dozens of light bounces between fibers by treating the aggregate hair volume as an anisotropic medium, preserving high-frequency strand details while efficiently capturing soft volumetric glow. Unlike microflake models that assume opaque particles, this approach normalizes the hair BCSDF to define the phase function, preserving complex transmission paths.

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
