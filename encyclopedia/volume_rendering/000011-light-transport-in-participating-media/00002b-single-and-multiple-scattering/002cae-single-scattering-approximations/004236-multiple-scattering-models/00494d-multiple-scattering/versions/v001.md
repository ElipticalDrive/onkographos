# Multiple Scattering

Also called: Multiple Scattering Simulation

## Overview
Multiple scattering simulates light transport in participating media where photons undergo more than one scattering event before escaping the volume [S1][S2, unofficial]. While single scattering suffices for thin media like light haze, multiple scattering is required to render dense media such as clouds or smoke, which can reflect up to 90% of incident light through dozens or hundreds of bounces [S2, unofficial]. It is distinguished from single scattering approximations by its ability to capture the bright, diffuse appearance of thick volumes, though it incurs significantly higher computational costs and noise [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Volume density field, scattering coefficient $\sigma_s$, absorption coefficient $\sigma_a$, phase function $p(\omega', \omega)$, and a maximum bounce depth parameter (Volume Trace Depth).
*   **Output:** Radiance $L$ at the camera ray origin, including direct illumination and indirect scattered light contributions.

**Data Structures:**
The method relies on the standard volume rendering integral, extended to account for multiple bounces. The radiance $L(\mathbf{x}, \omega)$ at position $\mathbf{x}$ in direction $\omega$ is computed by integrating along the ray. For multiple scattering, the in-scattering term is not approximated by a single direction but includes contributions from all directions $\omega'$, weighted by the phase function and scattering coefficient [Parent].

**Steps:**
1.  **Ray Marching:** Cast a ray from the camera into the volume. Sample the density $\sigma_t(\mathbf{x}) = \sigma_s(\mathbf{x}) + \sigma_a(\mathbf{x})$ at discrete intervals.
2.  **Transmittance Calculation:** Compute the optical depth $\tau$ along the ray segment to determine the probability of light transmission without interaction.
3.  **Scattering Event:** If a scattering event occurs (determined by comparing a random number to $e^{-\tau}$), calculate the new direction $\omega'$ based on the phase function $p(\omega', \omega)$.
4.  **Recursive Contribution:**
    *   If the current bounce count is less than the **Volume Trace Depth**, recursively trace a new ray in direction $\omega'$ to gather indirect illumination [S2, unofficial].
    *   If the bounce count equals the **Volume Trace Depth**, terminate the recursion (or use a Russian Roulette termination if unbiased) [S1][S2, unofficial].
5.  **Accumulation:** Accumulate the scattered radiance, weighted by the transmittance from the previous interaction point and the scattering coefficient.

**Parameters:**
*   **Volume Trace Depth:** Controls the maximum number of scattering bounces simulated. Typical values tested include 1 (single scattering), 5, and 31 [S2, unofficial]. Higher values increase accuracy for dense media but also increase render time [S2, unofficial].
*   **Scattering Strength:** The scattering coefficient must be sufficiently high for the intensity of scattered light to be visible [S2, unofficial].

**Pitfalls:**
*   **Noise:** Unbiased rendering of multiple scattering is inherently noisy due to the variance introduced by random walk paths [S1].
*   **Performance:** Every added bounce multiplies the computational work, making dense volume rendering expensive [S2, unofficial].
*   **Denoising Difficulty:** Volumes are harder to denoise than surfaces because helper data such as surface normals is not available inside the volume [S2, unofficial].
*   **Zero Bounces:** By default, some renderers set the number of bounces to zero, resulting in single scattering only, which renders dense clouds dark and lifeless [S1].

## Operating characteristics
*   **Complexity:** Time complexity grows exponentially with the number of bounces and the optical thickness of the medium. Memory complexity is dominated by the volume grid storage.
*   **Performance:** Rendering is significantly slower than single scattering. The cost is proportional to the number of bounces and the density of the medium [S2, unofficial].
*   **Strengths:** Accurately renders dense, optically thick media like cumulus clouds, smoke, and subsurface scattering materials where light penetrates deeply [S1][S2, unofficial].
*   **Limitations:** High noise levels require many samples or denoising. Denoising is less effective for volumes than surfaces [S2, unofficial]. Not suitable for real-time rendering without approximations or hardware acceleration [S2, unofficial].
*   **When to use:** For offline rendering of clouds, smoke, or materials like skin/milk where subsurface scattering is modeled via volume (though dedicated subsurface scattering methods may be more efficient) [S1][S2, unofficial].
*   **When not to use:** For thin media like haze where single scattering is sufficient, or in real-time applications where performance is critical and approximations are preferred [S1][S2, unofficial].

## Background
Multiple scattering is a fundamental concept in radiative transfer theory. In computer graphics, it is implemented to overcome the limitations of single scattering approximations, which fail to capture the brightness of dense media. The Blender Manual notes that while subsurface scattering is often preferred for materials with well-defined surfaces (like skin), volume rendering with multiple scattering is necessary for media without clear boundaries, such as clouds and smoke [S1]. The concept is widely supported in unbiased renderers, though often disabled by default due to performance costs [S1].

## Sources
[S1] Volumes — Blender Manual, https://www.blender.org/manual/nb/3.6/render/materials/components/volume.html, fetched 2026-10-26
[S2] What Is Volumetric Rendering? Volume Shaders Explained, from ... - Maxon, https://www.maxon.net/en/article/volumetric-rendering, fetched 2026-10-26 (unofficial)
