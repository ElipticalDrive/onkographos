# Planetary Surface Shading

Also called: Surface contribution to planetary rendering

## Overview
Planetary Surface Shading computes the radiance emitted or reflected by a planetary body's surface (ground or water) as seen by an observer, accounting for direct illumination, ambient scattering, and subsurface effects, before applying atmospheric attenuation [S1]. It distinguishes itself from pure atmospheric scattering by handling the distinct bidirectional reflectance distribution functions (BRDF) of diffuse terrain and specular water, while integrating precomputed spectral scattering data [S1].

## How it works
### Inputs and Outputs
**Inputs:**
*   Observer position $P_0$ and view direction $V$.
*   Light source direction $L$ (e.g., sun).
*   Wavelength $\lambda$.
*   Surface point $P_g$ on the planetary body.
*   Surface normal $N$ at $P_g$.
*   Diffuse ground color $I_D(\lambda)$.
*   Precomputed scattering textures for atmosphere and water.
*   Precomputed extinction lookup texture parameterized by observer altitude $h$ and zenith angle $\vartheta$ [S1].

**Outputs:**
*   Surface radiance contribution $I_P(P_0, V, L, \lambda)$.
*   Final fragment color $I_F$ after atmospheric attenuation [S1].

### Data Structures
1.  **Extinction Lookup Texture:** A 2D texture storing extinction $t(P P_c)$, parameterized by observer altitude $h$ and zenith angle $\vartheta$ [S1]. $P_c$ is the intersection of the ray starting at $P$ with zenith angle $\vartheta$ with the upper atmosphere boundary [S1].
2.  **Scattering Texture:** Stores atmospheric scattering contributions $I_S$ [S1].
3.  **Water Scattering Texture:** Stores water volume light $I_S^{\text{water}}$, indexed by depth $D$ [S1].

### Steps
1.  **Determine Surface Type:** Identify if $P_g$ is on ground or water [S1].
2.  **Compute Ambient and Direct Illumination:**
    *   Calculate attenuated direct illumination $I'_I(P_g, L, \lambda)$ [S1].
    *   Calculate ambient illumination $I_A(P_g, L, \lambda)$ caused by scattering [S1].
3.  **Compute Surface Radiance $I_P$:**
    *   **If Ground:** Apply the diffuse ground model [S1]:
        $$ I_P = I_D(\lambda)(-L \cdot N)I'_I(P_g, L, \lambda) + I_D(\lambda)I_A(P_g, L, \lambda) $$
        where $-L \cdot N$ is the dot product of the light direction and surface normal (assuming $L$ points from surface to light, or adjusted for sign convention such that the term is positive for lit surfaces) [S1].
    *   **If Water:** Apply the specular water model [S1]:
        $$ I_P = \hat{F}(\varphi)(I_S(P_g, V_R, L, \lambda) + I'_I(P_g, V_R, \lambda)) + I_S^{\text{water}}(P_g, V'_R, D, \lambda) $$
        *   Compute refraction ray $V_R$ using the `refract()` intrinsic function [S1].
        *   Determine $I'_I(P_g, V_R, \lambda)$: Set to $I'_I(P_g, L, \lambda)$ if $V_R$ hits the star disc, otherwise zero [S1].
        *   Fetch $I_S^{\text{water}}$ using the depth $D$ of the point where $V_R$ intersects the seabed [S1].
        *   $\hat{F}(\varphi)$ is a Fresnel approximation [S1].
4.  **Apply Atmospheric Attenuation:**
    *   Compute the path from surface point $P_g$ to observer $P_O$ [S1].
    *   Retrieve extinction $t(P_a P_g, \lambda)$ from the precomputed texture, where $P_a$ is the intersection of the view ray with the atmosphere boundary if the observer is outside, or $P_0$ if inside [S1].
    *   Calculate final fragment color:
        $$ I_F = I_S + I_P \cdot e^{-t(P_a P_g, \lambda)} $$
        where $I_S$ is the atmospheric scattering contribution fetched for the observer's position [S1].

### Formulas
*   **Ground Surface Radiance:**
    $$ I_P(P_0, V, L, \lambda) = I_D(\lambda)(-L \cdot N)I'_I(P_g, L, \lambda) + I_D(\lambda)I_A(P_g, L, \lambda) $$
    [S1]
*   **Water Surface Radiance:**
    $$ I_P(P_0, V, L, \lambda) = \hat{F}(\varphi)(I_S(P_g, V_R, L, \lambda) + I'_I(P_g, V_R, \lambda)) + I_S^{\text{water}}(P_g, V'_R, D, \lambda) $$
    [S1]
*   **Final Fragment Color:**
    $$ I_F = I_S + I_P \cdot e^{-t(P_a P_g, \lambda)} $$
    [S1]

### Parameters
*   $\hat{F}(\varphi)$: Fast and accurate Fresnel approximation from [Lazanyi and Szirmay-Kalos 2005], precomputable [S1].
*   $I_D(\lambda)$: Diffuse ground color [S1].
*   $I_S$: Atmospheric scattering contribution [S1].
*   $t$: Extinction coefficient, precomputed in a 2D texture [S1].

### Edge Cases and Pitfalls
*   **Observer Position:** If the observer is outside the atmosphere, the intersection point $P_a$ between the view ray and the atmosphere sphere must be evaluated, as there is no scattering or attenuation outside the atmosphere [S1].
*   **Spectral vs RGB:** Spectral precomputation yields more natural results than RGB precomputation, particularly during sunset when higher scattering orders are stronger, affecting lightness, chroma, and hue [S1].
*   **Refraction Ray:** The refraction ray $V_R$ must be correctly computed in the fragment shader to determine the correct water scattering and direct illumination contributions [S1].

## Operating characteristics
*   **Performance:** Relies heavily on precomputed textures (extinction, scattering, water scattering) to avoid costly real-time calculation of extinction coefficients and complex volume integration [S1].
*   **Memory:** Requires storage for multiple textures (extinction 2D texture, scattering textures, water scattering texture) [S1].
*   **Accuracy:** Spectral precomputation provides higher accuracy in color reproduction compared to RGB, especially for complex lighting conditions like sunset [S1].
*   **Limitations:** Assumes finely tessellated spheres for the planet and atmosphere [S1]. The method is specific to the planetary rendering context described in [S1].

## Background
This method is part of a real-time spectral scattering framework for large-scale natural participating media, specifically designed for rendering planets and their atmospheres [S1]. It integrates surface shading with atmospheric scattering to provide a cohesive visual result [S1].

## Sources
[S1] Real-time spectral scattering in large-scale natural participating media, https://doi.org/10.1145/1925059.1925074, fetched 2026-10-28
