# Heterogeneous In-scattering with Multiple Scattering


## Overview
This method computes the radiance contribution from light scattering within a heterogeneous participating medium, handling both direct lighting from light sources and recursively computing indirect lighting (multiple scattering) up to a maximum depth [S1, unofficial]. It differs from its siblings, such as [Single-scattering homogeneous volume rendering](1135-single-scattering-homogeneous-volume-rendering.md) and [Volume Single Scattering (Monte Carlo with Point Light)](2293-volume-single-scattering-monte-carlo-with-point-light.md), by explicitly supporting spatially varying density and coefficients (heterogeneity) and approximating multiple scattering events through recursive ray casting, rather than relying solely on single-scattering approximations or homogeneous assumptions.

## How it works
**Inputs:**
*   Ray origin $\mathbf{o}$ and direction $\boldsymbol{\omega}$.
*   Scene geometry (for intersection tests).
*   Light sources (position, intensity).
*   Base absorption coefficient $\mu_a = 0.05$ [S1, unofficial].
*   Base scattering coefficient $\mu_s = 0.1$ [S1, unofficial].
*   Maximum recursion depth $D_{max} = 5$ [S1, unofficial].

**Outputs:**
*   Outgoing radiance $L_o$ along the ray.

**Data Structures:**
*   The medium is assumed to be sampled via a function `getDensity(x)` and `getExtinction(x)` which return local $\mu_t(\mathbf{x}) = \mu_a + \mu_s$ and $\mu_s(\mathbf{x})$ respectively. [G]
*   A `HemisphericalSampler` is used to generate directions for indirect lighting. [S1, unofficial]

**Steps:**

1.  **Termination Check:** If the local extinction coefficient $\mu_t < 0.0001$, return 0 (no interaction) [S1, unofficial].
2.  **Single Scattering Albedo:** Calculate the probability of scattering versus extinction:
    $$ p_{scatter} = \frac{\mu_s}{\mu_t} $$
3.  **Direct In-scattering:** For each light source:
    *   Sample the light position $\mathbf{l}$.
    *   Cast a shadow ray from the current point to $\mathbf{l}$.
    *   If the path is not occluded, compute the transmittance $T_{to\_light}$ from the current point to the light using 16-step integration [S1, unofficial].
    *   Accumulate the direct contribution:
        $$ L_{direct} += I_l \cdot p(\boldsymbol{\omega}_l, \boldsymbol{\omega}) \cdot \frac{1}{\|\mathbf{x} - \mathbf{l}\|^2} \cdot T_{to\_light} \cdot A_{light} $$
        where $I_l$ is light intensity, $p$ is the phase function, and $A_{light}$ is the light area [S1, unofficial].
4.  **Indirect In-scattering (Multiple Scattering):** If current depth $d < D_{max}$:
    *   Sample a hemisphere direction $\boldsymbol{\omega}'$ using `HemisphericalSampler`.
    *   Cast a ray from the scatter position $\mathbf{x}$ in direction $\boldsymbol{\omega}'$.
    *   If the ray hits an object at distance $d_{hit}$:
        *   Recursively compute the in-scattering $L_{indirect}$ for the segment from $\mathbf{x}$ to the surface hit point.
        *   Add to the total in-scattering:
            $$ L_{indirect\_contrib} = L_{indirect} \cdot \rho \cdot T_{to\_scatter} \cdot \frac{1}{4\pi} $$
            where $\rho$ is the surface albedo, $T_{to\_scatter}$ is the transmittance from the surface back to $\mathbf{x}$, and $\frac{1}{4\pi}$ is the hardcoded isotropic phase function [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_inscattering(ray_origin, ray_direction, depth, max_depth=5):
    """
    Computes in-scattering for a ray in a heterogeneous medium.
    """
    # 1. Check for negligible interaction
    local_mu_t = get_extinction(ray_origin)
    if local_mu_t < 0.0001:
        return 0.0

    # 2. Calculate single scattering albedo
    local_mu_s = get_scattering_coefficient(ray_origin)
    p_scatter = local_mu_s / local_mu_t

    L_inscatter = 0.0

    # 3. Direct lighting contribution
    for light in scene.lights:
        # Sample light position (simplified for point light)
        light_pos = light.position
        light_dir = normalize(light_pos - ray_origin)
        dist_to_light = length(light_pos - ray_origin)
        
        # Shadow ray
        if not is_occluded(ray_origin, light_dir, dist_to_light):
            # Compute transmittance using 16-step integration
            T_to_light = integrate_transmittance(ray_origin, light_dir, dist_to_light, steps=16)
            
            # Geometric attenuation and phase function
            # Note: Source uses hardcoded isotropic phase function 1/(4*pi)
            phase = 1.0 / (4.0 * PI)
            geometric = 1.0 / (dist_to_light ** 2)
            
            L_inscatter += light.intensity * phase * geometric * T_to_light * light.area

    # 4. Indirect lighting contribution (Multiple Scattering)
    if depth < max_depth:
        # Sample hemisphere direction
        indirect_dir = hemispherical_sampler.sample(ray_origin)
        
        # Cast ray to find surface interaction
        hit = intersect_scene(ray_origin, indirect_dir)
        if hit.is_valid:
            # Recursively compute in-scattering for the segment up to the surface
            # The segment length is hit.distance
            L_indirect = compute_inscattering(hit.point, indirect_dir, depth + 1, max_depth)
            
            # Transmittance from scatter point to surface
            T_to_scatter = integrate_transmittance(ray_origin, indirect_dir, hit.distance, steps=16)
            
            # Add contribution: L_indirect * albedo * T_to_scatter * (1/(4*pi))
            # Note: Source uses hardcoded isotropic phase function
            surface_albedo = hit.material.albedo
            L_inscatter += L_indirect * surface_albedo * T_to_scatter * (1.0 / (4.0 * PI))

    return L_inscatter

def compute_total_radiance(ray_origin, ray_direction, depth=0, max_depth=5):
    """
    Computes total radiance combining volumetric in-scattering, surface emission, and reflection.
    """
    # 1. Find closest intersection
    hit = intersect_scene(ray_origin, ray_direction)
    
    if not hit.is_valid:
        # No intersection: return background radiance
        # Formula: ambient + bgColor * exp(-(mu_a + mu_s) * avg_density * 10.0)
        # Clamped to min 0.2
        avg_density = 1.0 # Assumed constant for background calculation if not specified
        mu_a = 0.05
        mu_s = 0.1
        bg_term = ambient + background_color * exp(-(mu_a + mu_s) * avg_density * 10.0)
        return max(0.2, bg_term)

    # 2. Compute transmittance to the hit point
    T = integrate_transmittance(ray_origin, ray_direction, hit.distance, steps=16)
    
    # 3. Compute in-scattering contribution for the segment
    L_inscatter = compute_inscattering(ray_origin, ray_direction, depth, max_depth)
    
    # 4. Add surface emission
    L_emission = hit.material.get_emissive_radiance() * T
    
    # 5. Compute reflected radiance if depth allows
    L_reflection = 0.0
    if depth < max_depth:
        # Compute reflected radiance (direct + indirect)
        # This typically involves shading the hit point with the computed in-scattering
        # as an environment map or direct lighting calculation
        L_reflection = shade_surface(hit, L_inscatter) * T

    return L_inscatter + L_emission + L_reflection
```

**Edge Cases and Pitfalls:**
*   The implementation uses a hardcoded isotropic phase function ($1/4\pi$) for both direct and indirect scattering contributions, ignoring the actual phase function of the material [S1, unofficial].
*   Precision issues may arise in `integrate_transmittance` if density varies rapidly; 16 steps is a fixed choice [S1, unofficial].
*   Recursion depth is capped at 5 to prevent infinite loops and excessive computation [S1, unofficial].

## Subtopics

### [Correction Texture for Surface Albedo](9421-correction-texture-for-surface-albedo.md)
This method corrects the main scattering texture for light reflected off the ground, which is precomputed assuming a homogeneous mean surface albedo $\bar{a}$. It allows the renderer to remove these homogeneous contributions from the main texture and evaluate surface albedos and reflection characteristics separately at runtime for surfaces with varying properties.

### Heterogeneous Radiance Computation
Computes the total radiance for a ray by combining volumetric in-scattering, surface emission, and surface reflection, accounting for transmittance through the heterogeneous medium [S1, unofficial]. It finds the closest intersection with scene geometry; if none, returns background radiance $ambient + bgColor \cdot \exp(-(mu_a + mu_s) \cdot avg\_density \cdot 10.0)$, clamped to min 0.2 [S1, unofficial]. It then computes in-scattering $L_{inscatter}$, adds surface emission scaled by transmittance $T$, and if depth < MaxDepth, computes reflected radiance (direct + indirect) and adds it to $L_o$ multiplied by $T$ [S1, unofficial].

### Perturbation Solution for High Albedo
Used when the scattering medium has a very high albedo (e.g., clouds), where standard approximations suffer from defects like abnormal darkening due to shadowing [S2, unofficial]. The solution expands the radiance $Z$ into a power series in the single-scattering albedo $fl$:
$$ Z = \sum fl^k Z_k $$
Substituting this into the original equation and equating like powers of $fl$ yields a set of equations:
$$ LI_k + MI_k = -MI_{k-1} $$
This presents a series of forced conservative ($w_0 = 1$) scattering equations, allowing approximation of solutions for the conservative case [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Exponential in the number of scattering events due to recursion, but capped at depth 5 [S1, unofficial]. Each step involves ray intersection and 16-step transmittance integration.
*   **Memory Complexity:** Low, as it does not store large precomputed textures for the volume itself, but relies on scene geometry and light data.
*   **Strengths:** Handles heterogeneous media and multiple scattering effects, providing more realistic lighting than single-scattering methods.
*   **Limitations:** Computationally expensive due to recursion and multiple ray casts. The hardcoded isotropic phase function reduces accuracy for anisotropic media [S1, unofficial].
*   **When to use:** When realistic multiple scattering and heterogeneous density are required, and performance allows for recursive ray tracing.
*   **When not to use:** For real-time applications requiring high frame rates, or when the medium is homogeneous and single-scattering is sufficient.

## Background
This method originates from volumetric path tracing implementations, specifically as seen in the unofficial repository [S1, unofficial]. It builds upon the principles of single-scattering approximations but extends them to handle multiple scattering through recursion. The perturbation solution for high albedo is derived from earlier work on ray tracing for volume densities, addressing defects in modeling clouds [S2, unofficial].

## Sources
[S1] ainagutierrez01/Volumetric-Path-Tracing-, https://github.com/ainagutierrez01/Volumetric-Path-Tracing-, fetched 2026-10-26 (unofficial)
[S2] Ray Tracing for Volume Densities, https://www.scribd.com/document/746685923/ray-tracking-volume-densities, fetched 2026-10-25 (unofficial)
