# Spherical Harmonics Irradiance Encoding

Also called: Good and Taylor Spherical Harmonics

## Overview
Spherical Harmonics Irradiance Encoding is a technique for encoding full spherical irradiance information using spherical harmonic basis functions, serving as the most generic method to represent directional surface lighting [S1, unofficial]. It solves the problem of modeling how irradiance changes with surface normal, which is required to provide indirect lighting for dynamic objects where the lighting value must be known for every possible surface orientation [S1, unofficial].

## How it works
**Inputs:** Incoming radiance/irradiance field over the sphere [G].
**Outputs:** Spherical harmonic coefficients stored in textures [S1, unofficial].

**Data Structures:**
*   **SH Texture:** A texture array or multi-channel texture storing the coefficients [G].
    *   **2nd Order:** 4 coefficients per texel [S1, unofficial].
    *   **3rd Order:** 9 coefficients per texel [S1, unofficial].

**Steps:**
1.  **Basis Selection:** Choose the order of spherical harmonics ($L$) [G].
    *   For $L=2$, use 4 coefficients.
    *   For $L=3$, use 9 coefficients [S1, unofficial].
2.  **Projection:** Project the incoming irradiance signal onto the chosen spherical harmonic basis functions to compute the coefficients [G].
3.  **Storage:** Store the computed coefficients in a texture map [S1, unofficial].
4.  **Evaluation:** At runtime, for a given surface normal $\mathbf{n}$, evaluate the spherical harmonic basis functions at $\mathbf{n}$ and dot product with the stored coefficients to reconstruct the irradiance [G].

**Parameters:**
*   **Order ($L$):**
    *   $L=3$: High quality, significant storage and bandwidth cost [S1, unofficial].
    *   $L=2$: Lower cost, but loss of subtleties, reduced lighting contrast, and less pronounced normal maps [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Ringing:** Both spherical harmonics and H-Basis approaches can exhibit ringing artifacts. Prefiltering can mitigate this but leads to unwanted smoothing of the lighting [S1, unofficial].
*   **Cost:** Both approaches present relatively high memory and bandwidth costs [S1, unofficial].

## Subtopics

### Halo 3 Irradiance Variation
A variation of spherical harmonic encoding utilized in Halo 3 to achieve the quality level of 3rd order spherical harmonics at a much lower cost [S1, unofficial]. It extracts the most dominant light and stores it separately as color and direction [S1, unofficial]. The residual data is then encoded in 2nd order spherical harmonics [S1, unofficial]. This results in very little quality loss at a much lower cost compared to full 3rd order encoding [S1, unofficial].

## Operating characteristics
*   **Memory/Bandwidth:** High. 3rd order requires 9 coefficients per texel; 2nd order requires 4 [S1, unofficial].
*   **Quality:** 3rd order provides very high quality; 2nd order loses subtleties and contrast [S1, unofficial].
*   **Artifacts:** Susceptible to ringing artifacts, requiring prefiltering which smooths lighting [S1, unofficial].

## Background
Good and Taylor presented the approach of encoding full spherical irradiance information using spherical harmonics [S1, unofficial]. Chen and Liu presented a variation of this method, which they utilized in Halo 3 [S1, unofficial].

## Sources
[S1] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
