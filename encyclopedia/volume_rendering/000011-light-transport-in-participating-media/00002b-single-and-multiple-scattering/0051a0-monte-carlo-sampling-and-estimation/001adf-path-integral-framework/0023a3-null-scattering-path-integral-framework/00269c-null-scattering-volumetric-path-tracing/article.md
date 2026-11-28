# Null-Scattering Volumetric Path Tracing

Also called: Volumetric Path Tracing with Null Scattering

## Overview
Null-Scattering Volumetric Path Tracing is an unbiased algorithm for rendering participating media that uses a homogeneous majorant medium with an added null-scattering term to sample distances along a ray [S1]. It solves the problem of efficiently sampling interaction points in sparse or heterogeneous media by treating absorption, scattering, and null-scattering as mutually exclusive events with probabilities proportional to their coefficients [S1]. It differs from standard delta tracking by explicitly maintaining a majorant transmittance factor $T_{maj}$ and sampling distances from the majorant distribution, ensuring unbiased estimates even when the majorant significantly overestimates the physical extinction [S1]. It builds on the [Null-Scattering Path Integral Framework](3070-null-scattering-path-integral-framework.md) [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Ray origin $p_0$, direction $\omega_0$, medium properties (absorption coefficient $\sigma_a$, scattering coefficient $\sigma_s$, emission $L_e$, phase function $p$), majorant coefficient $\sigma_{maj}$, and scene geometry [S1].
*   **Output:** Radiance $L_i(p_0, \omega_0)$ along the ray [S1].

**Data Structures:**
*   **Majorant Iterator:** Provides piecewise-constant $\sigma_{maj}$ segments along the ray [S1].
*   **State:** Current position $p$, current direction $\omega$, accumulated majorant transmittance $T_{maj}$, and current segment index in the majorant iterator [S1].

**Parameters:**
*   $\sigma_{maj}$: The majorant coefficient, defined as $\sigma_a(p) + \sigma_s(p) + \sigma_n(p)$ [S1]. It must be greater than or equal to the sum of absorption and scattering coefficients at all points in the medium [S1].
*   $\sigma_n$: The null-scattering coefficient, defined as $\sigma_{maj} - \sigma_a(p) - \sigma_s(p)$ [S1].

**Steps:**
1.  **Initialize:** Set current position $p = p_0$, direction $\omega = \omega_0$, and accumulated majorant transmittance $T_{maj} = 1.0$ [S1].
2.  **Sample Distance:** Sample a distance $t$ from the exponential distribution $p(t) = \sigma_{maj} e^{-\sigma_{maj} t}$ using the current $\sigma_{maj}$ value [S1].
3.  **Check Geometry:** Let $t_{geom}$ be the distance to the nearest surface intersection along $\omega$ [S1]. If $t > t_{geom}$, return surface radiance $L_o(p_s, \omega_0)$ where $p_s = p + t_{geom}\omega$ [S1].
4.  **Update Position:** Set $p' = p + t\omega$ [S1]. Update $T_{maj}$ by multiplying by $e^{-\sigma_{maj} t}$ [S1].
5.  **Evaluate Null-Scattering Source:** Evaluate $L_n(p', -\omega)$ [S1]. This involves selecting one of three terms based on probabilities $\sigma_a(p')/\sigma_{maj}$, $\sigma_s(p')/\sigma_{maj}$, and $\sigma_n(p')/\sigma_{maj}$ [S1]:
    *   **Absorption:** Return $L_e(p', \omega_0)$ and terminate the path [S1].
    *   **Scattering:** Sample a new direction $\omega'$ from the phase function $p(p', \omega', \omega_0)$ [S1]. Recursively evaluate $L_i(p', \omega')$ and return $\frac{\sigma_s(p')}{\sigma_{maj}} \frac{p(p', \omega', \omega_0)}{q(\omega')} L_i(p', \omega')$ where $q(\omega')$ is the PDF of the sampled direction [S1].
    *   **Null-Scattering:** Recursively evaluate $L_i(p', \omega_0)$ and return $\frac{\sigma_n(p')}{\sigma_{maj}} L_i(p', \omega_0)$ [S1].
6.  **Return:** Return the result from step 5 [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def null_scattering_path_trace(p0, omega0, majorant_iterator, geometry):
    T_maj = 1.0
    p, omega = p0, omega0
    while True:
        sigma_maj = majorant_iterator.current_sigma_maj()
        t = sample_exponential(sigma_maj)
        t_geom = geometry.distance_to_intersection(p, omega)
        if t > t_geom:
            ps = p + t_geom * omega
            return T_maj * geometry.eval_radiance(ps, omega, -omega)
        p = p + t * omega
        T_maj *= exp(-sigma_maj * t)
        sigma_a = majorant_iterator.sigma_a(p)
        sigma_s = majorant_iterator.sigma_s(p)
        sigma_n = sigma_maj - sigma_a - sigma_s
        r = random()
        if r < sigma_a / sigma_maj:
            return T_maj * majorant_iterator.emission(p, omega)
        elif r < (sigma_a + sigma_s) / sigma_maj:
            omega_prime = sample_phase_function(p, omega)
            L_rec = null_scattering_path_trace(p, omega_prime, majorant_iterator, geometry)
            return T_maj * (sigma_s / sigma_maj) * (phase_function(p, omega_prime, omega) / pdf_phase(omega_prime)) * L_rec
        else:
            L_rec = null_scattering_path_trace(p, omega, majorant_iterator, geometry)
            return T_maj * (sigma_n / sigma_maj) * L_rec
```
*Note: The sources do not provide exact code for advancing through piecewise-constant majorant segments; implement a loop that clamps $t$ to segment boundaries and updates $T_{maj}$ with partial exponentials [G].*

**Formulas:**
*   Majorant Transmittance: $T_{maj}(p' \rightarrow p_0) = e^{-\sigma_{maj} d}$ [S1].
*   Null-Scattering Source Function:
    $$L_n(p_0, \omega_0) = \frac{\sigma_a(p_0)}{\sigma_{maj}} L_e(p_0, \omega_0) + \frac{\sigma_s(p_0)}{\sigma_{maj}} \int_S p(p_0, \omega_i, \omega_0) L_i(p_0, \omega_i) d\omega_i + \frac{\sigma_n(p_0)}{\sigma_{maj}} L_i(p_0, \omega_0)$$
    [S1]
*   PDF for Distance Sampling: $p(t) = \sigma_{maj} e^{-\sigma_{maj} t}$ [S1].
*   Recursive Estimator: $L_i(p_0, \omega_0) \approx L_n(p', -\omega_0)$ [S1].
*   Accumulated Transmittance: $T_{maj}(p_0 \rightarrow p') = T_{maj}^1(p_0 \rightarrow p_1) T_{maj}^2(p_1 \rightarrow p_2) \dots T_{maj}^n(p_{n-1} \rightarrow p')$ [S1].

**Pitfalls:**
*   The majorant $\sigma_{maj}$ is often piecewise-constant along a ray [S1]. The transmittance $T_{maj}$ is continuous but has discontinuous first derivatives at segment boundaries [S1]. Sampling must account for these segments by updating $\sigma_{maj}$ and $T_{maj}$ correctly at each segment transition [S1].
*   If $\sigma_{maj}$ is not constant, the simple exponential sampling $p(t) = \sigma_{maj} e^{-\sigma_{maj} t}$ is invalid across segment boundaries [S1]. The algorithm must sample within the current segment and handle transitions explicitly [S1].

## Subtopics

### [Null-Scattering Extension](4371-null-scattering-extension.md)
Generalizes the equation of transfer to enable unbiased sampling from a modified transmittance equation using a constant majorant extinction coefficient. Solves efficient sampling in sparse or heterogeneous media by homogenizing density. Differs from standard delta tracking by maintaining $T_{maj}$ and applying Russian Roulette based on null density ratio. Choose when media density varies significantly along rays and standard tracking becomes inefficient.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Not given in the sources.
*   **Limitations:** Not given in the sources.
*   **When to Use:** Not given in the sources.
*   **When Not to Use:** Not given in the sources.

## Background
The method originated in the context of volume rendering and light transport in participating media [S1]. It is described in "Light Transport II: Volume Rendering / The Equation of Transfer" [S1]. It is an application of the Null-Scattering Path Integral Framework to path tracing [S1].

## Sources
[S1] Light Transport II: Volume Rendering / The Equation of Transfer, https://pbr-book.org/4ed/Light_Transport_II_Volume_Rendering/The_Equation_of_Transfer.html, fetched 2026-10-27
