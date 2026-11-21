# Volume Scatter

## Overview
Volume Scatter is a volumetric shading component that models the redirection of light as it interacts with particles within a participating medium [S1]. It simulates how light propagates through semi-transparent media like smoke or clouds by scattering incoming radiance into other directions [S1].

## How it works
**Inputs and Outputs**
The method takes incident radiance $L_i$ and medium properties to compute the scattered radiance contribution [G]. Key inputs are the density (total extinction coefficient $\sigma_t$), the anisotropy parameter $g$, and the medium color [S1]. The output is the scattered radiance added to the ray's accumulated color [G].

**Data Structures**
The implementation relies on a scalar field for density and a scalar or vector for color [S1]. The anisotropy is stored as a single floating-point value [G].

**Method Steps**
1. **Sample Interaction Distance:** Determine the distance to the next scattering event using the medium's extinction coefficient $\sigma_t$ [G].
2. **Evaluate Phase Function:** Sample a new direction $\boldsymbol{\omega}_s$ based on $g$ [G]. For $g=0$, use a uniform distribution over the sphere [G]. For $g \neq 0$, use a biased phase function [G].
3. **Compute Scattering Contribution:** Calculate radiance scattered into the view direction by multiplying $L_i$, the scattering coefficient $\sigma_s$, the phase function value $p(\boldsymbol{\omega}_i \to \boldsymbol{\omega}_s)$, and the transmittance to the interaction point [G].
4. **Accumulate:** Add the scattered radiance to the accumulated color, accounting for the transmittance of the remaining path to the camera [G].

**Formulas**
The total extinction coefficient is defined as $\sigma_t = \sigma_a + \sigma_s$, where $\sigma_a$ is the absorption coefficient and $\sigma_s$ is the scattering coefficient [G]. The transmittance over a path length $t$ is $T(t) = e^{-\tau}$, where $\tau = \int_0^t \sigma_t \, dt$ is the optical depth [G].

**Parameters**
*   **Anisotropy ($g$):** Controls scattering directionality [S1]. $g = 0$ scatters evenly in all directions (similar to a diffuse BSDF) [S1]. $g < 0$ scatters mostly backwards [S1]. $g > 0$ scatters mostly forwards [S1].
*   **Density:** Defines how much light interacts with the volume (absorbed or scattered) versus passing through [S1]. Typical values are not given in the sources.

**Pitfalls**
*   **Anisotropy Range:** Values outside $[-1, 1]$ may cause physical inaccuracies or numerical instability [G].
*   **Density Scaling:** High density without field scaling can make volumes opaque, obscuring scattering from absorption [G].

## Subtopics

### Volume Diffuse Limit
A control parameter that limits the number of diffuse bounces light undergoes inside the volume [S2].
*   **Definition:** Equivalent to the surface diffuse limit property, controlling internal light bounces [S2].
*   **Parameter:** `Volume limit` (integer). Default is 0 (no bounces, volume does not transmit light from behind) [S2]. Can be increased to 1, 2, 3, or 4 for more bounces [S2].
*   **Usage:** Increasing the value makes the volume brighter by transmitting more light [S2].
*   **Pitfalls:** The visual difference between 0 and 1 is dramatic, while 1 to 2 is much less [S2]. Values should generally not exceed 4 due to diminishing returns and increased render time [S2].

## Operating characteristics
*   **Complexity:** Cost is proportional to sampled scattering events, depending on optical depth and path depth limit [G].
*   **Performance:** Higher anisotropy (forward scattering) may require more samples to resolve sharp phase function peaks [G]. Higher `Volume limit` values significantly increase render time by tracing additional bounces [S2].
*   **Strengths:** Provides physically plausible scattering for smoke and clouds [S1].
*   **Limitations:** Without multiple scattering approximations or high bounce limits, volumes may appear too dark or lack internal illumination [S2].

## Background
Volume Scatter is a standard component in volumetric shading systems [G]. It is based on the physical principle that light interacts with particles in a medium, redirecting energy according to the medium's properties [G]. The concept of anisotropy in scattering is rooted in radiative transfer theory [G]. It is commonly used to shade white smoke or clouds [S1].

## Sources
[S1] Volumes — Blender Manual, https://www.blender.org/manual/nb/3.6/render/materials/components/volume.html, fetched 2026-10-26
[S2] Rendering volumes - SideFX, https://www.sidefx.com/docs/houdini/render/volumes.html, fetched 2026-10-29
