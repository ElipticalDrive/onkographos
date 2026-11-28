# Direct Lighting for Volumes (Single Scattering)

Also called: Single Scattering via Light Source Importance Sampling, Single scattering estimator

## Overview
This method calculates the contribution to ray radiance from direct illumination (single scattering) by importance sampling light sources, solving the failure of brute-force scattering sampling for small light sources [S1, unofficial]. It replaces the low-probability event of a scattering event coincidentally directing light toward a source with a direct query to the light geometry, resembling a Kajiya-style path tracer for participating media [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** Camera ray origin $\mathbf{x}$, direction $\omega$, medium properties ($\sigma_s, \sigma_t$), light source geometry and emission $L_e^0$, and a sampling strategy for light sources defined by area PDF $p_A(\mathbf{z})$.
*   **Output:** An unbiased estimator for the single-scattered radiance $L^1(\mathbf{x}, \omega)$.

**Data Structures**
*   Light source representation supporting area sampling and normal retrieval $n(\mathbf{z})$.
*   Medium representation supporting extinction $\tau(\mathbf{x}, \mathbf{y})$ and scattering coefficient $\sigma_s(\mathbf{x})$ evaluation.

**Steps**
1.  **Sample Light Source:** Generate a point $\mathbf{z}$ on the light source surface using the area-based PDF $p_A(\mathbf{z})$ [S1, unofficial].
2.  **Compute Direction:** Determine the direction from the scattering point $\mathbf{x}'$ (the first interaction point along the camera ray $\mathbf{x}$) to the light source point $\mathbf{z}$. Let $\omega'$ be the unit vector pointing from $\mathbf{x}'$ to $\mathbf{z}$ [S1, unofficial].
3.  **Check Visibility:** Compute the transmittance $\tau(\mathbf{x}', \mathbf{z})$ between the scattering point and the light source. If occluded, the contribution is zero [S1, unofficial].
4.  **Convert PDF:** Convert the area-based PDF $p_A(\mathbf{z})$ to a solid-angle-based PDF $p_\sigma(\omega')$ relative to the scattering point $\mathbf{x}'$ [S1, unofficial]:
    $$ p_\sigma(\omega') = p_A(\mathbf{z}) \frac{\|\mathbf{x}' - \mathbf{z}\|^2}{\langle \omega', n(\mathbf{z}) \rangle} $$
    where $n(\mathbf{z})$ is the surface normal at $\mathbf{z}$ and $\langle \cdot, \cdot \rangle$ denotes the dot product [S1, unofficial].
5.  **Evaluate Estimator:** Compute the single scattering estimator $g(\omega')$ [S1, unofficial]:
    $$ g(\omega') = \frac{\tau(\mathbf{x}, \mathbf{z}) \sigma_s(\mathbf{x}') f_p(\mathbf{x}', \omega, \omega') L_e^0(\mathbf{x}', \omega') \langle \omega', n(\mathbf{z}) \rangle}{p_A(\mathbf{z}) \|\mathbf{x}' - \mathbf{z}\|^2} $$
    where:
    *   $\tau(\mathbf{x}, \mathbf{z})$ is the transmittance from the camera ray origin $\mathbf{x}$ to the light source point $\mathbf{z}$ (attenuating the light before it reaches the scattering point).
    *   $\sigma_s(\mathbf{x}')$ is the scattering coefficient at the interaction point.
    *   $f_p(\mathbf{x}', \omega, \omega')$ is the phase function evaluating the probability of scattering from direction $\omega'$ to $\omega$.
    *   $L_e^0(\mathbf{x}', \omega')$ is the emitted radiance from the light source at $\mathbf{z}$ in direction $\omega'$, attenuated by the medium between $\mathbf{z}$ and $\mathbf{x}'$ (often simplified to $L_e(\mathbf{z}, \omega')$ if the medium is homogeneous or if pre-attenuation is handled by $\tau(\mathbf{x}, \mathbf{z})$ and $\tau(\mathbf{x}', \mathbf{z})$ separately; note the source formula uses $L_e^0$ which is defined as $\tau(\mathbf{x}, \mathbf{z}) L_e^0(\mathbf{z}, \omega)$ in the text, but the final estimator formula explicitly includes $\tau(\mathbf{x}, \mathbf{z})$ and $L_e^0(\mathbf{x}', \omega')$. The source defines $L^0(\mathbf{x}, \omega) = \tau(\mathbf{x}, \mathbf{z}) L_e^0(\mathbf{z}, \omega)$. The final estimator uses $L_e^0(\mathbf{x}', \omega')$ which likely refers to the source radiance $L_e(\mathbf{z}, \omega')$ attenuated by the path from $\mathbf{z}$ to $\mathbf{x}'$? No, the source text says $L_e^0(\mathbf{x}', \omega')$ in the final formula. Given $L^0(\mathbf{x}', \omega') = \tau(\mathbf{x}', \mathbf{z}) L_e(\mathbf{z}, \omega')$, and the estimator has $\tau(\mathbf{x}, \mathbf{z})$ outside, it implies the term is $\tau(\mathbf{x}, \mathbf{z}) \sigma_s(\mathbf{x}') f_p L_e(\mathbf{z}, \omega') \tau(\mathbf{x}', \mathbf{z}) \dots$? Let's stick strictly to the source's final formula: $g(\omega') = \frac{\tau(\mathbf{x}, \mathbf{z}) \sigma_s(\mathbf{x}') f_p(\mathbf{x}', \omega, \omega') L_e^0(\mathbf{x}', \omega') \langle \omega', n(\mathbf{z}) \rangle}{p_A(\mathbf{z}) \|\mathbf{x}' - \mathbf{z}\|^2}$. The source defines $L_e^0$ as emitted radiance. It is likely $L_e(\mathbf{z}, \omega')$ [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def estimate_single_scattering(ray_origin, ray_direction, medium, light_source):
    # 1. Sample first interaction point x' along the camera ray
    # This depends on the parent method's path generation (e.g., Delta Tracking)
    x_prime = sample_interaction_point(ray_origin, ray_direction, medium)
    
    # 2. Sample a point z on the light source
    z, pdf_area = light_source.sample_area()
    
    # 3. Compute direction from scattering point to light source
    direction_to_light = normalize(z - x_prime)
    distance_to_light = length(z - x_prime)
    
    # 4. Compute visibility/transmittance from x' to z
    # Note: The source formula uses tau(x, z) and tau(x, x') is implicit in x' sampling?
    # The source formula: tau(x, z) * sigma_s(x') * ...
    # tau(x, z) is transmittance from camera origin to light source point.
    # This accounts for attenuation along the entire path from camera to light.
    # However, standard single scattering splits this:
    # Transmittance from camera to x': tau(x, x')
    # Transmittance from x' to z: tau(x', z)
    # The source formula explicitly writes tau(x, z). 
    # tau(x, z) = tau(x, x') * tau(x', z).
    # Since x' is the first interaction, tau(x, x') is the transmittance to the first hit.
    
    transmittance_camera_to_light = medium.transmittance(ray_origin, z)
    
    # 5. Evaluate phase function
    phase = medium.phase_function(x_prime, ray_direction, direction_to_light)
    
    # 6. Evaluate scattering coefficient
    sigma_s = medium.scattering_coefficient(x_prime)
    
    # 7. Evaluate light emission
    # L_e^0(x', omega') in source likely means L_e(z, omega') attenuated?
    # Source defines L^0(x, omega) = tau(x, z) L_e^0(z, omega).
    # The final estimator has tau(x, z) * L_e^0(x', omega').
    # This implies L_e^0(x', omega') is the source radiance L_e(z, omega')?
    # Or is it L_e(z, omega') attenuated by tau(x', z)?
    # If L_e^0(x', omega') = L_e(z, omega') * tau(x', z), then the term becomes:
    # tau(x, z) * tau(x', z) * L_e ... which would be tau(x, z)^2.
    # Standard physics: L = tau(x, x') * sigma_s * phase * tau(x', z) * L_e(z).
    # tau(x, z) = tau(x, x') * tau(x', z).
    # So tau(x, z) * L_e(z) = tau(x, x') * tau(x', z) * L_e(z).
    # This matches standard physics if L_e^0(x', omega') in the source formula 
    # is interpreted as the source radiance L_e(z, omega') without extra attenuation.
    # Let's assume L_e^0(x', omega') refers to the emitted radiance L_e(z, omega').
    light_radiance = light_source.emission(z, direction_to_light)
    
    # 8. Compute normal at z
    normal_z = light_source.normal(z)
    
    # 9. Compute dot product
    dot_product = dot(direction_to_light, normal_z)
    
    # 10. Compute estimator
    # Avoid division by zero
    if pdf_area == 0 or dot_product == 0:
        return 0.0
        
    numerator = transmittance_camera_to_light * sigma_s * phase * light_radiance * dot_product
    denominator = pdf_area * (distance_to_light ** 2)
    
    estimator = numerator / denominator
    
    return estimator
```

**Parameters**
*   $p_A(\mathbf{z})$: Area PDF of the light source sampler. Must be non-zero where the light is visible.

**Edge Cases**
*   **Zero Dot Product:** If the light surface is tangent to the ray from $\mathbf{x}'$ to $\mathbf{z}$, the solid angle conversion fails. Guard against division by zero [S1, unofficial].
*   **Occlusion:** If $\mathbf{z}$ is not visible from $\mathbf{x}'$, the transmittance is zero [S1, unofficial].
*   **Small Light Sources:** This method is specifically designed to handle small light sources where brute-force scattering sampling fails [S1, unofficial].

## Operating characteristics
*   **Complexity:** $O(1)$ per sample, assuming constant-time light sampling and visibility checks.
*   **Strengths:** Efficiently captures direct illumination from small, bright light sources. Reduces variance significantly compared to brute-force scattering sampling in scenes with small lights [S1, unofficial].
*   **Limitations:** Only accounts for single scattering. Multiple scattering events require additional estimators or path tracing steps. Requires a light source sampler [S1, unofficial].
*   **When to use:** Scenes with small, bright light sources where brute-force scattering sampling produces noisy results.
*   **When not to use:** Scenes with only large, area lights where brute-force sampling might be sufficient, or when multiple scattering is dominant and a full path tracer is used.

## Background
This method was developed as part of volumetric path tracing techniques, drawing parallels to surface path tracing (Kajiya-style). It addresses the limitation of "BRDF-only sampling" in volumes, which fails for small light sources [S1, unofficial]. The approach generalizes multiple importance sampling and Russian Roulette from surface path tracing to participating media [S1, unofficial].

## Sources
[S1] PDF Volumetric Path Tracing - Department of Computer Science, https://www.cs.cornell.edu/courses/cs6630/2012sp/notes/09volpath.pdf, fetched 2026-10-27 (unofficial)
