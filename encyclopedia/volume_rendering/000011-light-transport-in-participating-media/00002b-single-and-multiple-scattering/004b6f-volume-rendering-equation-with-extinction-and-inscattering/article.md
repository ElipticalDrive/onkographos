# Volume Rendering Equation with Extinction and Inscattering

Also called: Beer-Lambert with Inscattering, Physical Volume Model

## Overview
This method computes the radiance arriving at a camera by integrating the Radiative Transfer Equation (RTE), accounting for light attenuation via extinction (absorption and out-scattering) and light gain via emission and in-scattering [S1, unofficial]. It differs from the [Absorption-only model](839-absorption-only-model.md) by including scattering effects and from the [Single Scattering Model](1053-single-scattering-model.md) by integrating over the full sphere of directions for in-scattering, providing a physically accurate solution for participating media [S1, unofficial].

## How it works
**Inputs:**
*   Camera ray origin $\mathbf{o}$ and direction $\boldsymbol{\omega}$ (unit vector pointing from eye to scene).
*   Volume density field $\rho(\mathbf{p})$ (scalar field, typically $0$ to $1$).
*   Absorption coefficient $\sigma_a$ (base value, units $m^{-1}$).
*   Scattering coefficient $\sigma_s$ (base value, units $m^{-1}$).
*   Emissive term $L_e(\mathbf{p}, \boldsymbol{\omega})$ (radiance emitted per unit length).
*   Incident radiance $L_i(\mathbf{p}, \boldsymbol{\omega}')$ from light sources, attenuated through the volume.
*   Phase function $p(\boldsymbol{\omega}, \boldsymbol{\omega}')$.

**Outputs:**
*   Final radiance $L_o$ at the camera.

**Data Structures:**
*   **Extinction Coefficient $\sigma_t$**: A scalar value computed per sample.
    *   Layout: $\sigma_t = \sigma_a + \sigma_s$ [S1, unofficial].
*   **Phase Function $p$**: A scalar value in range $[0, 1]$ [S1, unofficial].
    *   Depends on the angle $\theta$ between view direction $\boldsymbol{\omega}$ and light direction $\boldsymbol{\omega}'$.
    *   For isotropic scattering: $p = \frac{1}{4\pi}$ [S1, unofficial].
    *   Normalization: $\int_{\Omega} p(\boldsymbol{\omega}, \boldsymbol{\omega}') d\omega' = 1$ to conserve energy [S1, unofficial].

**Parameters:**
*   $\sigma_a$: Absorption coefficient. Represents light absorbed by the volume [S1, unofficial].
*   $\sigma_s$: Scattering coefficient. Represents probability of light scattering outwards [S1, unofficial].
*   $\sigma_t$: Extinction coefficient. Sum of $\sigma_a$ and $\sigma_s$, used in Beer's Law [S1, unofficial].
*   Density: A global scaling variable for $\sigma_a$ and $\sigma_s$ to adjust volume opacity uniformly [S1, unofficial].

**Steps:**
1.  **Compute Extinction**: Calculate the total extinction coefficient $\sigma_t$ as the sum of absorption and scattering coefficients:
    $$ \sigma_t = \sigma_a + \sigma_s $$
    Apply global density scaling if applicable [S1, unofficial].
2.  **Compute Transmittance**: Use $\sigma_t$ in the Beer-Lambert law to compute transmission for the camera ray and the ray to the light source:
    $$ T = e^{-\tau} = e^{-\int \sigma_t dt} $$
    This accounts for both absorption and out-scattering (light lost from the view ray) [S1, unofficial].
3.  **Check Density**: If the local density is zero, skip accumulation for this sample. No light is accumulated, and no in-scattering is added [S1, unofficial].
4.  **Compute In-Scattering**: Calculate the in-scattering contribution $L_i(x, \boldsymbol{\omega})$ by integrating incident radiance over the sphere of directions:
    $$ L_i(x, \boldsymbol{\omega}) = \int_{\Omega} p(\boldsymbol{\omega}, \boldsymbol{\omega}') L(x, \boldsymbol{\omega}') d\omega' $$
    Where:
    *   $x$ is the sample position.
    *   $\boldsymbol{\omega}$ is the view direction.
    *   $\boldsymbol{\omega}'$ is the light direction (opposite to radiance flow, from light to particle) [S1, unofficial].
    *   $L(x, \boldsymbol{\omega}')$ is the incident radiance at $x$ from direction $\boldsymbol{\omega}'$, attenuated through the volume [S1, unofficial].
    *   $p(\boldsymbol{\omega}, \boldsymbol{\omega}')$ is the phase function, determined by the angle $\theta$ between $\boldsymbol{\omega}$ and $\boldsymbol{\omega}'$ (via dot product and cosine) [S1, unofficial].
5.  **Accumulate**: Add the in-scattering term and emission term to the result, weighted by the transmittance from the camera to the sample point.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_volume_ray(ray_origin, ray_direction, volume_density, sigma_a_base, sigma_s_base, L_e, L_incident, phase_function):
    """
    Renders a single ray through a participating medium using the volume rendering equation with extinction and inscattering.
    
    Args:
        ray_origin: vec3, origin of the camera ray.
        ray_direction: vec3, direction of the camera ray (unit vector).
        volume_density: function(p) -> float, returns density at position p.
        sigma_a_base: float, base absorption coefficient.
        sigma_s_base: float, base scattering coefficient.
        L_e: function(p, omega) -> vec3, emissive term at position p in direction omega.
        L_incident: function(p, omega) -> vec3, incident radiance from lights, attenuated.
        phase_function: function(omega, omega_prime) -> float, returns scattering probability.
        
    Returns:
        vec3, final radiance.
    """
    # Constants
    MAX_STEPS = 100
    STEP_SIZE = 0.1 # Example step size, depends on resolution
    
    accumulated_radiance = vec3(0.0)
    transmittance = 1.0
    
    # Ray marching
    for i in range(MAX_STEPS):
        # Current sample position
        pos = ray_origin + ray_direction * (i * STEP_SIZE)
        
        # Get density at current position
        density = volume_density(pos)
        
        # Edge case: If density is zero, no interaction occurs
        if density == 0.0:
            continue
            
        # Compute extinction coefficient
        # Density scales the base coefficients globally
        sigma_a = sigma_a_base * density
        sigma_s = sigma_s_base * density
        sigma_t = sigma_a + sigma_s
        
        # Compute transmittance for this step
        # T = exp(-sigma_t * dt)
        step_transmittance = exp(-sigma_t * STEP_SIZE)
        
        # Compute in-scattering
        # L_i(x, omega) = integral over sphere of p(omega, omega') * L(x, omega') domega'
        # In practice, this is approximated by sampling light directions
        inscattering = vec3(0.0)
        
        # Example: Single light source approximation for inscattering
        # Note: Full integration requires sampling the sphere, which is computationally expensive.
        # Here we assume L_incident provides the attenuated light from specific directions.
        # For a single light, we might sample the direction to the light.
        light_direction = normalize(light_position - pos)
        
        # Phase function value
        # theta is angle between view direction and light direction
        # p depends on theta
        p_val = phase_function(ray_direction, light_direction)
        
        # Incident radiance from light, already attenuated through volume
        # L(x, omega') is the light reaching this point from the light source
        L_light = L_incident(pos, light_direction)
        
        # In-scattering contribution for this direction
        # Note: The integral is over all directions. This is a simplification.
        inscattering += p_val * L_light
        
        # Add emission
        emission = L_e(pos, ray_direction)
        
        # Accumulate radiance
        # dL = (sigma_s * L_i + sigma_e) * T * dt
        # Here, sigma_s * L_i is the inscattering term, sigma_e is emission
        # We multiply by the transmittance from the camera to this point (transmittance)
        # And the transmittance of the current step (step_transmittance)
        
        # The term (sigma_s * inscattering + emission) represents the source term
        # multiplied by the step length (implicitly handled by integration)
        
        # Correct accumulation formula:
        # L_o = integral from 0 to t_max of T(0, t) * (sigma_s * L_i(t) + sigma_e(t)) dt
        # Discretized:
        # accumulated_radiance += transmittance * (sigma_s * inscattering + emission) * STEP_SIZE
        # transmittance *= step_transmittance
        
        # However, the source notes emphasize that sigma_s is used for transmission calculation too.
        # And that result is premultiplied by opacity.
        
        # Let's follow the standard volume rendering integral:
        # C = integral T(t) * sigma_s * L_i(t) dt + integral T(t) * sigma_e(t) dt
        
        accumulated_radiance += transmittance * (sigma_s * inscattering + emission) * STEP_SIZE
        transmittance *= step_transmittance
        
        # Early termination if transmittance is negligible
        if transmittance < 0.001:
            break
            
    return accumulated_radiance
```

**Edge Cases and Pitfalls:**
*   **Zero Density**: If density is zero, no light is accumulated, and no in-scattering is added [S1, unofficial].
*   **Phase Function Normalization**: The phase function must be normalized such that its integral over the sphere equals 1 to conserve energy [S1, unofficial].
*   **Anisotropic Scattering**: Most volumes are anisotropic, scattering light biased towards certain directions (e.g., forward scattering). Isotropic scattering ($p = 1/4\pi$) is a simplification [S1, unofficial].
*   **Computational Cost**: Integrating over the sphere of directions for in-scattering is expensive. Approximations (like single scattering or gradient-guided approximations) are often used [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(N \cdot M)$, where $N$ is the number of samples per ray and $M$ is the number of light directions sampled for in-scattering integration. Full integration over the sphere is $O(N \cdot S)$, where $S$ is the number of samples over the sphere.
*   **Memory Complexity**: $O(V)$, where $V$ is the size of the volume density field.
*   **Strengths**: Physically accurate; handles complex lighting interactions; accounts for both absorption and scattering.
*   **Limitations**: Computationally expensive due to the integration over the sphere of directions for in-scattering; sensitive to noise if not enough samples are used.
*   **When to Use**: High-quality offline rendering; scenes with significant scattering effects (fog, smoke, clouds).
*   **When Not to Use**: Real-time applications with limited computational resources; scenes where scattering effects are negligible.

## Background
The volume rendering equation with extinction and inscattering is derived from the Radiative Transfer Equation (RTE), a fundamental equation in physics describing light transport in participating media [S1, unofficial]. It generalizes the Beer-Lambert law by including scattering and emission terms. The method is widely used in computer graphics for rendering volumetric effects like fog, smoke, and clouds [S1, unofficial]. The phase function concept, introduced to model directional scattering, is central to this model [S1, unofficial].

## Sources
[S1] "Volume Rendering for Developers" Group Read Part 1 [2023-03-30], https://www.youtube.com/watch?v=mZzMVD4LQvk, fetched 2026-10-26 (unofficial)
