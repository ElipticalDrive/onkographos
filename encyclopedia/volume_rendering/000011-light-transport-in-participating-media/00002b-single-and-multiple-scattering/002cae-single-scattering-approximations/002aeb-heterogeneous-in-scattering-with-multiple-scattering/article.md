# Heterogeneous In-scattering with Multiple Scattering

Also called: Recursive Volumetric In-scattering, Direct and Indirect Volumetric Lighting

## Overview
This method computes the total radiance contribution from light scattering within a heterogeneous participating medium by combining direct illumination from light sources with recursively computed indirect illumination (multiple scattering) up to a fixed maximum depth [S1, unofficial]. It differs from single-scattering approximations by accounting for light that scatters more than once before reaching the viewer, and from full path tracing by using a truncated recursion depth rather than a probabilistic path termination [S1, unofficial]. It builds on [Single and multiple scattering](35-single-and-multiple-scattering.md) by implementing the recursive solution to the radiative transfer equation for heterogeneous media.

## How it works
**Inputs and Outputs**
*   **Input:** Ray origin $\mathbf{o}$, direction $\boldsymbol{\omega}$, current recursion depth $d$, scene geometry, light sources, and medium coefficients $\mu_a$ (absorption) and $\mu_s$ (scattering).
*   **Output:** Outgoing radiance $L_o$ at the ray origin.

**Data Structures**
*   **Medium Coefficients:** Scalar fields or constant values for $\mu_a$ and $\mu_s$. Typical base values are $\mu_a = 0.05$ and $\mu_s = 0.1$ [S1, unofficial].
*   **Recursion Depth:** Integer counter $d$. Maximum depth is typically 5 [S1, unofficial].

**Steps**
1.  **Evaluate Local Coefficients:** Sample $\mu_a(\mathbf{p})$ and $\mu_s(\mathbf{p})$ at the current position $\mathbf{p}$.
2.  **Check Interaction Probability:** Calculate the local extinction $\mu_t = \mu_a + \mu_s$. If $\mu_t < 0.0001$, return 0, as no significant interaction occurs [S1, unofficial].
3.  **Compute Single Scattering Albedo:** Calculate the probability of scattering versus extinction:
    $$ p_{scatter} = \frac{\mu_s}{\mu_t} $$
4.  **Direct In-scattering:** For each light source:
    *   Sample a point on the light source.
    *   Cast a shadow ray to the current position. If occluded, skip.
    *   Compute transmittance $T_{to\_light}$ from the current position to the light sample using 16-step integration through the heterogeneous medium [S1, unofficial].
    *   Accumulate direct contribution:
        $$ L_{direct} += L_e \cdot p(\omega_i, \omega_o) \cdot \frac{1}{r^2} \cdot T_{to\_light} \cdot A_{light} $$
        where $L_e$ is emitted radiance, $r$ is distance, and $A_{light}$ is light area factor.
5.  **Indirect In-scattering (Recursive):** If $d < 5$:
    *   Sample a hemisphere direction $\omega'$ using a `HemisphericalSampler`.
    *   Cast a ray from the current position in direction $\omega'$.
    *   If the ray hits an object at distance $t_{hit}$:
        *   Compute transmittance $T_{to\_scatter}$ from current position to hit point.
        *   Recursively call `computeInscattering` for the segment up to the surface to get $L_{indirect}$ from the environment/surfaces.
        *   Add to total:
            $$ L_{indirect} += L_{indirect\_env} \cdot p_{scatter} \cdot T_{to\_scatter} \cdot \frac{1}{4\pi} $$
            Note: The sources use a hardcoded isotropic phase function $\frac{1}{4\pi}$ for this term [S1, unofficial].
6.  **Combine:** Sum direct and indirect contributions.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_inscattering(ray_origin, ray_direction, depth, scene, lights, mu_a_base, mu_s_base):
    """
    Computes radiance from in-scattering in a heterogeneous medium.
    """
    # 1. Evaluate local coefficients at ray origin
    pos = ray_origin
    mu_a = mu_a_base # Simplified: assume constant or sampled from field
    mu_s = mu_s_base
    
    # 2. Check for negligible interaction
    mu_t = mu_a + mu_s
    if mu_t < 0.0001:
        return 0.0
    
    # 3. Compute scattering albedo
    p_scatter = mu_s / mu_t
    
    total_radiance = 0.0
    
    # 4. Direct In-scattering
    for light in lights:
        # Sample light position (simplified)
        light_pos, light_normal, light_area = light.sample()
        
        # Shadow ray
        to_light = light_pos - pos
        dist_to_light = length(to_light)
        if dist_to_light < 1e-6:
            continue
            
        shadow_ray = Ray(pos, normalize(to_light), dist_to_light)
        if scene.is_occluded(shadow_ray):
            continue
            
        # Compute transmittance to light (16-step integration)
        T_to_light = compute_transmittance_16_steps(pos, light_pos, mu_a, mu_s)
        
        # Geometric term and phase function
        # Note: Sources use hardcoded isotropic phase function 1/(4*pi)
        phase = 1.0 / (4.0 * PI)
        geometric = light.emitted_radiance * light_area / (dist_to_light ** 2)
        
        total_radiance += geometric * phase * T_to_light
        
    # 5. Indirect In-scattering (Recursive)
    if depth < 5:
        # Sample hemisphere direction
        omega_indirect = hemispherical_sampler.sample()
        
        # Cast ray to find indirect source
        indirect_ray = Ray(pos, omega_indirect, MAX_DISTANCE)
        hit = scene.intersect(indirect_ray)
        
        if hit.is_valid():
            # Transmittance to the scattering point
            T_to_scatter = compute_transmittance_16_steps(pos, hit.point, mu_a, mu_s)
            
            # Recursive call for indirect radiance from the hit surface/environment
            # This assumes the recursive call handles the surface interaction and further scattering
            L_indirect_env = compute_inscattering(hit.point, hit.normal, hit.t, omega_indirect, depth + 1, scene, lights, mu_a_base, mu_s_base)
            
            # Add indirect contribution
            # Formula from sources: L_indirect * albedo * T_to_scatter * (1/(4*pi))
            total_radiance += L_indirect_env * p_scatter * T_to_scatter * (1.0 / (4.0 * PI))
            
    return total_radiance

def compute_transmittance_16_steps(start, end, mu_a, mu_s):
    """
    Approximates transmittance using 16 uniform steps.
    """
    direction = normalize(end - start)
    distance = length(end - start)
    step_size = distance / 16.0
    
    optical_depth = 0.0
    for i in range(16):
        current_pos = start + direction * (i + 0.5) * step_size
        mu_t = mu_a + mu_s
        optical_depth += mu_t * step_size
        
    return exp(-optical_depth)
```

**Parameters**
*   `mu_a`: Base absorption coefficient. Default: 0.05 [S1, unofficial].
*   `mu_s`: Base scattering coefficient. Default: 0.1 [S1, unofficial].
*   `depth`: Current recursion depth. Stops if `depth >= 5` [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Isotropic Phase Function:** The implementation uses a hardcoded isotropic phase function ($1/4\pi$) for both direct and indirect contributions, ignoring the material's actual phase function [S1, unofficial]. This is a significant simplification that may reduce accuracy for anisotropic media.
*   **Recursion Limit:** Hardcoded depth limit of 5. Higher albedo media may require deeper recursion for convergence, but this is capped [S1, unofficial].
*   **Transmittance Approximation:** Using only 16 steps for transmittance integration may introduce errors in highly heterogeneous media with sharp density gradients.

## Subtopics

### Heterogeneous Radiance Computation
This variant computes the total radiance for a ray by combining volumetric in-scattering, surface emission, and surface reflection, while accounting for transmittance through the heterogeneous medium [S1, unofficial]. It extends the basic in-scattering calculation by handling surface interactions at the end of the ray segment. Steps include: finding the closest scene intersection; if none, returning background radiance ($ambient + bgColor \cdot \exp(-(mu_a + mu_s) \cdot avg\_density \cdot 10.0)$, clamped to min 0.2); computing in-scattering $L_{inscatter}$ for the segment; adding surface emission ($material.getEmissiveRadiance() \cdot T$); and if depth < MaxDepth, computing reflected radiance (direct + indirect) and adding it to $L_o$ multiplied by transmittance $T$ [S1, unofficial].

### Perturbation Solution for High Albedo
This approach addresses defects in standard approximations for scattering media with very high albedo (e.g., clouds), where shadowing artifacts appear due to neglecting higher-order scattering [S2, unofficial]. It uses a perturbation solution where the total radiance $Z$ is expanded as a power series in the single scattering albedo $fl$: $Z = \sum fl^k Z_k$. The equations for order $k$ are derived by substituting into the original equation and equating like powers, resulting in $LI_k + MI_k = -MI_{k-1}$ [S2, unofficial]. This presents a series of forced conservative scattering equations, allowing approximation of solutions for the conservative case ($w_0 = 1$) [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot L \cdot D)$, where $N$ is the number of light sources, $L$ is the number of integration steps for transmittance (16), and $D$ is the recursion depth (5). The complexity grows linearly with the number of lights and exponentially with depth if not capped.
*   **Memory Complexity:** $O(D)$ for the recursion stack.
*   **Performance:** The method is computationally expensive due to recursive ray casting and multiple transmittance integrations per pixel. The 16-step integration for transmittance is a fixed cost per light per recursion level.
*   **Strengths:** Handles heterogeneous media and multiple scattering effects better than single-scattering approximations. Simple to implement compared to full path tracing.
*   **Limitations:** The hardcoded isotropic phase function reduces physical accuracy. The fixed recursion depth may not converge for high-albedo media. The 16-step transmittance integration may be insufficient for highly variable density fields.
*   **When to Use:** Suitable for offline rendering or real-time applications where multiple scattering is important but full path tracing is too slow, and the medium is not extremely high-albedo.
*   **When Not to Use:** Not suitable for highly anisotropic media without modification, or for media with very high albedo where the perturbation solution is more appropriate.

## Background
This method is derived from the principles of volumetric path tracing and the radiative transfer equation. The specific implementation details, including the 16-step transmittance integration and the hardcoded isotropic phase function, are found in an unofficial GitHub repository [S1, unofficial]. The perturbation solution for high albedo media is discussed in a document on ray tracing for volume densities [S2, unofficial]. The approach builds on the conservation of energy and the differential forms of the RTE as described in standard graphics literature [S1, unofficial].

## Sources
[S1] ainagutierrez01/Volumetric-Path-Tracing-, https://github.com/ainagutierrez01/Volumetric-Path-Tracing-, fetched 2026-10-26 (unofficial)
[S2] Ray Tracing for Volume Densities, https://www.scribd.com/document/746685923/ray-tracking-volume-densities, fetched 2026-10-25 (unofficial)
