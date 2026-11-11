# Volumetric Scattering Model

Also called: Participating Media Physics, Participating Medium Rendering

## Overview
Volumetric scattering models the interaction of light with participating media—substances like fog, smoke, clouds, and translucent solids (e.g., wax, skin) where the space itself participates in lighting [S1, unofficial][S2]. Unlike surface rendering, where radiance is constant along a ray between surfaces, participating media modify radiance through emission, absorption, out-scattering, and in-scattering [S1, unofficial]. This method distinguishes itself by explicitly modeling the transport of light through volume using the radiative transfer equation, decomposed into extinction (absorption + out-scattering) and in-scattering terms, rather than approximating volume effects via surface shaders or simple opacity masks [S1, unofficial][S2].

## How it works
### Inputs and Outputs
**Inputs:**
*   Ray origin $x$ and direction $\omega$.
*   Volume definition (homogeneous or heterogeneous) providing spatially varying coefficients.
*   Light sources (for direct in-scattering).
*   Environment map or indirect lighting cache (for indirect in-scattering).
*   Phase function parameters (e.g., anisotropy $g$, wavelength $\lambda$).

**Outputs:**
*   Radiance $L(x, \omega)$ accumulated along the ray segment.

### Data Structures
The implementation relies on two primary abstractions, as defined in the Volume Shader parent article [10695-volume-shader.md]:

1.  **Volume Class**:
    *   `evaluate(x)`: Returns a struct containing:
        *   $\sigma_a(x)$: Absorption coefficient ($1/m$).
        *   $\sigma_s(x)$: Scattering coefficient ($1/m$).
        *   $E(x)$: Emission radiance ($W/m^2/sr$).
        *   Phase function object or parameters.
    *   `intersect(ray)`: Returns entry and exit distances or a bounding volume test.

2.  **Phase Function Class**:
    *   `evaluate(cos_theta)`: Returns probability density value.
    *   `sample(cos_theta)`: Returns a sampled direction and PDF.

### Formulas and Definitions
The change in radiance $dL$ along a differential path length $ds$ is governed by four processes [S1, unofficial]:

1.  **Emission**:
    $$ dL(x, \omega) = \frac{E(x) ds}{4\pi} $$
    Where $E(x)$ is isotropic emission per unit length [S1, unofficial].

2.  **Absorption**:
    $$ dL(x, \omega) = -\sigma_a(x) L(x, \omega) ds $$
    Where $\sigma_a$ is the absorption coefficient [S1, unofficial].

3.  **Out-Scattering**:
    $$ dL(x, \omega) = -\sigma_s(x) L(x, \omega) ds $$
    Where $\sigma_s$ is the scattering coefficient [S1, unofficial].

4.  **Extinction**:
    Combining absorption and out-scattering:
    $$ \sigma_t(x) = \sigma_a(x) + \sigma_s(x) $$
    $$ dL(x, \omega) = -\sigma_t(x) L(x, \omega) ds $$

5.  **Transmittance**:
    The fraction of light surviving distance $s$:
    $$ T(s) = e^{-\tau(s)} = e^{-\int_0^s \sigma_t(x + s'\omega) ds'} $$
    For homogeneous media, $\tau(s) = \sigma_t s$ [S1, unofficial].

6.  **In-Scattering**:
    Light scattered *into* the ray direction from other directions $\omega_i$:
    $$ dL(x, \omega) = \sigma_s(x) ds \int_{4\pi} p(\omega_i \cdot \omega) L(x, \omega_i) d\omega_i $$
    Where $p(\cos \theta)$ is the phase function [S1, unofficial].

### Phase Functions
The phase function $p(\cos \theta)$ describes the angular distribution of scattered light. It must be reciprocal and energy-conserving ($\int_{4\pi} p(\omega_i \cdot \omega) d\omega_i = 1$) [S1, unofficial].

*   **Isotropic**:
    $$ p(\cos \theta) = \frac{1}{4\pi} $$
    Scatters equally in all directions [S1, unofficial].

*   **Henyey-Greenstein**:
    $$ p(\cos \theta) = \frac{1 - g^2}{4\pi (1 + g^2 - 2g\cos \theta)^{1.5}} $$
    Uses anisotropy parameter $g \in [-1, 1]$. $g=0$ is isotropic; $g>0$ is forward-scattering; $g<0$ is back-scattering [S1, unofficial].

*   **Rayleigh** (for particles $d \ll \lambda$):
    $$ p(\cos \theta) = \frac{3}{16\pi} (1 + \cos^2 \theta) $$
    Scattering coefficient: $\sigma_s = \frac{2\pi^5 d^6}{3 \lambda^4} \frac{(n^2 - 1)^2}{(n^2 + 2)^2}$ [S1, unofficial].

*   **Empirical Mie Approximation**:
    $$ p(\cos \theta) = \frac{1}{4\pi} \frac{z+1}{2} \left( \frac{1 + \cos \theta}{2} \right)^z $$
    Controls forward scattering sharpness via parameter $z$ (e.g., $z=8, 32$) [S1, unofficial].

### Algorithm: Ray Marching
The integral form of the radiative transfer equation is solved numerically via ray marching [S1, unofficial].

1.  **Discretization**: Divide the ray segment inside the volume into $N$ steps.
2.  **Step Size**: Use a maximum step size $\Delta s_{max}$. The actual step $ds$ is a random value in $[0, \Delta s_{max}]$ to reduce banding artifacts [S1, unofficial].
3.  **Accumulation**: For each segment from $x$ to $x + s\omega$:
    *   Compute extinction (transmittance) from previous point.
    *   Add emission contribution.
    *   Add in-scattering contribution.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_march_volume(ray_origin, ray_direction, volume, light_sources, env_map):
    # 1. Find entry and exit points
    t_entry, t_exit = volume.intersect(ray_origin, ray_direction)
    if t_entry is None:
        return 0.0

    current_pos = ray_origin + t_entry * ray_direction
    accumulated_radiance = 0.0
    transmittance = 1.0
    
    # 2. Ray march through the volume
    # Max step size is a tunable parameter, e.g., 0.1 units
    max_step = 0.1 
    
    while current_pos < t_exit:
        # Random step size to avoid banding
        step_size = random.uniform(0, max_step)
        
        # Clamp step to not exceed volume exit
        remaining = t_exit - (current_pos - ray_origin).dot(ray_direction)
        if step_size > remaining:
            step_size = remaining
            
        next_pos = current_pos + step_size * ray_direction
        
        # 3. Evaluate volume properties at current position
        vol_props = volume.evaluate(current_pos)
        sigma_t = vol_props.sigma_a + vol_props.sigma_s
        sigma_s = vol_props.sigma_s
        emission = vol_props.emission
        
        # 4. Compute Transmittance for this segment
        # Optical thickness tau = sigma_t * step_size (assuming constant over segment)
        tau = sigma_t * step_size
        segment_transmittance = exp(-tau)
        
        # 5. Compute In-Scattering Li(x)
        # Approximate the integral over the sphere by sampling
        # Direct lighting component
        direct_light = 0.0
        for light in light_sources:
            # Trace shadow ray to light
            shadow_t = volume.intersect(current_pos, light.direction)
            if shadow_t is not None and shadow_t < light.distance:
                # Attenuate light by extinction through volume to light
                # For homogeneous: exp(-sigma_t * shadow_t)
                # For heterogeneous: ray march from current_pos to light
                light_transmittance = compute_volume_transmittance(current_pos, light.direction, shadow_t, volume)
                direct_light += light.intensity * light_transmittance
            
        # Indirect lighting component (simplified as environment map sample)
        # In a full implementation, this requires tracing rays into the environment
        indirect_light = env_map.sample(current_pos, ray_direction) # Placeholder for complex indirect
        
        # Total incoming radiance Li(x)
        # Note: The source approximates Li(x) as the sum of direct and indirect contributions
        # scaled by the phase function.
        # Li(x) approx = sum( p(wi, w) * L(wi) )
        
        # Simplified In-Scattering contribution for this segment
        # Li(x) is the incoming radiance at x. 
        # The term in the RTE is sigma_s * ds * Li(x)
        # We approximate Li(x) by sampling directions.
        
        # Direct In-Scattering
        phase_val = vol_props.phase_function.evaluate(ray_direction, light.direction)
        # Note: Source formula: Li(x) approx (4*pi * sigma_s / N) * sum( p * L )
        # Here we compute the contribution directly:
        # dL_in = sigma_s * ds * p * L_direct
        in_scattering_direct = sigma_s * step_size * phase_val * direct_light
        
        # Indirect In-Scattering (simplified)
        # In practice, this requires Monte Carlo integration over the hemisphere/sphere
        # For this basic model, we assume a single indirect sample or cache
        phase_val_indirect = vol_props.phase_function.evaluate(ray_direction, env_direction)
        in_scattering_indirect = sigma_s * step_size * phase_val_indirect * indirect_light
        
        total_in_scattering = in_scattering_direct + in_scattering_indirect
        
        # 6. Update Accumulated Radiance
        # L_out = T * L_in + E * (1 - T) / sigma_t  <-- Analytic solution for constant sigma_t
        # Or the discrete approximation from source:
        # L(x+s) = e^(-sigma_t * s) * L(x) + E(x)*s + Li(x)*s
        
        # Emission contribution: E(x) * ds (isotropic, so divided by 4pi in definition, 
        # but here E(x) is usually defined as radiance emitted per length)
        # Source: dL = E(x) ds / 4pi. 
        # If E(x) is total emission power, we divide by 4pi.
        emission_contrib = (emission / 4.0 * PI) * step_size
        
        # Add to accumulated radiance, weighted by transmittance of subsequent segments?
        # No, we accumulate radiance *along* the ray.
        # The radiance arriving at the camera is the sum of emitted/scattered light 
        # attenuated by the rest of the path.
        
        # Correct accumulation:
        # The radiance leaving the segment is:
        # L_next = T_segment * L_current + emission_contrib + in_scattering_contrib
        # But we want the radiance at the end of the ray.
        # We accumulate the contribution of each segment to the final pixel.
        
        # Contribution of this segment to the final pixel:
        # It is the light generated in this segment, attenuated by the rest of the volume.
        # However, the standard ray marching update is:
        # L(x + s) = T(s) * L(x) + SourceTerm
        # We can rewrite this as:
        # L_final = sum( Source_i * Transmittance_from_i_to_end )
        
        # Let's use the update rule from the source:
        # L(x + s) approx e^(-sigma_t * s) L(x) + E(x)s + Li(x)s
        # This computes the radiance at the next point.
        
        # To get the final pixel value, we track the radiance at the current point.
        # But we need the initial radiance L(x) entering the volume (from background or previous volume).
        
        # Let's assume L_current is the radiance entering this segment.
        # L_next = segment_transmittance * L_current + emission_contrib + total_in_scattering
        
        # However, standard accumulation sums the contributions.
        # Let's stick to the source's discrete update:
        
        L_current = segment_transmittance * L_current + emission_contrib + total_in_scattering
        
        current_pos = next_pos
        
    return L_current
```

*Note: The pseudocode above implements the discrete update rule $L(x+s\omega) \approx e^{-\sigma_t s} L(x) + E(x)s + L_i(x)s$ described in [S1, unofficial]. In practice, $L_i(x)$ requires Monte Carlo integration over the sphere of directions, often split into direct (shadow rays) and indirect (environment/brdf) components. The shadow ray calculation for volumes involves ray marching through the volume to compute attenuation [S1, unofficial].*

### Edge Cases and Pitfalls
*   **Homogeneous vs. Heterogeneous**: Homogeneous volumes allow closed-form transmittance ($e^{-\sigma_t s}$). Heterogeneous volumes require ray marching for every shadow ray and in-scattering calculation, significantly increasing cost [S1, unofficial].
*   **Single vs. Multiple Scattering**: Single scattering (one event) is sufficient for thin haze or highly absorbing smoke. Dense, low-absorption volumes (clouds) require multiple scattering, which is computationally expensive and often approximated or precomputed [S1, unofficial].
*   **Precision**: Small step sizes are needed for high-frequency heterogeneous volumes to avoid aliasing, but increase computational cost. Randomized step sizes help reduce banding [S1, unofficial].
*   **Phase Function Sampling**: Importance sampling the phase function is required for efficient Monte Carlo integration of in-scattering. Isotropic scattering is trivial; Henyey-Greenstein requires specific sampling algorithms [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(N \cdot M)$, where $N$ is the number of ray march steps per pixel and $M$ is the number of light/environment samples per step for in-scattering. Heterogeneous volumes increase $M$ due to per-step extinction checks for shadow rays [S1, unofficial].
*   **Memory Complexity**: Low, if volumes are evaluated on-the-fly (e.g., via noise functions). High if precomputed voxel grids are used.
*   **Strengths**: Physically accurate representation of light transport in participating media; handles complex lighting effects like god rays and subsurface scattering [S1, unofficial][S2].
*   **Limitations**: Computationally expensive due to the need for in-scattering integration; single scattering approximations fail for dense media [S1, unofficial].
*   **When to Use**: Clouds, smoke, fog, translucent materials.
*   **When Not to Use**: Real-time applications requiring high frame rates without approximations; very dense media where multiple scattering dominates and single scattering is insufficient [S1, unofficial].

## Background
The volumetric scattering model is rooted in the physics of radiative transfer in participating media. The term "participating medium" was popularized in computer graphics to describe media that interact with light throughout their volume [S1, unofficial][S2]. The mathematical framework relies on the Radiative Transfer Equation (RTE). Key developments include the use of the Henyey-Greenstein phase function for galactic dust [S1, unofficial], Rayleigh scattering for atmospheric effects [S1, unofficial], and Mie scattering for water droplets [S1, unofficial]. The ray marching algorithm is a standard numerical solution for the RTE in rendering [S1, unofficial].

## Sources
[S1] CSE 168: Rendering Algorithms, https://cseweb.ucsd.edu/classes/sp17/cse168-a/CSE168_14_Volumetric.pdf, fetched 2026-10-27 (unofficial)
[S2] Introduction to Volumetric Rendering, https://help.maxon.net/c4d/2027/en-us/Content/html/What_is_Volumetric_Rendering.html, fetched 2026-10-29
