# Volume Scatter

Also called: Volume Scatter node, Volumetric Scattering

## Overview
Volume Scatter is a volumetric shading component that models the redirection of light as it interacts with particles within a participating medium [S1]. It solves the problem of simulating how light propagates through semi-transparent media like smoke, fog, or clouds by scattering incoming radiance into other directions, distinct from pure absorption or emission [S1]. It is distinguished from [Volume Absorption](link) by its role in redistributing light energy rather than solely attenuating it, and from [Emission](link) by its reliance on incoming light rather than generating it [S1].

## How it works
**Inputs and Outputs**
The method takes the incident radiance $L_i$ from the scene and the medium's properties to compute the scattered radiance contribution. Key inputs include:
*   **Density ($\sigma_t$):** The total extinction coefficient, determining the probability of interaction [S1].
*   **Anisotropy ($g$):** A scalar parameter defining the scattering phase function's directionality [S1].
*   **Color:** The albedo or scattering color of the medium [S1].

**Data Structures**
The implementation typically relies on a scalar field for density and a scalar or vector for color. The anisotropy is stored as a single floating-point value.

**Parameters**
*   **Anisotropy ($g$):**
    *   $g = 0$: Isotropic scattering; light scatters evenly in all directions, similar to a diffuse BSDF [S1].
    *   $g < 0$: Backward scattering; light is more likely to scatter in the opposite direction of incidence [S1].
    *   $g > 0$: Forward scattering; light is more likely to scatter in the same direction as incidence [S1].
*   **Density:** Controls the frequency of interactions. Higher density results in more scattering and absorption events over a given path length [S1].

**Method Steps**
1.  **Sample Interaction Distance:** Determine the distance to the next scattering event using the medium's extinction coefficient $\sigma_t$. This is typically done via [Homogeneous Medium Sampling](980-homogeneous-medium-sampling.md) or [Delta tracking](985-delta-tracking.md) for inhomogeneous media.
2.  **Evaluate Phase Function:** Sample a new direction $\boldsymbol{\omega}_s$ based on the anisotropy parameter $g$. For $g=0$, this is a uniform distribution over the hemisphere or sphere. For $g \neq 0$, a phase function such as Henyey-Greenstein is used to bias the direction [S1].
3.  **Compute Scattering Contribution:** Calculate the radiance scattered into the view direction. This involves multiplying the incoming radiance $L_i$ by the scattering coefficient $\sigma_s$ (derived from $\sigma_t$ and the albedo), the phase function value $p(\boldsymbol{\omega}_i \to \boldsymbol{\omega}_s)$, and the transmittance to the interaction point.
4.  **Accumulate:** Add the scattered radiance to the accumulated color, accounting for the transmittance of the remaining path to the camera.

**Pitfalls**
*   **Anisotropy Range:** Values outside the typical range $[-1, 1]$ may cause physical inaccuracies or numerical instability depending on the phase function implementation.
*   **Density Scaling:** High density values without corresponding density field scaling can lead to opaque volumes where scattering is indistinguishable from absorption.

## Subtopics

### Volume Diffuse Limit
A variant or control parameter that limits the number of diffuse bounces light is allowed to undergo within the volume [S2].
*   **Definition:** Controls the extent of multiple scattering bounces inside the volume, analogous to the diffuse limit for surface materials [S2].
*   **Parameter:** `Volume limit` (integer). Default is 0. Values 1, 2, 3, and 4 allow increasing numbers of bounces [S2].
*   **Usage:** Increasing the limit makes the volume brighter by allowing more light to transmit through and bounce within the medium [S2].
*   **Pitfalls:** The visual difference between 0 and 1 is dramatic, while the difference between 1 and 2 is much less. Values should generally not exceed 4 due to diminishing returns and increased render time [S2].

## Operating characteristics
*   **Complexity:** The cost is proportional to the number of scattering events sampled, which depends on the optical depth and the chosen path depth limit.
*   **Performance:** Higher anisotropy values (forward scattering) may require more samples to resolve sharp peaks in the phase function if not sampled efficiently. Higher `Volume limit` values significantly increase render time as they require tracing additional bounces [S2].
*   **Strengths:** Provides physically plausible scattering for smoke and clouds [S1].
*   **Limitations:** Without multiple scattering approximations or high bounce limits, volumes may appear too dark or lack internal illumination [S2].

## Background
Volume Scatter is a standard component in volumetric shading systems, widely implemented in renderers such as Blender and Houdini [S1, S2]. It is based on the physical principle that light interacts with particles in a medium, redirecting energy according to the medium's properties. The concept of anisotropy in scattering is rooted in radiative transfer theory, where the Henyey-Greenstein phase function is commonly used to model forward or backward scattering biases [S1].

## Sources
[S1] Volumes — Blender Manual, https://www.blender.org/manual/nb/3.6/render/materials/components/volume.html, fetched 2026-10-26
[S2] Rendering volumes - SideFX, https://www.sidefx.com/docs/houdini/render/volumes.html, fetched 2026-10-29
