# Empirical PSF Factorization

Also called: Amplitude-Angular PSF Factorization, NPSF Factorization

## Overview
Empirical PSF Factorization decomposes the Airlight Point Spread Function (PSF) into a purely angular component (NPSF) and an amplitude component dependent on optical thickness. This separation allows the angular convolution to be precomputed once, enabling interactive changes to medium parameters (such as density or extinction coefficients) without recomputing expensive convolutions. It differs from direct integration methods by decoupling the geometric scattering profile from the medium's attenuation properties.

## How it works
The method relies on the observation that the PSF, when normalized by the term $T_{sve} e^{-T_{sve}}$, becomes largely independent of the medium's physical parameters and depends primarily on the scattering angle $\gamma$ [S1].

**Inputs and Outputs**
*   **Input:** Scattering angle $\gamma$; optical thickness $T_{sv}$ (or $T_{sve}$).
*   **Output:** PSF value $\text{PSF}(\gamma)$.

**Data Structures**
*   **NPSF Texture:** A 1D texture or lookup table storing values of $\text{NPSF}(\gamma)$ for $\gamma \in [0, \pi]$. The values are precomputed using Equation 21.
*   **Parameters:** $T_{sv}$ (optical thickness), $\beta$ (extinction coefficient, implicit in $T_{sv}$).

**Steps**
1.  **Precompute NPSF:** Calculate the Normalized PSF (NPSF) for all relevant angles $\gamma$. The NPSF is defined as the PSF at $T_{sv}=1$, normalized by the factor $T_{sve} e^{-T_{sve}}$ [S1]. The formula is:
    $$
    \text{NPSF}(\gamma) = \frac{F(\sin(\gamma, \frac{\pi}{2}) - F(\sin(\gamma, \frac{\gamma}{2}))}{2\pi \sin \gamma \cdot e^{(\cos \gamma - 1)}} \quad (21)
    $$
    where $F$ is a special function precomputed in textures [S1]. Note: The source text contains a likely typo in the numerator's argument structure ($F(\sin(\gamma, \frac{\pi}{2})$), but the implementation must follow the provided formula exactly as cited [S1].

2.  **Define Amplitude Component:** The amplitude component is defined as the normalization factor:
    $$
    A(T_{sv}) = T_{sv} e^{-T_{sv}} \quad (22)
    $$
    This component captures the dependence on the medium's optical thickness [S1].

3.  **Compute PSF:** Combine the amplitude and angular components to obtain the final PSF for any optical thickness $T_{sv}$:
    $$
    \text{PSF}(\gamma) = T_{sv} e^{-T_{sv}} \cdot \text{NPSF}(\gamma) \quad (22)
    $$
    This factorization ensures that $\text{NPSF}(\gamma)$ remains independent of other physical parameters, allowing it to be reused for different medium settings [S1].

**Parameters**
*   $T_{sv}$: Optical thickness. Typical values in validation range from 0.1 to 3.1 [S1].

**Edge Cases and Pitfalls**
*   **Small Angles:** The approximation error increases for smaller angles $\gamma$ [S1]. Implementers should be aware that the NPSF formula involves division by $\sin \gamma$, requiring careful handling of $\gamma \to 0$ to avoid numerical instability.
*   **Special Function F:** The term $F$ in Equation 21 is described as a "special function precomputed in textures" [S1]. The exact definition of $F$ is not provided in the main text but is referenced in Appendix C of the source. If Appendix C is unavailable, the implementation cannot fully realize Equation 21.
*   **Error Bounds:** The absolute approximation error remains well below 0.05 across the tested range of optical thicknesses [S1].

## Subtopics

### [Environment Map Convolution with PSF](23366-environment-map-convolution-with-psf.md)
This method approximates the appearance of complex, distant lighting in participating media by convolving a distant environment map with the medium's Point Spread Function (PSF).

## Operating characteristics
*   **Time Complexity:** $O(1)$ per pixel for PSF evaluation after precomputation, as it involves a texture lookup for NPSF and a simple arithmetic operation for the amplitude.
*   **Memory Complexity:** $O(N)$ for storing the NPSF lookup table, where $N$ is the number of angular samples.
*   **Strengths:** Enables real-time interaction with medium parameters (density, extinction) without re-rendering or recomputing convolutions.
*   **Limitations:** Approximation error increases at small scattering angles. Relies on the validity of the empirical factorization, which was validated for optical thicknesses between 0.1 and 3.1 [S1].
*   **When to use:** When interactive adjustment of participating media parameters is required, and the scene lighting can be approximated via the underlying Airlight PSF model.

## Background
The method was introduced in "A Practical Analytic Single Scattering Model for Real Time Rendering" by Ramamoorthi et al. [S1]. It is based on the empirical observation that the PSF, when normalized by $T_{sve} e^{-T_{sve}}$, becomes essentially independent of the medium's physical parameters (optical thickness) and depends largely on the angle $\gamma$ [S1]. This allows for the decoupling of the angular scattering profile from the medium's attenuation.

## Sources
[S1] A Practical Analytic Single Scattering Model for Real Time Rendering, https://cseweb.ucsd.edu/~ravir/papers/singlescat/scattering.pdf, fetched 2026-10-29
