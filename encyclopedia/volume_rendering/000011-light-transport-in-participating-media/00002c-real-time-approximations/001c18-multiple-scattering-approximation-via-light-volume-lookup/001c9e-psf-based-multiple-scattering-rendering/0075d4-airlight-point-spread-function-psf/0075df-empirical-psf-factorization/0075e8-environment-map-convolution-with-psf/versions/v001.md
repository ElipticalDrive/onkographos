# Environment Map Convolution with PSF

Also called: Distant Environment Map Convolution, Spherical Harmonics PSF Convolution

## Overview
This method approximates the appearance of complex, distant lighting in participating media by convolving a distant environment map with the medium's Point Spread Function (PSF). It solves the problem of efficiently rendering atmospheric scattering effects (glows and blurriness) for high-frequency lighting without per-pixel volumetric integration. It differs from direct integration methods by operating in the frequency domain (Spherical Harmonics) and assumes all light sources are equidistant from the scene.

## How it works
**Inputs:**
*   Original environment map $L^{original}$, represented as Spherical Harmonic (SH) coefficients $L_{lm}^{original}$.
*   PSF coefficients $PSF_l$ derived from the medium's optical thickness $T_{sv}$ and the normalized PSF (NPSF) angular component.

**Outputs:**
*   Final surface radiance $L^{final}$.

**Data Structures:**
*   **SH Coefficients:** The environment map is stored as a vector of coefficients $L_{lm}$, where $l$ is the band index ($0 \le l \le N$) and $m$ is the order ($-l \le m \le l$). The total number of coefficients is $(N+1)^2$.
*   **PSF Coefficients:** The PSF is represented by radial coefficients $PSF_l$, one per band $l$, independent of $m$ due to the rotational symmetry of the single-scattering PSF approximation.

**Steps:**
1.  **Represent Lighting in SH:** Convert the distant environment map into Spherical Harmonic coefficients $L_{lm}^{original}$. This assumes the environment map represents a distant sphere where all light sources are at infinite distance relative to the scene scale.
2.  **Compute PSF Coefficients:** Calculate the radial PSF coefficients $PSF_l$ for the current medium parameters. These coefficients encapsulate the scattering behavior (amplitude and angular spread) determined by the optical thickness $T_{sv}$ and the normalized PSF shape.
3.  **Convolve in Frequency Domain:** Compute the convolved environment map coefficients $L_{lm}^{convolved}$ using the following formula [S1]:
    $$
    L_{lm}^{convolved} = \sqrt{\frac{4\pi}{2l+1}} PSF_l L_{lm}^{original}
    $$
    Here, $l$ is the SH band index. The term $\sqrt{\frac{4\pi}{2l+1}}$ is the normalization factor for the SH basis functions. Note that the convolution is diagonal in the SH domain because the PSF is rotationally symmetric; thus, it scales each band $l$ uniformly across all orders $m$.
4.  **Compute Attenuated Direct Lighting:** Calculate the direct transmission component by attenuating the original environment map coefficients by the optical thickness $T_{sv}$ along the view ray. In the SH domain, this is applied to the coefficients:
    $$
    L_{lm}^{attenuated} = L_{lm}^{original} e^{-T_{sv}}
    $$
    [S1]
5.  **Reconstruct Radiance:** Evaluate the SH basis functions at the view direction $\omega_v$ to reconstruct the radiance values:
    $$
    L^{attenuated}(\omega_v) = \sum_{l,m} L_{lm}^{attenuated} Y_{lm}(\omega_v)
    $$
    $$
    L^{convolved}(\omega_v) = \sum_{l,m} L_{lm}^{convolved} Y_{lm}(\omega_v)
    $$
6.  **Combine Components:** Sum the attenuated direct lighting and the convolved airlight to get the final shading:
    $$
    L^{final} = L^{attenuated} + L^{convolved}
    $$
    [S1]

**Parameters:**
*   **SH Order ($N$):** Determines the frequency detail of the environment map and PSF. Higher orders capture sharper glows but increase computational cost. Typical values depend on the required visual fidelity, often $N=4$ to $N=8$ for real-time applications.
*   **Optical Thickness ($T_{sv}$):** The integral of extinction along the view ray. This drives the attenuation $e^{-T_{sv}}$ and the PSF coefficients.

**Edge Cases and Pitfalls:**
*   **Non-Distant Lights:** The method assumes all light sources are equidistant (distant environment map). If lights are close, the PSF convolution is invalid because the scattering geometry changes with distance. [S1]
*   **Precision:** SH reconstruction can introduce ringing (Gibbs phenomenon) if the environment map has high-frequency discontinuities and the SH order is insufficient.
*   **PSF Validity:** The PSF coefficients $PSF_l$ must be precomputed or computed analytically for the specific medium parameters. Using incorrect PSF coefficients results in physically inaccurate scattering.

## Subtopics

### Precomputed Radiance Transfer with Participating Media
This variant integrates the PSF convolution into precomputed radiance transport (PRT) methods. Instead of rendering a simple environment map, it convolves the lighting with the PSF before feeding it into PRT techniques (e.g., separable BRDF approximation) that handle complex shadows and interreflections. This allows for the inclusion of participating media effects in scenes with complex geometry and materials, where standard PRT would ignore scattering. The key distinction is the use of the convolved lighting as input to a precomputed transfer function, enabling interactive rendering of complex scenes with fog/mist. [S1]

## Operating characteristics
*   **Time Complexity:** The convolution step is $O(L^2)$ where $L$ is the number of SH bands (since it scales each band). SH evaluation is $O(L^2)$. This is significantly faster than per-pixel volumetric integration, which is $O(N_{samples})$.
*   **Memory Complexity:** Requires storing SH coefficients for the environment map ($O(L^2)$) and PSF coefficients ($O(L)$). Minimal additional memory compared to standard SH rendering.
*   **Performance:** Enables real-time rendering of complex lighting with atmospheric effects. Performance is largely independent of the number of lights in the environment map, as they are compressed into SH coefficients.
*   **Strengths:** Efficient handling of high-frequency lighting; simple implementation; compatible with existing SH-based rendering pipelines.
*   **Limitations:** Only valid for distant lights; assumes single scattering or a specific PSF approximation; cannot handle complex shadowing from the medium itself (shadows are handled by the underlying PRT or direct attenuation).
*   **When to Use:** Real-time applications with distant environment maps and participating media where single-scattering approximation is acceptable.
*   **When Not to Use:** Scenes with close-range light sources, or when multiple scattering effects dominate and cannot be approximated by the PSF.

## Background
The method originates from the "Practical Analytic Single Scattering Model for Real Time Rendering" by Ramamoorthi et al. (2001). It leverages the property that convolution with a rotationally symmetric function (like the single-scattering PSF) is diagonal in the Spherical Harmonics domain. This allows for efficient precomputation and real-time evaluation of atmospheric scattering effects for complex lighting. The approach was extended to integrate with Precomputed Radiance Transfer techniques to handle complex geometry and materials. [S1]

## Sources
[S1] A Practical Analytic Single Scattering Model for Real Time Rendering, https://cseweb.ucsd.edu/~ravir/papers/singlescat/scattering.pdf, fetched 2026-10-29
