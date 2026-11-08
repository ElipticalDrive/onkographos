# Monte Carlo Method for Multiple Scattering

Also called: Monte Carlo Method for Directional Scattering, Monte Carlo ray tracing, Two-pass Monte Carlo

## Overview
The Monte Carlo method for multiple scattering solves the integral equation for directional scattering intensity $I(X, \omega)$ by stochastically tracing rays that undergo random absorption or scattering events based on the medium's optical properties [S1, unofficial][S2]. It is distinguished from deterministic solvers (such as the P-N method or Diffusion Approximation) by its ability to handle complex phase functions and geometries without discretizing the angular domain, though this comes at the cost of high variance and slow convergence [S1, unofficial][S2].

## How it works
**Inputs and Outputs**
*   **Inputs:** Extinction coefficient $\tau$, albedo $a$, phase function $p$, light source positions/intensities, and volume geometry.
*   **Outputs:** Pixel intensity values representing the radiance reaching the eye.

**Data Structures**
*   **Texture Map:** A 2D or 3D buffer used to store intermediate radiance. In the two-pass variant, it stores either isotropic scattering intensity or directionally scattered texture $I(X_i, \omega)$ [S1, unofficial][S2].
*   **Ray State:** Contains current position, direction, and accumulated throughput (weight).

**Algorithm Steps**
The method can be implemented as a direct path tracer or a two-pass hybrid approach. The two-pass approach is preferred to mitigate the low probability of eye rays connecting to light sources [S1, unofficial][S2].

1.  **First Pass (Light Propagation):**
    Trace rays from light sources into the volume.
    *   For each ray, determine interaction at voxel $X_i$.
    *   Sample absorption or scattering based on extinction coefficient $\tau$ and albedo $a$ [S1, unofficial][S2].
    *   If scattering occurs, compute the scattered radiance using the phase function $p$ [S1, unofficial][S2].
    *   Store the scattered contribution in a texture map. Blasi et al. stored an isotropic scattering texture, but the method generalizes to storing directionally scattered texture $I(X_i, \omega)$ [S1, unofficial][S2].

2.  **Second Pass (Eye Rendering):**
    Trace rays from the eye through each pixel.
    *   If a ray reaches a light source or volume emitter directly, add its flux to the pixel intensity [S1, unofficial][S2].
    *   If the ray intersects a voxel, sample the stored texture map to retrieve the pre-computed scattered radiance $I(X_i, \omega)$ [S1, unofficial][S2].
    *   Alternatively, perform one extra directional bounce toward the viewpoint using Monte Carlo sampling if a hybrid approach with the zonal method is used [S1, unofficial].

**Formulas**
The probability of absorption or scattering is governed by the extinction coefficient $\tau$ and albedo $a$. The scattered radiance contribution involves the phase function $p$. The specific integral equation for $I(X, \omega)$ is solved by summing contributions from rays that successfully connect the eye to a light source via scattering events [S1, unofficial][S2].

**Parameters**
*   **Number of Samples:** High sample counts are required to decrease variance [S1, unofficial][S2].
*   **Texture Resolution:** Determines the fidelity of the intermediate scattering data [S1, unofficial][S2].

**Edge Cases and Pitfalls**
*   **Noise:** Contributing rays are a small fraction of all considered rays, leading to noisy images [S1, unofficial][S2].
*   **Performance:** The method takes a long time to compute due to the need for many samples [S1, unofficial][S2].
*   **Connectivity:** Rays from the eye rarely end up at a light source, and rays from light sources rarely end up at the eye, necessitating the two-pass or hybrid approaches [S1, unofficial][S2].

## Subtopics

## Operating characteristics
*   **Time Complexity:** High. Requires many samples to reduce variance, making it computationally expensive [S1, unofficial][S2].
*   **Memory Complexity:** Depends on the texture map size in the two-pass variant. Storing directional textures $I(X_i, \omega)$ increases memory usage significantly compared to isotropic storage [S1, unofficial][S2].
*   **Strengths:** Handles complex phase functions and geometries naturally; no angular discretization required [S1, unofficial][S2].
*   **Limitations:** Noisy images; slow convergence; requires many samples for acceptable quality [S1, unofficial][S2].
*   **When to Use:** When high accuracy in complex scattering scenarios is needed and computational resources are sufficient [S1, unofficial][S2].
*   **When Not to Use:** When real-time performance is required or when the scene has simple scattering properties where deterministic methods suffice [S1, unofficial][S2].

## Background
The Monte Carlo method was originally developed by physicists for neutron transport and applied to rendering surface interreflection by Cook et al. and Kajiya [S1, unofficial][S2]. It was applied to volume applications by Rushmeier [S1, unofficial][S2]. The two-pass approach using caustic texture maps was proposed by Shirley, Chen et al., and Heckbert for surface interreflection, and adapted for volume rendering by Blasi et al. [S1, unofficial][S2].

## Sources
[S1] Optical Models for Direct Volume Rendering - Duke Computer Science, https://courses.cs.duke.edu/spring03/cps296.8/papers/max95opticalModelsForDirectVolumeRendering.pdf, fetched 2026-10-26 (unofficial)
[S2] Optical models for direct volume rendering (PDF) - Academia.edu, https://www.academia.edu/49467168/Optical_models_for_direct_volume_rendering, fetched 2026-10-27
