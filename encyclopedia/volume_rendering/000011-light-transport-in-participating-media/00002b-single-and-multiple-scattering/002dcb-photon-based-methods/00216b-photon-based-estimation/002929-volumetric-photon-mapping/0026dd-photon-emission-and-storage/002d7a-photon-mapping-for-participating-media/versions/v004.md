# Photon Mapping for Participating Media

Also called: Jensen and Christensen, Nonlinear RTE photon mapping, Photon Maps for Participating Media, Volume Photon Map, nl-RTE PM

## Overview
Photon Mapping for Participating Media extends photon mapping to handle light transport within volumetric media, introducing a distinct volume photon map to estimate radiance via density estimation [S1]. It uses Monte Carlo simulation of photon transport from light sources to account for spatially varying optical properties and general phase functions, making it suitable for non-homogeneous materials [S3, book]. It is distinguished by its separation of the volume photon map (used for indirect illumination) from the surface photon map and its reliance on a final gather pass to reconstruct inscattered light [S1].

## How it works
### Inputs and Outputs
*   **Inputs:** Light source definitions (position, flux, spectrum), scene geometry, participating media properties (extinction coefficient $\sigma_t$, scattering coefficient $\sigma_s$, albedo, phase function $p$), and camera parameters [S1][S3, book].
*   **Outputs:** A rendered image with indirect illumination and inscattered light from participating media [S1][S3, book].

### Data Structures
1.  **Volume Photon Map:** A spatial data structure storing photons that have interacted with the media [S1].
    *   **Fields per photon:**
        *   Position $\mathbf{x}$: 3D coordinates [G].
        *   Direction of incidence $\boldsymbol{\omega}_i$: 3D vector (required for anisotropic scattering to recover source radiance toward the eye) [S1].
        *   Flux $W$: Scalar energy value [G].
        *   Wavelength/Color: RGB or spectral data [G].
    *   **Storage Criteria:** Only photons representing indirect illumination are stored [S1]. This includes photons reflected or transmitted by surfaces before entering the media, and photons that have been scattered at least once within the media [S1]. Photons are stored after the first bounce in refractive media (see Subtopics) [S2].

### Algorithm Steps
1.  **Photon Tracing (Preprocess):** Emit photons from each light source [S4, unofficial]. Trace rays through the scene. When a photon enters a participating medium, stochastically choose the distance to the next interaction according to the medium's extinction coefficient [S4, unofficial]. At each interaction point, determine the probability of scattering versus absorption using the albedo [S4, unofficial]. If scattering occurs, compute the new direction by importance-sampling the phase function (e.g., Schlick phase function) [S1][S4, unofficial]. Store the photon at the interaction point if it represents indirect illumination [S1][S4, unofficial]. Use Russian roulette to terminate photons instead of generating multiple descendent photons with scaled-down powers, maintaining approximately equal photon power [S4, unofficial]. Continue tracing until absorption, escape, or a maximum bounce count [S4, unofficial].
2.  **Final Gather (Rendering):** Cast view rays from the camera [S3, book]. For each differential segment of the ray within the medium, gather an irradiance estimate from the stored photons using a spherical region around the segment [S4, unofficial]. Collect scattering events relevant to the viewing ray to account for inscattered photons [S3, book]. For anisotropic scattering, use the stored direction of incidence at hit points to correctly evaluate the phase function and recover source radiance toward the eye [S1].

### Parameters
*   **Number of Photons ($N$):** High number required for accurate convergence; the method is slow due to this requirement [S3, book].
*   **Number of Nearest Neighbors ($k$) / Spherical Radius:** Used in density estimation. Not given in the sources.
*   **Phase Function:** e.g., Schlick phase function [S1].
*   **Maximum Bounce Depth:** Not given in the sources.

### Edge Cases and Pitfalls
*   **Overestimation:** The method overestimates radiance inside participating media because energy conservation is not enforced within the volume, only at the surface via a nonsymmetric BTDF [S2].
*   **Performance:** The method can be slow because many photons are required for accurate convergence [S3, book].
*   **Refractive Media:** In continuously refractive media, straight shadow rays cannot be used for single scattering. Photons must be stored after the first bounce, increasing the required photon count [S2].
*   **Nonlinear Dependency:** The squared refractive index $n^2$ creates a strong nonlinear dependency that cannot be compensated by simple rescaling of light source fluxes [S2].

## Subtopics

### Photon Mapping for Refractive Media
A variant of photon mapping for participating media that handles single-scattered light in continuously refractive media where straight shadow rays are insufficient [S2]. It stores photons after the first bounce (unlike nonrefractive media which store after at least two bounces) to solve single scattering via radiance estimation from the photon map [S2]. The key distinction is the handling of basic radiance $\tilde{L}_i$ and scattered radiance $L_i$ via the squared refractive index $n^2$:
$$ \tilde{L}_i(\mathbf{x}, \boldsymbol{\omega}, v, t) = \frac{1}{n^2} L_i(\mathbf{x}, \boldsymbol{\omega}_i, v, t) $$
This relationship is derived from the distribution function and energy conservation principles [S2]. The drawback is that the necessary amount of photons to obtain high-quality results increases significantly due to this approach [S2]. The strong nonlinear dependency of $n^2$ cannot be compensated by simple rescaling of light source fluxes [S2].

## Operating characteristics
*   **Time Complexity:** Slow. Requires a large number of photons for accurate convergence, making the preprocessing step computationally expensive [S3, book].
*   **Memory Complexity:** High. Stores a large volume photon map containing positions, directions, and fluxes for all scattered photons [S1][S3, book].
*   **Strengths:** Handles non-homogeneous materials, spatially varying optical properties, and general phase functions effectively [S3, book]. Straightforward handling of single-scattered light in refractive media [S2].
*   **Limitations:** Overestimates radiance inside the medium due to lack of volume energy conservation [S2]. Performance degrades with complex refractive media due to increased photon requirements [S2].
*   **When to Use:** Scenes with complex participating media, non-homogeneous materials, or refractive media where single scattering is significant and cannot be handled by simple shadow rays [S2][S3, book].
*   **When Not to Use:** Real-time applications or scenes requiring strict energy conservation within the volume [S2].

## Background
Photon Mapping for Participating Media was introduced by Henrik Wann Jensen and Per Christensen [S1][S3, book]. It extends the original photon mapping method, which was limited to surfaces, to participating media by introducing a volume photon map [S1]. The method uses Monte Carlo simulation of photon transport from light sources, accounting for spatially varying optical properties and general phase functions [S3, book]. It was developed to address the limitations of previous methods that were less useful for volume rendering of spatially varying data [S3, book]. The approach has been used in research for simulating fog effects and other atmospheric phenomena [S1].

## Sources
[S1] A survey on participating media rendering techniques, https://doi.org/10.1007/s00371-005-0287-1, fetched 2026-10-26
[S2] Refractive radiative transfer equation, https://doi.org/10.1145/2557605, fetched 2026-10-31
[S3] Local and Global Illumination in the Volume Rendering Integral, https://avishek.net/assets/papers/nerfs/local-and-global-illumination-rendering-integral.pdf, fetched 2026-10-27 (book)
[S4] 348B Project Report, http://www-graphics.stanford.edu/courses/cs348b-competition/cs348b-02/ong/, fetched 2026-10-30 (unofficial)
