# In-scattering Model

Also called: In-Scattering Integration, Particle In-Scattering

## Overview
In-scattering is the phenomenon where light is scattered from other directions into the viewing direction as it travels through a participating medium, representing "light gained" [S1, unofficial]. It solves the problem of calculating the radiance contribution from the environment and light sources that is redirected by the medium's particles toward the camera, contrasting with out-scattering (light lost) [S1, unofficial]. Unlike its siblings which define the angular distribution of scattering (the Phase Function), the In-scattering Model defines the integral accumulation of this scattered light along a ray, incorporating attenuation [S2, unofficial].

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
def compute_in_scattering(ray_origin, ray_direction, medium, light_field):
    """
    Computes in-scattering radiance at ray_origin.
    medium: object with sigma_t(t), sigma_s(t), phase_function(p, wi, wo)
    light_field: function L(p, wo) returning incident radiance at p from wo
    """
    # Step 1: Define the source term S(p)
    # S(p) = q(p) + j(p)
    # j(p) = integral over sphere of sigma_s(p, wi) * phase(p, wi, wo) * L(p, wi) dwi
    # Note: In practice, this integral is approximated (e.g., via spherical harmonics or Monte Carlo)
    def source_term(p, wo):
        emission = medium.get_emission(p) # q(p)
        scattering_integral = 0.0
        # Approximate integral over S^2
        # ... implementation depends on light field representation ...
        scattering_integral = compute_scattering_integral(p, wo, medium, light_field)
        return emission + scattering_integral

    # Step 2: Integrate source term attenuated by transmittance from 0 to infinity
    # L(p(0)) = integral_0^inf T_r(p(t) -> p(0)) * S(p(t)) dt
    # T_r(p(t) -> p(0)) = exp(- integral_0^t sigma_t(p(tau)) dtau)
    
    total_radiance = 0.0
    t = 0.0
    dt = medium.step_size # Or adaptive step
    
    # Accumulate transmittance from origin to current point t
    # This is the "light lost" factor for light arriving from t to 0
    accumulated_extinction = 0.0
    
    while t < infinity_limit:
        p_t = ray_origin + ray_direction * t
        
        # Calculate transmittance from p(t) to p(0)
        # T_r(p(t) -> p(0)) = exp(- integral_0^t sigma_t dtau)
        transmittance_to_origin = exp(-accumulated_extinction)
        
        # Get source term at p_t for direction -ray_direction (incoming to eye)
        # Note: S(p) depends on incident radiance from all directions, 
        # but the specific contribution to the eye is weighted by the phase function 
        # inside the j(p) calculation if not pre-integrated.
        # However, the formula L(p(0)) = int T_r * S(p) dt assumes S(p) is the 
        # total source term. If S(p) is defined as total emission + in-scattering 
        # into ALL directions, we must be careful. 
        # Standard derivation: S(p, -omega) is the source term for direction -omega.
        
        s_p = source_term(p_t, -ray_direction)
        
        # Add contribution
        total_radiance += transmittance_to_origin * s_p * dt
        
        # Update accumulated extinction for next step
        sigma_t_t = medium.get_sigma_t(p_t)
        accumulated_extinction += sigma_t_t * dt
        
        t += dt
        
    return total_radiance
```

**Parameters**
*   $\sigma_s$: Scattering coefficient (probability density of scattering event) [S2, unofficial].
*   $\sigma_t$: Extinction coefficient ($\sigma_a + \sigma_s$) [S2, unofficial].
*   $f_p$: Phase function, directional distribution of out-scattering events [S3, unofficial].

**Edge Cases**
*   **Infinity Limit:** The integral is from $0$ to $\infty$. In practice, this is truncated when transmittance becomes negligible [S4, unofficial].
*   **Singularity:** Phase functions like Henyey-Greenstein can have singularities at $\cos \theta = 1$ (forward scattering) or $-1$ (backward scattering), requiring careful numerical integration [S1, unofficial].

## Subtopics

### [Out-Scattering Model](6803-out-scattering-model.md)
Models the loss of radiance from the viewing direction due to scattering away from the ray. It is the complementary term to in-scattering in the radiative transfer equation, representing "light lost".

## Operating characteristics
*   **Complexity:** The in-scattering integral is a 4D integral (2D for space along ray, 2D for incident direction sphere). It is computationally expensive, often requiring approximation techniques like spherical harmonics or precomputed radiance transfer [S2, unofficial].
*   **Performance:** Real-time solutions often use biased approximations or particle systems (billboards) because unbiased integration is too slow [S4, unofficial].
*   **Strengths:** Physically accurate representation of volumetric lighting effects (e.g., god rays, fog illumination) [S1, unofficial].
*   **Limitations:** High computational cost; sensitive to phase function choice and light field resolution [S2, unofficial].

## Background
*   **Origin:** The concept is rooted in the Radiative Transfer Equation (RTE) used in atmospheric radiation and physics [S2, unofficial].
*   **History:** The derivation using the integrating factor method is standard in physics-based rendering literature, such as PBRT [S4, unofficial].
*   **Usage:** Used in offline renderers for realistic volumetric effects and in real-time engines for approximations like fog and smoke [S4, unofficial].

## Sources
[S1] Volumetric rendering fundamental concepts and terminology, https://computergraphics.stackexchange.com/questions/145/volumetric-rendering-fundamental-concepts-and-terminology, fetched 2026-10-27 (unofficial)
[S2] [Graphics] Volume Rendering - GitHub Pages, https://leeyngdo.github.io/blog/computer-graphics/2024-03-28-volume-rendering/, fetched 2026-10-27 (unofficial)
[S3] Volume Rendering Volume Rendering, https://docslib.org/doc/1400821/volume-rendering, fetched 2026-10-28 (unofficial)
[S4] Volume Rendering in Offline Renderer · A Graphics Guy's Note, https://agraphicsguynotes.com/posts/volume_rendering_in_offline_renderer/, fetched 2026-10-28 (unofficial)
