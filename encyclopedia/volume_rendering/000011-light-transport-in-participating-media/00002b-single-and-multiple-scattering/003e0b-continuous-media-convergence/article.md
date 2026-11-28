# Continuous Media Convergence


## Overview
Continuous Media Convergence describes the phenomenon where a discrete medium composed of a large number of scattering particles renders with an appearance statistically indistinguishable from a homogeneous continuous medium [S1]. It solves the problem of efficiently rendering dense particulate scenes by allowing the use of traditional volumetric path tracing with global, averaged radiative transfer parameters instead of explicit particle geometry [S1].

## How it works
**Inputs:**
*   Particle radius $r$ [S1].
*   Total particle count $N_{tot}$ [S1].
*   Bounding volume $V$ containing the particles [S1].
*   Single-particle extinction coefficient $C_t(r)$ [S1].
*   Single-particle scattering coefficient $C_s(r)$ [S1].

**Outputs:**
*   Rendered image using standard volumetric path tracing [S1].

**Data Structures:**
*   Global extinction coefficient $\sigma_t$: A scalar value derived from particle density [S1].
*   Global scattering coefficient $\sigma_s$: A scalar value derived from particle density [S1].
*   Phase function table: Stores the computed phase function for sampling [S1].

**Steps:**
1.  **Derive Global Coefficients:** Calculate the effective extinction and scattering coefficients for the continuous medium based on the particle distribution [S1]:
    $$ \sigma_t = \frac{C_t(r) N_{tot}}{V} $$
    $$ \sigma_s = \frac{C_s(r) N_{tot}}{V} $$
    Where $C_t(r)$ and $C_s(r)$ are the extinction and scattering cross-sections for a single particle of radius $r$ [S1].
2.  **Compute Phase Function:** Calculate the phase function in a similar way and store it in a table [S1].
3.  **Render:** Use the derived global properties to render using traditional volumetric path tracing [S1].

**Parameters:**
*   $N_{tot}$: The number of particles. Convergence to the continuous appearance is observed when $N_{tot} = 10^7$ [S1]. For very dense media ($r = 400\ \mu\text{m}$, $N_{tot} = 10^8$), the method produces a smooth appearance matching that from a continuous medium [S1].

**Edge Cases and Pitfalls:**
*   **Low Density:** When $N_{tot}$ is small, individual particles lit by lamps are clearly visible, and the appearance differs from the continuous limit [S1].
*   **Distribution Assumption:** The coefficient derivation assumes a uniform particle distribution within volume $V$ [S1].

## Subtopics

### Homogeneous Media Derivative Simplification
A simplification for differentiating pixel intensities in homogeneous media with $\pi$-invariant parameters where $\nabla \sigma_s = \nabla \sigma_t \equiv 0$ [S2, unofficial]. This causes the gradient term $\beta$ to vanish in Lines 7 and 20 of Algorithm 1 [S2, unofficial]. The derivative of the zeroth-order radiance $\hat{L}^{(0)}$ becomes $\hat{L}^{(0)}(x, \omega) = e^{-\sigma_t D} \hat{L}_{es}(x_0, \omega) - \hat{D}_{\sigma_t} L_{es}(x_0, \omega)$ [S2, unofficial]. The derivative of the first-order radiance $\hat{L}^{(1)}$ becomes $\hat{L}^{(1)}(x, \omega) = \int_0^D e^{-\sigma_t \tau} \sigma_s \hat{L}_{ins}(x', \omega) d\tau + [e^{-\sigma_t D} D \hat{L}_{rs}(x_0, \omega) + \hat{D}_{\sigma_s} L_{ins}(x_0, \omega) - \sigma_t L_{rs}(x_0, \omega)]$ [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Equivalent to standard volumetric path tracing with homogeneous media [S1].
*   **Memory Complexity:** Low. Only global coefficients and a phase function table are stored, independent of $N_{tot}$ [S1].
*   **Performance:** Rendering time is constant regardless of particle count once the continuous approximation is valid [S1].
*   **Strengths:** Efficiently renders extremely dense media ($N_{tot} \ge 10^7$) without geometric overhead [S1].
*   **Limitations:** Fails to render the "grainy" appearance of sparse particle distributions or resolve individual particle occlusion when $N_{tot}$ is small [S1].
*   **When to use:** When the particle count is high enough that the medium appears hazy or smooth, and individual particle visibility is not required [S1].

## Background
The concept originates from the observation that discrete participating media converge to continuous media in the limit of particle concentration [S1].

## Sources
[S1] Rendering discrete participating media using geometrical optics approximation, https://doi.org/10.1007/s41095-021-0253-5, fetched 2026-10-26
[S2] PDF A Differential Theory of Radiative Transfer, https://cseweb.ucsd.edu/~ravir/diff_rendering.pdf, fetched 2026-10-29 (unofficial)
