# Single Scattering

Also called: Direct Lighting in Participating Media

## Overview
Single scattering is an approximation where in-scattered radiance is assumed to originate directly from light sources, treating volumetric lighting similarly to direct lighting in surface rendering [S1, unofficial]. It differs from multiple scattering methods by ignoring light that has bounced off other particles, and from homogeneous approximations by allowing for arbitrary spatial variation in density and phase functions [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** Ray origin $\mathbf{o}$, direction $\boldsymbol{\omega}$, scene geometry, light sources, and volume density field $\sigma_t(\mathbf{x})$.
*   **Output:** Radiance $L(\mathbf{o}, \boldsymbol{\omega})$ along the ray.

**Data Structures**
*   **Volume Grid:** A sparse or dense grid storing $\sigma_a(\mathbf{x})$ and $\sigma_s(\mathbf{x})$. Not given in the sources.
*   **Light List:** Array of light sources with position, intensity, and geometry.

**Numbered Steps**
1.  **Ray Marching:** Trace the ray from $\mathbf{o}$ to the first intersection or far plane.
2.  **Transmittance Calculation:** Compute the transmittance $T(\mathbf{o}, \mathbf{x})$ from the ray origin to a sample point $\mathbf{x}$ using the Beer-Lambert law:
    $$ T(\mathbf{o}, \mathbf{x}) = e^{-\int_{\mathbf{o}}^{\mathbf{x}} \sigma_t(\mathbf{u}) d\mathbf{u}} $$
3.  **In-Scattering Integration:** For each sample point $\mathbf{x}$ along the ray, compute the in-scattered radiance $L_s(\mathbf{x}, \boldsymbol{\omega})$ by integrating incident radiance from all lights over the sphere $S^2$:
    $$ L_s(\mathbf{x}, \boldsymbol{\omega}) = \int_{S^2} p(\mathbf{x}, \boldsymbol{\omega}', \boldsymbol{\omega}) L_i(\mathbf{x}, \boldsymbol{\omega}') d\boldsymbol{\omega}' $$
    where $p$ is the phase function and $L_i$ is incident radiance from lights [S1, unofficial].
4.  **Next Event Estimation (NEE):** Substitute the spherical integration by directly querying each light source. For a point light at $\mathbf{l}$ with intensity $I_l$:
    $$ L_s(\mathbf{x}, \boldsymbol{\omega}) \approx \sum_{l} \frac{I_l}{\|\mathbf{x} - \mathbf{l}\|^2} p(\mathbf{x}, \frac{\mathbf{l}-\mathbf{x}}{\|\mathbf{l}-\mathbf{x}\|}, \boldsymbol{\omega}) T(\mathbf{l}, \mathbf{x}) V(\mathbf{l}, \mathbf{x}) $$
    where $V(\mathbf{l}, \mathbf{x})$ is the visibility (shadow) between the light and the sample point [S1, unofficial].
5.  **Accumulation:** Accumulate the contribution of each sample, weighted by the transmittance from the ray origin:
    $$ L(\mathbf{o}, \boldsymbol{\omega}) = \int_{\mathbf{o}}^{\mathbf{far}} T(\mathbf{o}, \mathbf{x}) L_s(\mathbf{x}, \boldsymbol{\omega}) \sigma_s(\mathbf{x}) d\mathbf{x} $$
    Discretized as a sum over samples.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def single_scattering(ray_origin, ray_direction, lights, volume_sampler):
    total_radiance = 0.0
    current_transmittance = 1.0
    step_size = 0.1  # Not given in the sources.
    
    for t in range(0, 100, step_size):  # Not given in the sources.
        x = ray_origin + ray_direction * t
        sigma_s = volume_sampler.get_sigma_s(x)
        sigma_t = volume_sampler.get_sigma_t(x)
        
        # Calculate transmittance from origin to x
        # In practice, this is accumulated during the march
        transmittance_to_x = current_transmittance
        
        in_scatter = 0.0
        for light in lights:
            # Vector to light
            to_light = light.position - x
            dist = length(to_light)
            if dist < 1e-6: continue
            
            direction_to_light = normalize(to_light)
            
            # Shadow test
            if not volume_sampler.is_shadowed(x, light.position):
                # Phase function evaluation
                phase = evaluate_phase_function(x, direction_to_light, ray_direction)
                
                # Light intensity attenuation
                light_intensity = light.intensity / (dist * dist)
                
                # Transmittance from light to x
                transmittance_light_to_x = exp(-volume_sampler.integrate_sigma_t(x, light.position))
                
                in_scatter += light_intensity * phase * transmittance_light_to_x
        
        # Accumulate contribution
        # dL = T * L_s * sigma_s * dt
        contribution = transmittance_to_x * in_scatter * sigma_s * step_size
        total_radiance += contribution
        
        # Update transmittance for next step
        current_transmittance *= exp(-sigma_t * step_size)
        
        if current_transmittance < 1e-4: break  # Russian Roulette or threshold [G]
        
    return total_radiance
```

**Parameters**
*   **Step Size:** Depends on volume density. Not given in the sources.
*   **Phase Function:** Henyey-Greenstein is common. Not given in the sources.

**Edge Cases**
*   **High Density:** Requires smaller step sizes or adaptive sampling to avoid aliasing.
*   **Shadow Boundaries:** Discontinuities in visibility can cause noise. Not given in the sources.

## Subtopics

### Monochromatic Mie Scattering Shading
Treats Mie scattering intensity $I_{SM}$ as monochromatic, occupying only one channel in the scattering texture, and shades it during rendering by multiplying with attenuated direct illumination $I_0^I$ [S2]. Alternatively, the proportionality rule from [Bruneton and Neyret 2008] can be used instead of multiplying by attenuated direct illumination [S2].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot L)$, where $N$ is the number of samples per ray and $L$ is the number of lights.
*   **Memory Complexity:** $O(V)$, where $V$ is the volume data size.
*   **Strengths:** Physically plausible for thin media; simpler to implement than multiple scattering.
*   **Limitations:** Inaccurate for dense media where multiple scattering is significant.
*   **When to Use:** Thin fog, smoke, or when performance is critical and multiple scattering is negligible.
*   **When Not to Use:** Dense clouds, water with high particulate content.

## Background
Single scattering is a fundamental approximation in volumetric rendering, derived from the radiative transfer equation by neglecting the in-scattering term's dependence on previously scattered light [S1, unofficial]. It is widely used in real-time graphics and offline rendering for atmospheric effects and thin participating media.

## Sources
[S1] l0exy1h/VolumetricPathTracing-public, https://github.com/l0exy1h/VolumetricPathTracing-public, fetched 2026-10-26 (unofficial)
[S2] Real-time spectral scattering in large-scale natural participating media, https://doi.org/10.1145/1925059.1925074, fetched 2026-10-28
