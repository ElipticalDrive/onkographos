# In-scattering Model

Also called: In-Scattering Integration, Particle In-Scattering

## Overview
In-scattering is the phenomenon where light is scattered from other directions into the viewing direction as it travels through a participating medium, representing "light gained" [S1, unofficial]. It solves the problem of calculating the radiance contribution from the environment and light sources that is redirected by the medium's particles toward the camera, contrasting with out-scattering (light lost) [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** Ray origin $p(0)$, ray direction $\omega$, medium properties (scattering coefficient $\sigma_s$, phase function $f_p$), incident radiance field $L_i(p, \omega')$, and the in-scattering source term $S(p)$ [S2, unofficial][S4, unofficial].
*   **Output:** Radiance $L(p(0))$ at the ray origin due to in-scattering [S4, unofficial].

**Data Structures**
*   **Phase Function $f_p$:** A function mapping $(\mathbf{x}, \omega, \omega') \to [0, 1]$ (normalized). It acts as a probability density function for scattering direction [S2, unofficial].
*   **Source Term $S(p)$:** Represents the total emission at point $p$, defined as true emission $q(p)$ plus in-scattering $j(p)$ [S3, unofficial].
    *   $j(p, \omega) = \int_{S^2} \sigma_s(p, \omega_i) f_p(p, \omega_i, \omega) L(p, \omega_i) d\omega_i$ [S3, unofficial].
    *   $S(p, \omega) = q(p, \omega) + j(p, \omega)$ [S3, unofficial].

**Formulas**
The change in radiance along a ray is governed by the linear differential equation [S4, unofficial]:
$$ \frac{dL(p)}{dt} = -\sigma_t(p) L(p) + S(p) $$
where $\sigma_t = \sigma_a + \sigma_s$ is the extinction coefficient [S2, unofficial].

The normalization condition for the phase function is [S2, unofficial]:
$$ \forall \omega \in \Omega: \; \int_{S^2} f_p (\mathbf{x}, \omega, \omega^\prime) \; d\omega^\prime = 1 $$

The differential change in radiance due to in-scattering is [S2, unofficial]:
$$ \mathrm{d}L_\mathrm{i} (\mathrm{p}, \omega) = \sigma_{\mathrm{s}} (\mathrm{p}, \omega) \int_{S^2} f_p (\mathrm{p}, \omega, \omega^\prime) L_{\mathrm{i}} (\mathrm{p}, \omega^\prime) \; \mathrm{d} \omega^\prime $$

**Steps**
1.  **Invert Ray Direction:** To solve for incoming radiance, invert the ray direction from $\omega$ to $-\omega$. This changes the differential denominator from $dt$ to $d(-t)$ [S4, unofficial].
2.  **Formulate Differential Equation:** The equation becomes a first-order linear differential equation:
    $$ \frac{dL(p, -\omega)}{d(-t)} = -\sigma_t(p, -\omega) L(p, -\omega) + S(p, -\omega) $$
    Rearranging for standard form:
    $$ \frac{dL(p, -\omega)}{d(t)} = \sigma_t(p, -\omega) L(p, -\omega) - S(p, -\omega) $$
    [S4, unofficial].
3.  **Apply Integrating Factor:** Solve using the integrating factor method. The general solution is [S4, unofficial]:
    $$ L(p(t)) = \frac{1}{e^{-\int \sigma (p)dt}} \left( -\int e^{- \int \sigma (p)dt} S(p) dt + C \right) $$
4.  **Determine Constant $C$:** Assume $L(p(\infty)) = 0$ (radiance at infinity is zero). This allows solving for $C$ [S4, unofficial].
5.  **Compute Final Radiance:** The solution for radiance at any point $t$ is [S4, unofficial]:
    $$ L(p(t)) = \int_t^{\infty} e^{- \int_{t}^{t'} \sigma (p)dt''} S(p) dt' $$
    For the ray origin ($t=0$), the radiance is [S4, unofficial]:
    $$ L(p(0)) = \int_0^{\infty} T_r(p(t) \rightarrow p(0)) S(p(t)) dt $$
    where $T_r$ is the transmittance [S4, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_in_scattering(ray_origin, ray_direction, medium, light_field, max_distance=100.0, epsilon=1e-6):
    """
    Computes in-scattering radiance at ray_origin.
    medium: object with sigma_t(t), sigma_s(t), phase_function(p, wi, wo), get_emission(p)
    light_field: function L(p, wo) returning incident radiance at p from wo
    """
    def source_term(p, wo):
        emission = medium.get_emission(p)
        # Spherical integral for in-scattering j(p, wo)
        # j(p, wo) = integral over S^2 of sigma_s(p, wi) * phase(p, wi, wo) * L(p, wi) dwi
        # Standard uniform sphere sampling approximation [G]
        scattering_integral = 0.0
        num_samples = 16
        for i in range(num_samples):
            wi = sample_sphere_direction(i, num_samples)
            L_incident = light_field(p, wi)
            scattering_integral += medium.sigma_s(p) * medium.phase_function(p, wi, wo) * L_incident
        scattering_integral /= num_samples
        return emission + scattering_integral

    total_radiance = 0.0
    t = 0.0
    dt = medium.step_size
    accumulated_extinction = 0.0

    while t < max_distance:
        p_t = ray_origin + ray_direction * t
        transmittance_to_origin = exp(-accumulated_extinction)

        if transmittance_to_origin < epsilon:
            break

        s_p = source_term(p_t, -ray_direction)
        total_radiance += transmittance_to_origin * s_p * dt

        sigma_t_t = medium.get_sigma_t(p_t)
        accumulated_extinction += sigma_t_t * dt
        t += dt

    return total_radiance

def sample_sphere_direction(i, n):
    # Standard uniform sphere sampling [G]
    phi = 2.0 * pi * i / n
    theta = acos(1.0 - 2.0 * (i + 0.5) / n)
    return vec3(sin(theta)*cos(phi), sin(theta)*sin(phi), cos(theta))
```

**Parameters**
*   $\sigma_s$: Scattering coefficient (probability density of scattering event) [S2, unofficial].
*   $\sigma_t$: Extinction coefficient ($\sigma_a + \sigma_s$) [S2, unofficial].
*   $f_p$: Phase function, directional distribution of out-scattering events [S3, unofficial].

**Edge Cases**
*   **Infinity Limit:** The integral is from $0$ to $\infty$ [S4, unofficial].
*   **Singularity:** Phase functions like Henyey-Greenstein can have singularities at $\cos \theta = 1$ (forward scattering) or $-1$ (backward scattering), requiring careful numerical integration [S1, unofficial].

## Subtopics

### [Out-Scattering Model](6803-out-scattering-model.md)
Models the loss of radiance from the viewing direction due to scattering away from the ray. It is the complementary term to in-scattering in the radiative transfer equation, representing "light lost".

## Operating characteristics
*   **Complexity:** The in-scattering integral is a 4D integral (2D for space along ray, 2D for incident direction sphere) [S2, unofficial].
*   **Performance:** Real-time solutions often use biased approximations or particle systems (billboards) because unbiased integration is too slow [S4, unofficial].
*   **Strengths:** Physically accurate representation of volumetric lighting effects (e.g., god rays, fog illumination) [S1, unofficial].
*   **Limitations:** Requires accurate phase function modeling and incident radiance field evaluation [S2, unofficial].

## Background
*   **Origin:** The concept is rooted in the Radiative Transfer Equation (RTE) used in atmospheric radiation and physics [S2, unofficial].
*   **History:** The derivation using the integrating factor method is standard in physics-based rendering literature, such as PBRT [S4, unofficial].
*   **Usage:** Used in offline renderers for realistic volumetric effects and in real-time engines for approximations like fog and smoke [S4, unofficial].

## Sources
[S1] Volumetric rendering fundamental concepts and terminology, https://computergraphics.stackexchange.com/questions/145/volumetric-rendering-fundamental-concepts-and-terminology, fetched 2026-10-27 (unofficial)
[S2] [Graphics] Volume Rendering - GitHub Pages, https://leeyngdo.github.io/blog/computer-graphics/2024-03-28-volume-rendering/, fetched 2026-10-27 (unofficial)
[S3] Volume Rendering Volume Rendering, https://docslib.org/doc/1400821/volume-rendering, fetched 2026-10-28 (unofficial)
[S4] Volume Rendering in Offline Renderer · A Graphics Guy's Note, https://agraphicsguynotes.com/posts/volume_rendering_in_offline_renderer/, fetched 2026-10-28 (unofficial)
