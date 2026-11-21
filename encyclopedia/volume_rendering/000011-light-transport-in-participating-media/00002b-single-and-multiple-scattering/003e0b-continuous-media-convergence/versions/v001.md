# Continuous Media Convergence

Also called: Continuous approximation of discrete media, Particle-to-continuum convergence

## Overview
Continuous Media Convergence describes the phenomenon where a discrete medium composed of a large number of scattering particles renders with an appearance statistically indistinguishable from a homogeneous continuous medium [S1]. It solves the problem of efficiently rendering dense particulate scenes by allowing the use of traditional volumetric path tracing with global, averaged radiative transfer parameters instead of explicit particle geometry [S1]. It differs from [Delta tracking](985-delta-tracking.md) by replacing spatial queries with uniform or analytically derived coefficients, and from [Single-scattering homogeneous volume rendering](1135-single-scattering-homogeneous-volume-rendering.md) by handling multiple scattering regimes through the convergence limit [S1].

## How it works
**Inputs:**
*   Particle radius $r$.
*   Total particle count $N_{tot}$.
*   Bounding volume $V$ containing the particles.
*   Single-particle extinction coefficient $C_t(r)$.
*   Single-particle scattering coefficient $C_s(r)$.
*   Single-particle phase function $p(\Theta)$, stored in a lookup table [S1].

**Outputs:**
*   Rendered image using standard volumetric path tracing.

**Data Structures:**
*   Global extinction coefficient $\sigma_t$: A scalar value derived from particle density.
*   Global scattering coefficient $\sigma_s$: A scalar value derived from particle density.
*   Phase function table: A 1D array mapping scattering angle $\Theta$ to probability density [S1].

**Steps:**
1.  **Derive Global Coefficients:** Calculate the effective extinction and scattering coefficients for the continuous medium based on the particle distribution [S1]:
    $$ \sigma_t = \frac{C_t(r) N_{tot}}{V} $$
    $$ \sigma_s = \frac{C_s(r) N_{tot}}{V} $$
    Where $C_t(r)$ and $C_s(r)$ are the extinction and scattering cross-sections for a single particle of radius $r$ [S1].

2.  **Compute Phase Function:** Calculate the phase function for the continuous approximation. This is typically derived from the single-particle phase function averaged over the distribution, stored in a table for sampling [S1].

3.  **Render:** Use the derived $\sigma_t$, $\sigma_s$, and phase function table as inputs to a standard volumetric path tracer. The renderer treats the medium as homogeneous with these constant coefficients [S1].

**Parameters:**
*   $N_{tot}$: The number of particles. Convergence to the continuous appearance is observed when $N_{tot} \ge 10^7$ [S1]. For very dense media ($N_{tot} = 10^8$), the appearance is smooth and matches the continuous limit [S1].

**Edge Cases and Pitfalls:**
*   **Low Density:** When $N_{tot}$ is small, the discrete nature of particles is visible (individual particles lit by lamps). The continuous approximation fails to capture this granularity [S1].
*   **Heterogeneity:** This method assumes a uniform distribution within volume $V$. It does not handle spatially varying particle density without modification [S1].

## Subtopics

### Homogeneous Media Derivative Simplification
A simplification for differentiating pixel intensities in homogeneous media where $\nabla \sigma_s = \nabla \sigma_t \equiv 0$ [S2, unofficial]. This causes the gradient term $\beta$ to vanish in the general algorithm [S2, unofficial]. The derivative of the zeroth-order radiance $L^{\wedge(0)}$ becomes $L^{\wedge(0)}(x, \omega) = e^{-\sigma_t D} L^{\wedge}_{es}(x_0, \omega) - D^{\wedge}_{\sigma_t} L_{es}(x_0, \omega)$, and the first-order radiance derivative $L^{\wedge(1)}$ becomes $\int_0^D e^{-\sigma_t \tau} \sigma_s L^{\wedge}_{ins}(x', \omega) d\tau + [e^{-\sigma_t D} D L^{\wedge}_{rs}(x_0, \omega) + D^{\wedge}_{\sigma_s} L_{ins}(x_0, \omega) - \sigma_t L_{rs}(x_0, \omega)]$ [S2, unofficial]. This allows efficient gradient computation for optimization tasks [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Equivalent to standard volumetric path tracing with homogeneous media. No particle intersection tests are required [S1].
*   **Memory Complexity:** Low. Only global coefficients and a phase function table are stored, independent of $N_{tot}$ [S1].
*   **Performance:** Rendering time is constant regardless of particle count once the continuous approximation is valid [S1].
*   **Strengths:** Efficiently renders extremely dense media ($N_{tot} > 10^7$) without geometric overhead [S1].
*   **Limitations:** Fails to render the "grainy" appearance of sparse particle distributions. Cannot represent individual particle occlusion or specific geometric arrangements [S1].
*   **When to use:** When the particle count is high enough that the medium appears hazy/smooth, and individual particle visibility is not required [S1].

## Background
The concept originates from the observation that discrete participating media converge to continuous media in the limit of high particle concentration [S1]. It is used in rendering pipelines to optimize scenes with dense fog, smoke, or particulate matter where explicit particle tracking is computationally prohibitive [S1]. The method demonstrates that traditional volumetric rendering is the correct limit for dense discrete media [S1].

## Sources
[S1] Rendering discrete participating media using geometrical optics approximation, https://doi.org/10.1007/s41095-021-0253-5, fetched 2026-10-26
[S2] PDF A Differential Theory of Radiative Transfer, https://cseweb.ucsd.edu/~ravir/diff_rendering.pdf, fetched 2026-10-29 (unofficial)
