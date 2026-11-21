# Heterogeneous In-scattering with Multiple Scattering

Also called: Cloud Scattering Luminance, Cloud Scattering Luminance Scale

## Overview
This method computes the radiance contribution from light scattering within a heterogeneous volume, handling both direct lighting from light sources and recursively computing indirect lighting (multiple scattering) up to a maximum depth [S1, unofficial]. It is distinct from single-scattering approximations by explicitly simulating indirect light transport within the volume, transforming dense volumes from opaque grey masses into luminous, believable objects [S1, unofficial][S2]. Unlike the [Most Probable Paths (MPP) Lighting Model](12244-most-probable-paths-mpp-lighting-model.md) which focuses on specific path probabilities, or the [Riley et al. Angular Distribution Renderer](12249-riley-et-al-angular-distribution-renderer.md) which uses angular distributions, this approach uses recursive ray tracing with a fixed depth limit to approximate multiple bounces [S1, unofficial]. It is computationally more expensive than heuristic shortcuts but provides physically plausible indirect illumination [S1, unofficial][S2].

## How it works
The method operates by tracing rays through the volume, computing transmittance, and accumulating in-scattered radiance from direct and indirect sources. It relies on the radiative transfer equation principles described in [Parent](10687-multiple-scattering-models.md), specifically the scattering term $\sigma_s \int_{S^2} p(\omega', \omega) L(x, \omega') d\omega'$ [S3, unofficial].

### Inputs and Outputs
*   **Inputs:**
    *   Ray origin $O$ and direction $\omega$.
    *   Volume density field (heterogeneous $\sigma_t, \sigma_s, \sigma_a$).
    *   Scene geometry (for surface intersections and reflections).
    *   Light sources (position, intensity).
    *   Parameters: `mu_a` (base absorption), `mu_s` (base scattering), `max_depth` (recursion limit).
*   **Outputs:**
    *   Total radiance $L$ at the ray origin.

### Data Structures
*   **Volume Coefficients:** Heterogeneous fields for extinction $\sigma_t(x)$, scattering $\sigma_s(x)$, and absorption $\sigma_a(x)$.
*   **Ray:** Origin $O$, direction $\omega$, current distance $t$.
*   **Hit Record:** Intersection point $P$, surface normal $N$, material properties (albedo, emissive radiance).

### Parameters
*   `mu_a`: Base absorption coefficient. Typical value: $0.05$ [S1, unofficial].
*   `mu_s`: Base scattering coefficient. Typical value: $0.1$ [S1, unofficial].
*   `max_depth`: Maximum recursion depth for multiple scattering. Typical value: $5$ [S1, unofficial].
*   `Cloud Scattering Luminance Scale`: A property on the Atmosphere Light that scales the light contribution to compensate for energy loss when simulating multiple scattering via octave approximation [S2].

### Steps

1.  **Initial Check:** Calculate local extinction coefficient $\sigma_t$ at the ray origin. If $\sigma_t < 0.0001$, return $0$ (no interaction) [S1, unofficial].
2.  **Single Scattering Albedo:** Compute $\omega_0 = \frac{\sigma_s}{\sigma_t}$ [S1, unofficial].
3.  **Direct In-scattering:**
    *   For each light source:
        *   Sample light position $L_{pos}$.
        *   Create shadow ray from current point to $L_{pos}$.
        *   If not occluded, compute transmittance $T_{to\_light}$ using 16-step integration of $\sigma_t$ along the path [S1, unofficial].
        *   Accumulate direct contribution: $L_{direct} += L_{emitted} \cdot p(\omega_{in}, \omega_{out}) \cdot \frac{1}{r^2} \cdot T_{to\_light} \cdot A_{light}$ [S1, unofficial]. Note: The implementation uses a hardcoded isotropic phase function $p = \frac{1}{4\pi}$ [S1, unofficial].
4.  **Indirect In-scattering (Recursive):**
    *   If current `depth` < `max_depth`:
        *   Sample a hemisphere direction $\omega_{ind}$ using a HemisphericalSampler [S1, unofficial].
        *   Cast a ray from the scatter position in direction $\omega_{ind}$.
        *   If the ray hits an object:
            *   Compute transmittance $T_{to\_scatter}$ from the current point to the hit point.
            *   Recursively call `computeInscattering` for the segment up to the surface to get $L_{indirect}$ [S1, unofficial].
            *   Add to total in-scattering: $L_{indirect\_contrib} = L_{indirect} \cdot \text{albedo} \cdot T_{to\_scatter} \cdot \frac{1}{4\pi}$ [S1, unofficial].
5.  **Heterogeneous Radiance Computation:**
    *   Find the closest intersection with scene geometry along the main ray.
    *   If no intersection, return background radiance: $L_{bg} = L_{ambient} + L_{bgColor} \cdot \exp(-(\sigma_a + \sigma_s) \cdot \bar{\rho} \cdot 10.0)$, clamped to min $0.2$ [S1, unofficial].
    *   Compute in-scattering contribution $L_{inscatter}$ for the segment from origin to intersection [S1, unofficial].
    *   Add surface emission: $L_{emission} = L_{emissive} \cdot T_{surface}$ [S1, unofficial].
    *   If `depth` < `MaxDepth`, compute reflected radiance (direct + indirect) from the surface and add to $L_{out}$ multiplied by $T_{surface}$ [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_inscattering(ray_origin, ray_direction, depth, max_depth, mu_a, mu_s):
    """
    Computes in-scattered radiance for a ray in a heterogeneous volume.
    
    Args:
        ray_origin: Point3D, origin of the ray.
        ray_direction: Vector3D, direction of the ray.
        depth: int, current recursion depth.
        max_depth: int, maximum recursion depth.
        mu_a: float, base absorption coefficient.
        mu_s: float, base scattering coefficient.
        
    Returns:
        Vector3D, total in-scattered radiance.
    """
    # 1. Get local coefficients
    sigma_t = get_extinction(ray_origin)
    sigma_s = get_scattering(ray_origin)
    
    # 2. Check for no interaction
    if sigma_t < 0.0001:
        return Vector3D(0, 0, 0)
    
    # 3. Compute single scattering albedo
    omega_0 = sigma_s / sigma_t
    
    # 4. Direct In-scattering
    L_direct = Vector3D(0, 0, 0)
    for light in scene.lights:
        # Sample light position (simplified for point light)
        light_pos = light.position
        light_dir = normalize(light_pos - ray_origin)
        dist_to_light = length(light_pos - ray_origin)
        
        # Shadow ray
        if not is_occluded(ray_origin, light_dir, dist_to_light):
            # Transmittance using 16-step integration
            T_to_light = compute_transmittance_16_steps(ray_origin, light_dir, dist_to_light)
            
            # Isotropic phase function
            phase = 1.0 / (4.0 * PI)
            
            # Accumulate contribution
            # Note: geometric term 1/r^2 is included in light intensity or computed separately
            # Assuming light.emitted includes geometric falloff or is radiance
            L_direct += light.emitted * phase * T_to_light * light.area
            
    # 5. Indirect In-scattering
    L_indirect = Vector3D(0, 0, 0)
    if depth < max_depth:
        # Sample hemisphere direction
        omega_ind = sample_hemisphere(ray_origin, ray_direction) # Simplified sampling
        
        # Cast ray
        hit = cast_ray(ray_origin, omega_ind)
        if hit.is_valid:
            # Transmittance to scatter point
            T_to_scatter = compute_transmittance(ray_origin, hit.point)
            
            # Recursive call for indirect radiance at hit point
            # Note: The source says "recursively call computeInscattering for the segment up to the surface"
            # This implies computing the radiance arriving at the surface from the volume
            L_indirect_at_surface = compute_inscattering(ray_origin, omega_ind, depth + 1, max_depth, mu_a, mu_s)
            
            # Add to total
            # albedo is from hit.material
            albedo = hit.material.albedo
            L_indirect += L_indirect_at_surface * albedo * T_to_scatter * (1.0 / (4.0 * PI))
            
    return L_direct + L_indirect

def compute_heterogeneous_radiance(ray_origin, ray_direction, depth, max_depth, mu_a, mu_s):
    """
    Computes total radiance for a ray.
    """
    # 1. Find closest intersection
    hit = cast_ray(ray_origin, ray_direction)
    
    if not hit.is_valid:
        # No intersection, return background
        avg_density = get_average_density()
        T_bg = exp(-(mu_a + mu_s) * avg_density * 10.0)
        L_bg = scene.ambient + scene.bg_color * T_bg
        return max(L_bg, Vector3D(0.2, 0.2, 0.2))
    
    # 2. Compute in-scattering for the segment
    L_inscatter = compute_inscattering(ray_origin, ray_direction, depth, max_depth, mu_a, mu_s)
    
    # 3. Transmittance to surface
    T_surface = compute_transmittance(ray_origin, hit.point)
    
    # 4. Add surface emission
    L_emission = hit.material.emissive_radiance * T_surface
    
    # 5. Add reflected radiance
    L_reflected = Vector3D(0, 0, 0)
    if depth < max_depth:
        # Compute reflected radiance (direct + indirect)
        # This typically involves shading the hit point
        L_reflected = shade_surface(hit, depth + 1, max_depth, mu_a, mu_s) * T_surface
        
    return L_inscatter + L_emission + L_reflected
```

### Edge Cases and Pitfalls
*   **Isotropic Phase Function:** The implementation uses a hardcoded isotropic phase function ($1/4\pi$) for both direct and indirect scattering contributions, ignoring the material's actual phase function [S1, unofficial]. This is a significant simplification that may reduce accuracy for forward-scattering media.
*   **Energy Loss:** Simulating multiple scattering via octave approximation can result in energy loss, requiring compensation via the `Cloud Scattering Luminance Scale` parameter [S2].
*   **Precision:** The transmittance calculation uses 16-step integration, which may introduce errors in highly heterogeneous volumes if the step size is not adaptive [S1, unofficial].
*   **Recursion Depth:** The recursion stops at `depth >= 5`. Deeper volumes may require higher depths for accuracy, but this increases computational cost exponentially [S1, unofficial].

## Subtopics

### [Correction Texture for Surface Albedo](9421-correction-texture-for-surface-albedo.md)
This method corrects the main scattering texture for light reflected off the ground, which is precomputed assuming a homogeneous mean surface albedo $\bar{a}$. It allows the renderer to remove these homogeneous contributions from the main texture and evaluate surface albedos and reflection characteristics separately at runtime for surfaces with varying properties.

### [Bias Compensation for Instant Radiosity](9586-bias-compensation-for-instant-radiosity.md)
Bias Compensation for Instant Radiosity is an unbiased method to handle weak singularities (infinite variance) in Instant Radiosity when volume points are close to light sources, by bounding the geometric term and recursively compensating for the introduced bias.

### [Multiple Scattering via Volume Trace Depth](10698-multiple-scattering-via-volume-trace-depth.md)
This method enables volumetric multiple scattering by controlling the maximum number of scattering bounces computed within participating media, transforming dense volumes from opaque grey masses into luminous, believable objects. It is distinct from single-scattering approximations by explicitly simulating indirect light transport within the volume, but it is computationally more expensive and requires Brute Force Global Illumination, unlike faster heuristic shortcuts.

### Heterogeneous Radiance Computation
This method computes the total radiance for a ray by combining volumetric in-scattering, surface emission, and surface reflection, accounting for transmittance through the heterogeneous medium [S1, unofficial]. It finds the closest intersection with scene geometry, computes in-scattering for the segment, adds surface emission, and if depth allows, computes reflected radiance [S1, unofficial].

### Perturbation Solution for High Albedo
For media with very high albedo (close to 1), the perturbation solution is used. The radiance $Z$ is expanded into a power series in the single scattering albedo $fl$: $Z = \sum fl^k Z_k$ [S3, unofficial]. The equations for the $k$-th order are $LI_k + MI_k = -MI_{k-1}$, where $LI$ is the direct illumination term and $MI$ is the multiple scattering term [S3, unofficial]. This presents a series of forced conservative scattering equations [S3, unofficial].

### Multiple Scattering Approximation via Octaves
An approximation of realistic light scattering in participating media by tracing multiple octaves (steps) of light transmittance within the volume material to simulate the complex scattering effect where light is almost never absorbed (albedo close to 1) [S2].
*   **Parameters:**
    *   `Number of octaves`: Integer from 0 to 2. 0 is single scattering, 1 is one octave, 2 is two octaves. Higher octaves increase shader expense [S2].
    *   `Contribution`: Float parameter controlling the amount of multiple scattering contribution. High values with low Occlusion can mimic high octaves without performance impact [S2].
    *   `Occlusion`: Float parameter controlling the amount of occlusion in the multiple scattering approximation [S2].
    *   `Eccentricity`: Float parameter controlling the eccentricity of the multiple scattering approximation [S2].
*   **Comparison:** Single octave (1) is recommended for games for performance. High octaves (2) are more expensive. High Contribution/Low Occlusion can approximate high octaves with no performance impact [S2].

## Operating characteristics
*   **Time Complexity:** Exponential with respect to recursion depth due to the branching of indirect scattering rays [S1, unofficial]. The use of 16-step integration for transmittance adds a constant factor per light source per bounce [S1, unofficial].
*   **Memory Complexity:** Linear with respect to the number of light sources and the depth of the recursion stack [S1, unofficial].
*   **Measured Performance:** Not given in the sources. However, it is noted that higher octaves and deeper recursion increase shader expense [S2].
*   **Scaling:** Performance degrades significantly with increased volume density and number of light sources [S1, unofficial].
*   **Strengths:** Physically plausible indirect illumination; handles heterogeneous volumes; compensates for energy loss with luminance scaling [S1, unofficial][S2].
*   **Limitations:** Computationally expensive; uses isotropic phase function by default; requires careful tuning of parameters (depth, octaves, luminance scale) [S1, unofficial][S2].
*   **When to use:** When physically accurate multiple scattering is required and performance allows; for cloud rendering where energy loss compensation is needed [S1, unofficial][S2].
*   **When not to use:** When real-time performance is critical and single scattering approximations are sufficient; when the phase function is highly anisotropic and the isotropic approximation is unacceptable [S1, unofficial].

## Background
The method originates from volumetric path tracing techniques for participating media [S1, unofficial]. The concept of using octave approximation for multiple scattering in real-time rendering is documented in Unreal Engine's Volumetric Cloud Component [S2]. The perturbation solution for high albedo media is based on classical radiative transfer theory, as described in [S3, unofficial].

## Sources
[S1] ainagutierrez01/Volumetric-Path-Tracing-, https://github.com/ainagutierrez01/Volumetric-Path-Tracing-, fetched 2026-10-26 (unofficial)
[S2] Volumetric Cloud Component in Unreal Engine  | Unreal Engine 5.8 Documentation | Epic Developer Community, https://dev.epicgames.com/documentation/en-us/unreal-engine/volumetric-cloud-component-in-unreal-engine, fetched 2026-10-30
[S3] Ray Tracing for Volume Densities, https://www.scribd.com/document/746685923/ray-tracking-volume-densities, fetched 2026-10-25 (unofficial)
