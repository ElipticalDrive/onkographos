# Heterogeneous In-scattering with Multiple Scattering


## Overview
This method computes the radiance contribution from light scattering within a heterogeneous volume, handling direct lighting from light sources and recursively computing indirect lighting (multiple scattering) up to a maximum depth [S1, unofficial]. It distinguishes itself from homogeneous approximations by evaluating local extinction and scattering coefficients at each integration step, allowing for spatially varying density and material properties [S1, unofficial]. Unlike surface-only albedo corrections, it operates directly on the volumetric light transport integral to capture internal light transport [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Scene geometry, light source positions, volume density field (heterogeneous), base absorption coefficient ($\mu_a$), base scattering coefficient ($\mu_s$), maximum recursion depth ($MaxDepth$).
*   **Output:** Radiance $L$ for a given ray segment or surface intersection.

**Data Structures:**
*   **Volume Coefficients:** Local absorption $\mu_a(\mathbf{x})$ and scattering $\mu_s(\mathbf{x})$ retrieved from the heterogeneous volume field at position $\mathbf{x}$.
*   **Total Extinction:** $\mu_t(\mathbf{x}) = \mu_a(\mathbf{x}) + \mu_s(\mathbf{x})$.

**Parameters:**
*   $\mu_a$: Base absorption coefficient, typically $0.05$ [S1, unofficial].
*   $\mu_s$: Base scattering coefficient, typically $0.1$ [S1, unofficial].
*   $MaxDepth$: Maximum recursion depth for multiple scattering, typically $5$ [S1, unofficial].

**Algorithm Steps:**

1.  **Evaluate Local Extinction:** At the current position $\mathbf{x}$, retrieve local $\mu_a$ and $\mu_s$. Calculate total extinction $\mu_t = \mu_a + \mu_s$.
2.  **Check Interaction Probability:** If $\mu_t < 0.0001$, return $0$ (no interaction) [S1, unofficial].
3.  **Calculate Single Scattering Albedo:** Compute the probability of scattering vs. absorption:
    $$ p_{scatter} = \frac{\mu_s}{\mu_t} $$
4.  **Compute Direct In-scattering:** For each light source:
    a. Sample the light position $\mathbf{x}_L$.
    b. Cast a shadow ray from the current volume position $\mathbf{x}$ to $\mathbf{x}_L$.
    c. If the ray is not occluded by geometry, compute the transmittance $T_{to\_light}$ from $\mathbf{x}$ to $\mathbf{x}_L$ using 16-step integration through the heterogeneous volume [S1, unofficial].
    d. Accumulate the direct contribution:
    $$ L_{direct} += L_{emitted} \cdot p(\omega_i, \omega_o) \cdot G \cdot T_{to\_light} \cdot A_{light} $$
    Where $L_{emitted}$ is the light radiance, $p$ is the phase function, $G$ is the geometric term, and $A_{light}$ is the light area [S1, unofficial].
5.  **Compute Indirect In-scattering (Recursive):** If current recursion depth $< MaxDepth$:
    a. Sample a hemisphere direction $\omega$ using a HemisphericalSampler [S1, unofficial].
    b. Cast a ray from the scatter position $\mathbf{x}_{scatter}$ in direction $\omega$.
    c. If the ray hits an object at distance $d$:
        i. Compute the transmittance $T_{to\_scatter}$ from $\mathbf{x}$ to $\mathbf{x}_{scatter}$ [S1, unofficial].
        ii. Recursively call `computeInscattering` for the segment up to the surface to get $L_{indirect}$ [S1, unofficial].
        iii. Add to the total in-scattering:
        $$ L_{indirect\_contrib} = L_{indirect} \cdot \bar{a} \cdot T_{to\_scatter} \cdot \frac{1}{4\pi} $$
        Where $\bar{a}$ is the surface albedo (or mean surface albedo if using correction textures) [S1, unofficial].
    d. If the ray does not hit an object (hits background), return background radiance [S1, unofficial].
6.  **Combine Contributions:** Sum direct and indirect contributions, modulated by the transmittance from the ray origin to the current point.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_inscattering(ray_origin, ray_direction, depth, max_depth, mu_a_base, mu_s_base):
    """
    Computes in-scattering for a ray segment in a heterogeneous volume.
    
    Args:
        ray_origin (Vector3): Origin of the ray.
        ray_direction (Vector3): Direction of the ray.
        depth (int): Current recursion depth.
        max_depth (int): Maximum recursion depth.
        mu_a_base (float): Base absorption coefficient.
        mu_s_base (float): Base scattering coefficient.
        
    Returns:
        Vector3: Radiance contribution from in-scattering.
    """
    # Step 1 & 2: Evaluate local extinction and check for interaction
    # Note: In a real implementation, this would involve stepping along the ray
    # and sampling the heterogeneous volume field. For this simplified recursive
    # model based on the notes, we assume a representative point or segment.
    
    # Retrieve local coefficients from heterogeneous volume at current position
    local_mu_a = get_volume_coefficient(ray_origin, 'absorption', mu_a_base)
    local_mu_s = get_volume_coefficient(ray_origin, 'scattering', mu_s_base)
    local_mu_t = local_mu_a + local_mu_s
    
    if local_mu_t < 0.0001:
        return Vector3(0, 0, 0)
    
    # Step 3: Calculate single scattering albedo
    p_scatter = local_mu_s / local_mu_t
    
    L_inscatter = Vector3(0, 0, 0)
    
    # Step 4: Direct In-scattering
    for light in scene.lights:
        # Sample light position
        light_pos = light.sample_position()
        
        # Create shadow ray
        shadow_ray = Ray(ray_origin, light_pos - ray_origin)
        
        # Check occlusion
        if not scene.intersects(shadow_ray):
            # Compute transmittance using 16-step integration
            T_to_light = integrate_transmittance(ray_origin, light_pos, steps=16)
            
            # Compute geometric term and phase function
            # Note: The source uses a hardcoded isotropic phase function
            phase = 1.0 / (4.0 * math.pi)
            geometric_term = compute_geometric_term(ray_origin, light_pos)
            
            # Accumulate contribution
            # L_emitted * phase * geometric * T_to_light * light_area
            contribution = light.emitted_radiance * phase * geometric_term * T_to_light * light.area
            L_inscatter += contribution
            
    # Step 5: Indirect In-scattering (Recursive)
    if depth < max_depth:
        # Sample hemisphere direction
        omega = hemispherical_sampler.sample(ray_origin.normal)
        
        # Cast ray from scatter position
        indirect_ray = Ray(ray_origin, omega)
        hit = scene.intersect(indirect_ray)
        
        if hit is not None:
            # Compute transmittance to the hit point
            T_to_scatter = integrate_transmittance(ray_origin, hit.point, steps=16)
            
            # Recursively compute in-scattering for the segment up to the surface
            # This handles the indirect lighting bouncing off surfaces
            L_indirect = compute_inscattering(hit.point, hit.normal, depth + 1, max_depth, mu_a_base, mu_s_base)
            
            # Add to L_inscatter
            # L_indirect * albedo * T_to_scatter * (1/(4*pi))
            # Note: Albedo is assumed to be retrieved from the surface material
            albedo = hit.material.get_albedo()
            indirect_contrib = L_indirect * albedo * T_to_scatter * (1.0 / (4.0 * math.pi))
            L_inscatter += indirect_contrib
        else:
            # Hit background
            # Return background radiance
            L_inscatter += get_background_radiance()
            
    return L_inscatter

def get_volume_coefficient(position, type, base_value):
    """
    Retrieves the local coefficient from the heterogeneous volume field.
    In a real implementation, this would sample a 3D texture or density field.
    For this example, we assume a simple perturbation or constant value if not specified.
    [G] Placeholder for actual volume sampling logic.
    """
    # [G] In a real heterogeneous volume renderer, this would sample a 3D texture
    # or density field at 'position'. Here, we return the base value as a placeholder
    # since the specific heterogeneous field data structure is not detailed in the sources.
    return base_value

def integrate_transmittance(start, end, steps):
    """
    Computes transmittance between two points using numerical integration.
    """
    # [G] Placeholder for 16-step integration logic.
    # Typically involves stepping along the line segment, sampling mu_t at each step,
    # and computing exp(-sum(mu_t * delta_s)).
    # Since the exact integration routine is not provided, we return 1.0 as a placeholder.
    return 1.0

def compute_geometric_term(p1, p2):
    """
    Computes the geometric term between two points.
    """
    # [G] Placeholder for geometric term calculation.
    # Typically involves distance and dot products with normals if surfaces are involved.
    # For volume-to-light, it often simplifies to 1/d^2 or similar.
    return 1.0

def get_background_radiance():
    """
    Returns the background radiance.
    """
    # [G] Placeholder for background radiance.
    return Vector3(0.1, 0.1, 0.1)
```

**Edge Cases and Pitfalls:**
*   **Isotropic Phase Function:** The implementation uses a hardcoded isotropic phase function ($1/(4\pi)$) for both direct and indirect scattering contributions, ignoring the specific phase function of the material [S1, unofficial]. This is a significant simplification that may lead to inaccurate scattering angles.
*   **Precision:** The check $\mu_t < 0.0001$ is a hardcoded threshold to avoid division by zero or numerical instability in very sparse volumes [S1, unofficial].
*   **Recursion Depth:** The recursion depth is capped at 5 [S1, unofficial]. Deeper volumes may require higher depths for accuracy, but this increases computational cost exponentially [S3].

## Subtopics

### [Correction Texture for Surface Albedo](9421-correction-texture-for-surface-albedo.md)
This method corrects the main scattering texture for light reflected off the ground, which is precomputed assuming a homogeneous mean surface albedo $\bar{a}$. It allows the renderer to remove these homogeneous contributions from the main texture and evaluate surface albedos and reflection characteristics separately at runtime for surfaces with varying properties.

### [Bias Compensation for Instant Radiosity](9586-bias-compensation-for-instant-radiosity.md)
Bias Compensation for Instant Radiosity is an unbiased method to handle weak singularities (infinite variance) in Instant Radiosity when volume points are close to light sources, by bounding the geometric term and recursively compensating for the introduced bias. It sets apart from standard Instant Radiosity by eliminating the bias introduced by clamping the geometric term, while distinguishing from [Correction Texture for Surface Albedo](9421-correction-texture-for-surface-albedo.md) by operating directly on the volumetric light transport integral rather than surface albedo corrections.

### Heterogeneous Radiance Computation
This variant computes the total radiance for a ray by combining volumetric in-scattering, surface emission, and surface reflection, accounting for transmittance through the heterogeneous medium [S1, unofficial]. It finds the closest intersection with scene geometry; if no intersection, it returns background radiance: ambient + bgColor * exp(-(mu_a + mu_s) * avg_density * 10.0), clamped to min 0.2 [S1, unofficial]. It then computes in-scattering contribution $L_{inscatter}$ for the segment, adds surface emission ($its.shape->getMaterial().getEmissiveRadiance() * T$), and if depth < MaxDepth, computes reflected radiance (direct + indirect) and adds to $L_o$ multiplied by $T$ [S1, unofficial].

### Perturbation Solution for High Albedo
This method addresses defects in approximation when modeling clouds (a scattering medium of very high albedo) by expanding the solution into a power series in $fl$: $Z = \sum fl^k Z_k$ [S2, unofficial]. It uses the relation $LI_k + MI_k = -MI_{k-1}$ to solve a series of forced conservative ($w_0 = 1$) scattering equations, approximating the solutions for the conservative case [S2, unofficial].

### Multiple Scattering via Volume Trace Depth
This is a render setting that enables volumetric multiple scattering, required for dense volumes to appear luminous and believable [S3]. It works exclusively with Brute Force Global Illumination [S3]. The "Volume Trace Depth" parameter controls the number of scattering bounces, with exponentially growing render cost [S3]. Forward scattering clouds benefit enormously from more depth; isotropic smoke shows diminishing returns earlier; emissive fire gains internal light spill [S3]. A pitfall is that Brute Force Global Illumination is a strict requirement for volumetric multiple scattering to function [S3].

## Operating characteristics
*   **Time Complexity:** Exponentially growing with recursion depth [S3]. Each bounce requires additional ray casts and volume integrations.
*   **Memory Complexity:** Depends on the storage of the heterogeneous volume field and the recursion stack.
*   **Performance:** Raising the trace depth transforms a cloud from a grey mass into a luminous, believable object, but at exponentially growing render cost [S3].
*   **Strengths:** Captures complex light transport in heterogeneous media, including internal light spill and multiple bounces [S3].
*   **Limitations:** Computationally expensive, especially for dense volumes [S3]. The use of an isotropic phase function may reduce accuracy [S1, unofficial].
*   **When to Use:** When realistic light transport in dense, heterogeneous volumes is required, and computational resources allow for high trace depths [S3].
*   **When Not to Use:** For real-time applications or when performance is critical and approximate results are acceptable. Also not suitable if the specific phase function of the material is important and cannot be approximated by isotropic scattering [S1, unofficial].

## Background
The method originates from volumetric path tracing techniques for participating media [S1, unofficial]. The concept of multiple scattering in dense volumes is highlighted in practical rendering guides, such as those for Redshift, which emphasize the role of "Volume Trace Depth" in achieving believable results [S3]. The perturbation solution for high albedo media dates back to early ray tracing research for volume densities [S2, unofficial].

## Sources
[S1] ainagutierrez01/Volumetric-Path-Tracing-, https://github.com/ainagutierrez01/Volumetric-Path-Tracing-, fetched 2026-10-26 (unofficial)
[S2] Ray Tracing for Volume Densities, https://www.scribd.com/document/746685923/ray-tracking-volume-densities, fetched 2026-10-25 (unofficial)
[S3] Introduction to Volumetric Rendering, https://help.maxon.net/c4d/2027/en-us/Content/html/What_is_Volumetric_Rendering.html, fetched 2026-10-29
