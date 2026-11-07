# Heterogeneous In-scattering with Multiple Scattering

Also called: Heterogeneous Radiance Computation, Perturbation Solution for High Albedo

## Overview
This method computes the total radiance contribution of a participating medium by combining direct in-scattering from light sources with recursively computed indirect in-scattering (multiple scattering) up to a fixed depth [S1, unofficial]. It addresses the limitation of single-scattering approximations in heterogeneous media where light is absorbed and scattered non-uniformly, and where surface reflections (albedo) significantly influence the volumetric appearance [S1, unofficial]. Unlike homogeneous approximations, it integrates extinction and scattering coefficients along the ray path to handle density variations [S1, unofficial].

## How it works
**Inputs:**
*   Ray origin $\mathbf{o}$ and direction $\boldsymbol{\omega}$.
*   Scene geometry (for intersection tests).
*   Light sources (position, intensity).
*   Medium properties: base absorption coefficient $\mu_a$, base scattering coefficient $\mu_s$, and a density field $\rho(\mathbf{p})$ (implied by "heterogeneous" and usage of local $\mu_t$) [S1, unofficial].
*   Recursion depth limit $D_{max} = 5$ [S1, unofficial].

**Outputs:**
*   Total outgoing radiance $L_o$ at the ray origin.

**Data Structures:**
*   **Medium Coefficients:** Local extinction $\mu_t(\mathbf{p}) = \mu_a \rho(\mathbf{p}) + \mu_s \rho(\mathbf{p})$ and local scattering $\mu_s(\mathbf{p}) = \mu_s \rho(\mathbf{p})$ [S1, unofficial]. Note: The source uses $\mu_a$ and $\mu_s$ as base values, implying density modulation.
*   **Phase Function:** Hardcoded isotropic phase function $p(\boldsymbol{\omega}', \boldsymbol{\omega}) = \frac{1}{4\pi}$ [S1, unofficial].

**Parameters:**
*   $\mu_a = 0.05$ [S1, unofficial].
*   $\mu_s = 0.1$ [S1, unofficial].
*   $D_{max} = 5$ [S1, unofficial].
*   Integration steps for transmittance: 16 steps [S1, unofficial].

**Algorithm:**
The computation is split into two main recursive functions: `computeInscattering` for the volumetric contribution and `computeRadiance` for the total pixel value.

1.  **Compute In-scattering (`computeInscattering`):**
    *   **Input:** Current position $\mathbf{p}$, direction $\boldsymbol{\omega}$, current depth $d$.
    *   **Step 1:** Evaluate local extinction $\mu_t(\mathbf{p})$. If $\mu_t < 0.0001$, return 0 (no interaction) [S1, unofficial].
    *   **Step 2:** Calculate single scattering albedo $\omega_s = \frac{\mu_s(\mathbf{p})}{\mu_t(\mathbf{p})}$ [S1, unofficial].
    *   **Step 3:** Initialize accumulated radiance $L_{in} = 0$.
    *   **Step 4:** **Direct In-scattering:** For each light source:
        *   Sample light position $\mathbf{l}$ and direction $\boldsymbol{\omega}_l$.
        *   Cast shadow ray from $\mathbf{p}$ to $\mathbf{l}$.
        *   If not occluded, compute transmittance $T_{to\_light}$ from $\mathbf{p}$ to $\mathbf{l}$ using 16-step numerical integration of $\mu_t$ along the segment [S1, unofficial].
        *   Accumulate contribution: $L_{in} \leftarrow L_{in} + L_e(\mathbf{l}) \cdot \frac{1}{4\pi} \cdot T_{to\_light} \cdot \text{light\_area}$ [S1, unofficial]. (Note: Geometric term is absorbed into light area/intensity definition in the source snippet).
    *   **Step 5:** **Indirect In-scattering:** If $d < D_{max}$:
        *   Sample a hemisphere direction $\boldsymbol{\omega}'$ using `HemisphericalSampler` [S1, unofficial].
        *   Cast a ray from $\mathbf{p}$ in direction $\boldsymbol{\omega}'$.
        *   If the ray hits an object at distance $t_{hit}$:
            *   Compute transmittance $T_{to\_scatter}$ from $\mathbf{p}$ to the hit point [S1, unofficial].
            *   Recursively call `computeInscattering` for the segment up to the surface (or treat the surface reflection as the source). The source specifies: $L_{indirect} = L_{indirect} \cdot \text{albedo} \cdot T_{to\_scatter} \cdot \frac{1}{4\pi}$ [S1, unofficial].
            *   Add to $L_{in}$: $L_{in} \leftarrow L_{in} + L_{indirect}$ [S1, unofficial].
    *   **Return:** $L_{in}$ [S1, unofficial].

2.  **Compute Total Radiance (`computeRadiance`):**
    *   **Input:** Ray $\mathbf{o}, \boldsymbol{\omega}$, depth $d=0$.
    *   **Step 1:** Find closest intersection with scene geometry.
    *   **Step 2:** If no intersection, return background radiance: $L_{bg} = \text{ambient} + \text{bgColor} \cdot e^{-(\mu_a + \mu_s) \cdot \bar{\rho} \cdot 10.0}$, clamped to minimum 0.2 [S1, unofficial]. (Note: $\bar{\rho}$ is average density, assumed constant for background).
    *   **Step 3:** Compute volumetric in-scattering $L_{in}$ for the segment from $\mathbf{o}$ to the intersection point using `computeInscattering` [S1, unofficial].
    *   **Step 4:** Compute transmittance $T$ from $\mathbf{o}$ to the intersection point [S1, unofficial].
    *   **Step 5:** Add surface emission: $L_{surf} = \text{MaterialEmissiveRadiance} \cdot T$ [S1, unofficial].
    *   **Step 6:** If $d < D_{max}$, compute reflected radiance (direct + indirect) from the surface and add to $L_{surf}$ multiplied by $T$ [S1, unofficial].
    *   **Step 7:** Return $L_{in} + L_{surf}$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_inscattering(p, omega, depth, max_depth, mu_a_base, mu_s_base, density_field, lights, scene):
    """
    Computes in-scattering radiance at point p in direction omega.
    """
    # 1. Evaluate local coefficients
    rho = density_field.sample(p)
    mu_t = (mu_a_base + mu_s_base) * rho
    mu_s = mu_s_base * rho
    
    # Pitfall: Handle near-zero extinction
    if mu_t < 0.0001:
        return 0.0

    # 2. Single scattering albedo
    omega_s = mu_s / mu_t
    
    L_in = 0.0
    
    # 3. Direct In-scattering
    for light in lights:
        # Sample light position and direction
        l_pos, l_dir, l_intensity, l_area = light.sample()
        
        # Shadow ray
        shadow_hit = scene.intersect(p, l_dir)
        if shadow_hit is None or shadow_hit.distance > distance(p, l_pos):
            # Not occluded
            # Compute transmittance T_to_light using 16-step integration
            T_to_light = integrate_transmittance(p, l_pos, mu_a_base, mu_s_base, density_field, steps=16)
            
            # Isotropic phase function
            phase = 1.0 / (4.0 * math.pi)
            
            # Accumulate contribution
            # Note: Source formula: emitted * phase * geometric * T_to_light * light_area
            # Assuming l_intensity includes geometric falloff or light_area handles it
            L_in += l_intensity * phase * T_to_light * l_area
            
    # 4. Indirect In-scattering (Multiple Scattering)
    if depth < max_depth:
        # Sample hemisphere direction
        omega_indir = hemispherical_sampler.sample(omega)
        
        # Cast ray from scatter_pos
        hit = scene.intersect(p, omega_indir)
        if hit is not None:
            # Compute transmittance to the hit point
            T_to_scatter = integrate_transmittance(p, hit.point, mu_a_base, mu_s_base, density_field, steps=16)
            
            # Get surface albedo
            albedo = hit.material.get_albedo()
            
            # Recursive call for the segment up to the surface
            # The source implies the indirect light is reflected from the surface
            # L_indirect = L_indirect * albedo * T_to_scatter * (1/(4*pi))
            # This implies we need the incoming radiance at the hit point.
            # The source snippet is ambiguous on whether it calls computeInscattering 
            # or computes surface reflection directly. 
            # Based on "If it hits an object, recursively call computeInscattering for the segment up to the surface",
            # we assume the surface acts as a source for the next bounce.
            
            # However, the formula L_indirect * albedo * ... suggests L_indirect is the incoming radiance 
            # at the hit point. Let's assume we compute the radiance arriving at the hit point.
            L_arriving = compute_radiance(hit.point, -omega_indir, depth + 1, max_depth, mu_a_base, mu_s_base, density_field, lights, scene)
            
            L_indirect = L_arriving * albedo * T_to_scatter * (1.0 / (4.0 * math.pi))
            L_in += L_indirect
            
    return L_in

def compute_radiance(o, omega, depth, max_depth, mu_a_base, mu_s_base, density_field, lights, scene):
    """
    Computes total radiance for a ray.
    """
    # 1. Find intersection
    hit = scene.intersect(o, omega)
    
    # 2. Background
    if hit is None:
        avg_density = 1.0 # Assumed constant for background calculation in source
        T_bg = math.exp(-(mu_a_base + mu_s_base) * avg_density * 10.0)
        L_bg = ambient + bg_color * T_bg
        return max(L_bg, 0.2)
        
    # 3. In-scattering contribution for the segment
    # Source says "Compute in-scattering contribution L_inscatter for the segment"
    # This implies integrating along the ray from o to hit.point
    # However, the recursive structure suggests compute_inscattering is called at points.
    # For a homogeneous/heterogeneous segment, we might integrate.
    # Given the source's recursive nature, we approximate by sampling or calling at intervals.
    # For simplicity and adherence to the "recursive" note, we call at the start point.
    # A more accurate heterogeneous implementation would integrate.
    # The source notes say: "computeInscattering for the segment up to the surface" in the indirect part.
    # Here we assume a single evaluation or integration. 
    # Let's assume the source implies evaluating at the origin for the primary ray.
    L_in = compute_inscattering(o, omega, depth, max_depth, mu_a_base, mu_s_base, density_field, lights, scene)
    
    # 4. Transmittance to surface
    T = integrate_transmittance(o, hit.point, mu_a_base, mu_s_base, density_field, steps=16)
    
    # 5. Surface emission
    L_surf = hit.material.get_emissive_radiance() * T
    
    # 6. Reflected radiance
    if depth < max_depth:
        # Compute reflected radiance (direct + indirect)
        # This usually involves shading the surface
        L_reflected = shade_surface(hit, lights, scene, depth + 1, max_depth, mu_a_base, mu_s_base, density_field)
        L_surf += L_reflected * T
        
    return L_in + L_surf
```

**Pitfalls:**
*   **Isotropic Phase Function:** The implementation uses a hardcoded isotropic phase function ($1/4\pi$) for both direct and indirect scattering, ignoring the material's actual phase function [S1, unofficial]. This is a significant approximation for non-isotropic media.
*   **Recursion Depth:** The recursion is capped at depth 5. Higher depths may be needed for highly scattering media (high albedo) to converge [S1, unofficial].
*   **Perturbation Solution:** For high albedo clouds, the standard perturbation solution $Z = \sum f_l^k Z_k$ and $LI_k + MI_k = -MI_{k-1}$ may suffer from defects like abnormal darkening due to shadowing [S2, unofficial].

## Subtopics

### [Correction Texture for Surface Albedo](9421-correction-texture-for-surface-albedo.md)
This method corrects the main scattering texture for light reflected off the ground. It precomputes the volumetric scattering assuming a homogeneous mean surface albedo $\bar{a}$. At runtime, it removes these homogeneous contributions from the main texture and evaluates surface albedos and reflection characteristics separately for surfaces with varying properties. This allows for dynamic surface variations without recomputing the entire volume.

### Heterogeneous Radiance Computation
This variant computes the total radiance for a ray by combining volumetric in-scattering, surface emission, and surface reflection, accounting for transmittance through the heterogeneous medium [S1, unofficial]. It explicitly handles the background case with an exponential decay based on average density and clamps the result [S1, unofficial].

### Perturbation Solution for High Albedo
This approach addresses the defects of standard approximations in scattering media with very high albedo (e.g., clouds) where shadowing causes artificial darkening [S2, unofficial]. It uses a perturbation solution expanding the radiance $Z$ into a power series in the single scattering albedo $f_l$: $Z = \sum f_l^k Z_k$ [S2, unofficial]. The equations for the $k$-th order are derived by equating like powers: $LI_k + MI_k = -MI_{k-1}$ [S2, unofficial]. This presents a series of forced conservative scattering equations, allowing approximation of the conservative case ($\omega_0 = 1$) [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Exponential with respect to recursion depth due to multiple scattering. Each bounce involves ray intersection and transmittance integration. The 16-step integration for transmittance adds a constant factor per segment.
*   **Memory Complexity:** Linear with respect to recursion depth for the call stack. No large textures are required for the base method, unlike precomputed approaches.
*   **Performance:** Slow for real-time applications due to recursive ray tracing and integration. Suitable for offline rendering or low-resolution previews.
*   **Strengths:** Handles heterogeneous media and multiple scattering accurately (within the depth limit). Captures indirect lighting effects from surfaces.
*   **Limitations:** The isotropic phase function approximation reduces accuracy. High albedo media require high recursion depth or perturbation solutions to avoid darkening artifacts.

## Background
The method is derived from the principles of light transport in participating media, specifically the Radiative Transfer Equation (RTE). The recursive approach for multiple scattering is a standard technique in path tracing for volumes. The specific implementation details (16-step integration, hardcoded isotropic phase) come from an unofficial repository [S1, unofficial]. The perturbation solution for high albedo media originates from Blinn (1982) and is discussed in the context of ray tracing for volume densities [S2, unofficial].

## Sources
[S1] ainagutierrez01/Volumetric-Path-Tracing-, https://github.com/ainagutierrez01/Volumetric-Path-Tracing-, fetched 2026-10-26 (unofficial)
[S2] Ray Tracing for Volume Densities, https://www.scribd.com/document/746685923/ray-tracking-volume-densities, fetched 2026-10-25 (unofficial)
