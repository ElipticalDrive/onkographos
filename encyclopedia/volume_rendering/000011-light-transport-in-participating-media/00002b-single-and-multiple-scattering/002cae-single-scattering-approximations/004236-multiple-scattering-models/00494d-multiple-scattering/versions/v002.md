# Multiple Scattering

Also called: Multiple Scattering Simulation

## Overview
Multiple scattering simulates the effect resulting from more than one scattering bounce within a participating medium [S1]. It solves the problem of radiance underestimation in optically thick media, where single scattering approximations render dense clouds dark and lifeless [S2, unofficial]. Unlike single scattering, which only accounts for direct illumination, multiple scattering includes contributions from light that has scattered one or more times, allowing for realistic brightness in thick media like cumulus clouds, which can reflect up to 90 percent of incident light [S2, unofficial].

## How it works
Multiple scattering is implemented by extending the ray marching process to account for additional scattering events. The core mechanism involves following a ray through the medium and, upon a scattering event, generating new secondary rays to simulate subsequent bounces.

**Inputs and Outputs:**
*   **Input:** Volume density field (e.g., OpenVDB), scattering coefficient $\sigma_s$, absorption coefficient $\sigma_a$, phase function $p$, and a maximum bounce depth parameter.
*   **Output:** Radiance $L$ accumulated along the ray.

**Data Structures:**
*   **Volume Trace Depth:** An integer parameter controlling the maximum number of scattering bounces simulated [S2, unofficial].
*   **Ray State:** Position $\mathbf{x}$, direction $\omega$, and accumulated radiance $L_{acc}$.

**Steps:**
1.  Initialize accumulated radiance $L_{acc} = 0$.
2.  Cast a primary ray into the volume.
3.  March along the ray, sampling the medium properties at each step.
4.  Determine if a scattering event occurs based on $\sigma_s$ and the step size.
5.  If a scattering event occurs:
    *   Sample a new direction $\omega'$ from the phase function $p(\omega, \omega')$.
    *   If the current bounce count is less than `Volume Trace Depth`:
        *   Recursively or iteratively trace a new ray in direction $\omega'$ to compute the in-scattered radiance [S2, unofficial].
        *   Add the result to $L_{acc}$, attenuated by the extinction coefficient $\sigma_t = \sigma_a + \sigma_s$ [S1].
    *   If the bounce count equals `Volume Trace Depth`, terminate the scattering chain (approximation) [S2, unofficial].
6.  Accumulate emitted radiance $L_e$ and transmittance along the path.
7.  Return $L_{acc}$.

**Parameters:**
*   **Volume Trace Depth:** Controls the number of bounces. Values such as 1 (single scattering), 5, and 31 have been tested on clouds with cylindrical light sources [S2, unofficial].
*   **Scattering Strength:** Must be high enough for the intensity of scattered light to be visible [S2, unofficial].

**Pitfalls:**
*   **Noise:** Unbiased rendering of multiple scattering is inherently noisy due to the high variance of random walks [S1].
*   **Denoising Difficulty:** Volumes are harder to denoise than surfaces because helper data like surface normals does not exist inside a cloud [S2, unofficial].
*   **Performance:** Every added bounce multiplies the computational work, making high bounce counts expensive [S2, unofficial].

## Operating characteristics
*   **Complexity:** Time complexity grows exponentially with the number of bounces, as every added bounce multiplies the work [S2, unofficial].
*   **Performance:** Unbiased multiple scattering is computationally expensive and often requires limiting the number of bounces to keep render times acceptable [S1].
*   **Strengths:** Produces realistic lighting for optically thick media like clouds and smoke, which single scattering fails to render correctly [S2, unofficial].
*   **Limitations:** High noise levels and slow convergence; unsuitable for real-time without approximation or denoising [S1][S2, unofficial].

## Background
The distinction between single and multiple scattering is fundamental in volume rendering. Single scattering simulates only one bounce, which is sufficient for thin media like light haze but fails for dense clouds [S2, unofficial]. Multiple scattering is required for real-world effects such as scattering in clouds or subsurface scattering [S1]. In practice, unbiased rendering of multiple scattering is often disabled by default (e.g., zero bounces in Cycles) due to noise, with subsurface scattering preferred for materials like skin or milk [S1].

## Sources
[S1] Volumes — Blender Manual, https://www.blender.org/manual/nb/3.6/render/materials/components/volume.html, fetched 2026-10-26
[S2] What Is Volumetric Rendering? Volume Shaders Explained, from ... - Maxon, https://www.maxon.net/en/article/volumetric-rendering, fetched 2026-10-26 (unofficial)
