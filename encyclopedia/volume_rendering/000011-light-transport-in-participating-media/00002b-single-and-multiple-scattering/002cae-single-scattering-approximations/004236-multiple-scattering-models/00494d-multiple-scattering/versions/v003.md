# Multiple Scattering

Also called: Multiple Scattering Simulation

## Overview
Multiple scattering simulates the effect resulting from more than one scattering bounce, where photons interact with the medium repeatedly before escaping [S1][S2, unofficial]. It solves the problem of rendering dense participating media, such as thick clouds or smoke, which appear dark and lifeless under single scattering approximations because they fail to account for the light trapped and re-emitted by internal bounces [S2, unofficial]. Unlike single scattering, which is sufficient for thin media like light haze, multiple scattering is required to capture the high reflectivity of dense volumes (e.g., cumulus clouds reflecting up to 90% of incident light) [S2, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs:** Volume density fields (extinction $\sigma_t$, scattering $\sigma_s$, absorption $\sigma_a$), phase function $p(\omega_i, \omega_o)$, and scene geometry/lighting.
*   **Outputs:** Radiance $L$ at the camera pixel.

**Data Structures**
*   **Volume Trace Depth:** An integer parameter controlling the maximum number of scattering bounces simulated per ray [S2, unofficial].
*   **Single Scattering Albedo:** $\omega_0 = \sigma_s / \sigma_t$, representing the probability of scattering versus absorption [Parent].

**Method Steps**
1.  **Ray Initialization:** Cast a primary ray from the camera into the volume.
2.  **Marching Loop:** Step through the volume density field.
3.  **Scattering Event:** At each step, determine if a scattering event occurs based on $\sigma_t$ and the random number generator.
4.  **Bounce Counting:** If a scattering event occurs:
    *   Sample the outgoing direction $\omega_o$ using the phase function $p(\omega_i, \omega_o)$ [Parent].
    *   Increment the bounce counter.
    *   If the bounce counter $\le$ Volume Trace Depth, continue tracing the new ray segment [S2, unofficial].
    *   If the bounce counter > Volume Trace Depth, terminate the path (or treat remaining energy as absorbed, depending on implementation) [S2, unofficial].
5.  **Accumulation:** Accumulate the scattered radiance contribution from each bounce into the pixel color.

**Parameters**
*   **Volume Trace Depth:** Controls the number of scattering bounces. Typical tested values include 1 (single scattering), 5, and 31 [S2, unofficial].
*   **Scattering Strength:** The magnitude of $\sigma_s$ must be high enough for the intensity of scattered light to be visible [S2, unofficial].

**Pitfalls**
*   **Noise:** Unbiased rendering of multiple scattering is inherently noisy due to the high variance of long paths [S1].
*   **Performance:** Computational cost multiplies with every added bounce [S2, unofficial].
*   **Denoising Difficulty:** Volumes are harder to denoise than surfaces because helper data like surface normals does not exist inside a cloud [S2, unofficial].
*   **Default Behavior:** In some engines (e.g., Cycles), the number of bounces is zero by default, effectively disabling multiple scattering [S1].

## Operating characteristics
*   **Complexity:** Time complexity increases exponentially with the number of bounces (Volume Trace Depth) [S2, unofficial].
*   **Memory:** Depends on the volume data structure (e.g., OpenVDB) but generally requires significant memory for high-resolution density fields [S2, unofficial].
*   **Strengths:** Physically accurate for dense, optically thick media where light is trapped and diffused [S1][S2, unofficial].
*   **Limitations:** Computationally expensive; noisy without significant sampling or denoising; not suitable for real-time without approximation [S1][S2, unofficial].
*   **When to use:** Rendering thick clouds, smoke, or subsurface scattering in materials like skin or milk (though subsurface scattering methods may be more efficient for the latter) [S1][S2, unofficial].
*   **When not to use:** Thin media like light haze (single scattering is sufficient) or real-time applications requiring high frame rates without heavy denoising [S1][S2, unofficial].

## Background
Multiple scattering is a fundamental concept in radiative transfer theory. In computer graphics, it became a standard feature in unbiased renderers like Blender's Cycles to address the visual shortcomings of single scattering in dense volumes [S1]. The implementation often relies on random walk methods for accuracy [S1]. The data format for transporting volume data between simulation and rendering engines is commonly OpenVDB, originally developed at DreamWorks Animation and open-sourced in 2012 [S2, unofficial].

## Sources
[S1] Volumes — Blender Manual, https://www.blender.org/manual/nb/3.6/render/materials/components/volume.html, fetched 2026-10-26
[S2] What Is Volumetric Rendering? Volume Shaders Explained, from ... - Maxon, https://www.maxon.net/en/article/volumetric-rendering, fetched 2026-10-26 (unofficial)
