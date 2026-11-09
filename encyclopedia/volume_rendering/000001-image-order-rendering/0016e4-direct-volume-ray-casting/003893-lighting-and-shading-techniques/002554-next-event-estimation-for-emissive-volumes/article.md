# Next Event Estimation for Emissive Volumes

Also called: Emissive Media Rendering, Emissive Volume Rendering with Next Event Estimation (NEE), NEE for Emissive Volumes

## Overview
Next Event Estimation (NEE) for Emissive Volumes is a direct lighting estimation technique used in path tracing within participating media. It solves the problem of noisy or incorrect direct illumination from emissive volumes by explicitly sampling light sources and evaluating the emission along the ray segment between the surface hit point and the light source [S1, unofficial]. Unlike standard volume shading methods that may rely on ambient terms or approximate scattering, this method ensures energy consistency by correctly handling the probability density functions (PDFs) associated with sampling directions in 3D space [S1, unofficial]. It sets itself apart from siblings like [Direct Volume Rendering with Phong Illumination](3686-direct-volume-rendering-with-phong-shading.md) by explicitly accounting for the geometric attenuation and PDF conversion required when the light source is a volume rather than a surface, avoiding the pitfalls of naive solid-angle-to-volume PDF conversions [S1, unofficial].

## How it works
The method integrates NEE into the direct volume ray casting loop defined in [Direct volume ray casting](19-direct-volume-ray-casting.md). When a ray hits a surface or enters a volume, NEE estimates the direct lighting contribution from emissive volumes.

### Inputs and Outputs
*   **Input**: The current sample point $\mathbf{p}$ (hit point or volume sample), the view direction $\vec{\omega}_o$, the scene geometry including emissive volumes, and the light sources.
*   **Output**: The direct lighting term $L_{direct}(\mathbf{p})$ to be combined with the base color and other lighting terms.

### Data Structures
*   **Ray Segment**: Defined by entry point $\mathbf{p}_{entry}$ and exit point $\mathbf{p}_{exit}$ within the bounding volume of the emissive source.
*   **PDF Storage**: Requires tracking the solid angle PDF $p(\vec{\omega})$ and the derived volume PDF $p(\mathbf{x})$.

### Steps
1.  **Light Sampling**: For each light source (specifically emissive volumes), sample a point $\mathbf{x}_L$ on the light. Calculate the vector $\vec{L} = \mathbf{x}_L - \mathbf{p}$ and distance $r = \|\vec{L}\|$.
2.  **Shadow Ray**: Cast a shadow ray from $\mathbf{p}$ to $\mathbf{x}_L$ with near clip $0.001$ and far clip $r - 0.001$ [S2, unofficial]. Check for occlusion by intersecting with scene objects. If occluded, the direct lighting contribution from this light is zero [S2, unofficial].
3.  **PDF Conversion**: If not occluded, compute the MIS weight. The standard volume PDF $p(\mathbf{x})$ must be compared against the solid angle PDF $p(\vec{\omega})$ converted to volume space. The conversion factor involves $r^{-2}$ [S1, unofficial].
    *   The MIS weight is computed as $\mathrm{MIS}(r^{-2}p(\vec{\omega}), p(\mathbf{x}))$ [S1, unofficial].
    *   Note: $r^{-2}p(\vec{\omega})$ is a non-normalisable PDF because $\int_V r^{-2} p(\vec{\omega}) \mathrm{dV}$ does not converge to 1 in the standard manner [S1, unofficial].
4.  **Emission Evaluation**: Evaluate the emission along the entire ray segment between $\mathbf{p}$ and $\mathbf{x}_L$ to ensure consistency with the direction sampling used in the MIS calculation [S1, unofficial]. This involves integrating the emission coefficient over the path, accounting for transmittance.
5.  **Lighting Term Calculation**: Compute the direct lighting term using the evaluated emission, the geometric cosine factor, and the PDF weights.
    $$
    L_{direct} = \frac{f_r(\mathbf{p}, \vec{\omega}_o, \vec{\omega}_i) L_e(\mathbf{x}_L) \cos \theta}{p(\mathbf{x})} \cdot \text{MIS weight}
    $$
    Where $L_e$ is the emitted radiance, $\theta$ is the angle between the surface normal and light direction, and $f_r$ is the BSDF [S2, unofficial].
6.  **Accumulation**: Add $L_{direct}$ to the total lighting term $L(\mathbf{p})$ for the sample point.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def next_event_estimation_emissive_volume(p, wo, scene, depth):
    """
    Estimates direct lighting from emissive volumes using NEE.
    p: Hit point or volume sample position.
    wo: View direction.
    scene: Scene object containing lights and geometry.
    depth: Current path depth.
    """
    direct_light = RGB(0, 0, 0)
    
    if not scene.lights:
        return direct_light

    for light in scene.lights:
        # Sample point on light
        to_light = light.random_sample(p)
        distance_to_light = to_light.norm()
        if distance_to_light < 0.001:
            continue
            
        light_dir = to_light.normalize()
        
        # Shadow ray with specific intervals
        shadow_near = 0.001
        shadow_far = distance_to_light - 0.001
        shadow_ray = Ray(p + light_dir * shadow_near, light_dir, shadow_near, shadow_far)
        
        # Check occlusion
        hit = scene.intersect(shadow_ray)
        if hit is not None:
            continue # Occluded
            
        # Compute PDFs
        # p_omega is the solid angle PDF from light sampling
        p_omega = light.pdf(p)
        
        # Convert solid angle PDF to volume PDF for MIS
        # r^-2 * p(omega) is the non-normalisable volume PDF
        r = distance_to_light
        p_vol_converted = (1.0 / (r * r)) * p_omega
        
        # p_x is the standard volume PDF (if applicable, otherwise 1 or handled by MIS)
        # In many implementations, if sampling from light, p_x is effectively 1 for the direction
        # or handled via the MIS weight against the volume sampling strategy.
        # Here we assume p_x is the PDF of the current path strategy for this direction.
        p_x = 1.0 # Placeholder for path strategy PDF; adjust based on context
        
        # MIS Weight
        mis_weight = mis(p_vol_converted, p_x)
        
        # Evaluate emission along the ray segment
        # This ensures consistency with direction sampling [S1, unofficial]
        emission_integral = evaluate_emission_along_segment(p, p + to_light, scene)
        
        # Compute cosine term
        normal = get_normal(p)
        cosine = max(0.0, normal.dot(light_dir))
        
        # Get light emitted color
        light_color = light.emitted(p + to_light)
        
        # Attenuation factor (example from [S2, unofficial])
        attenuation = 1.0 # Or derived from material properties
        
        # Direct lighting formula [S2, unofficial]
        # Note: The formula in [S2, unofficial] uses attenuation/255.0 and divides by PI * pdf
        # Adapting to standard NEE form with MIS:
        term = (attenuation * light_color * cosine * emission_integral) / (math.pi * p_x)
        
        direct_light += term * mis_weight

    return direct_light

def evaluate_emission_along_segment(p_start, p_end, scene):
    """
    Integrates emission along the segment to ensure consistency [S1, unofficial].
    Returns the integrated emission value.
    """
    # Simplified integration: assume constant emission for demonstration
    # In practice, this requires ray marching through the volume
    direction = (p_end - p_start).normalize()
    distance = (p_end - p_start).norm()
    
    # Sample points along the segment
    num_samples = 10
    total_emission = RGB(0, 0, 0)
    
    for i in range(num_samples):
        t = (i + 0.5) / num_samples
        sample_point = p_start + direction * (t * distance)
        
        # Get emission coefficient at sample point
        emission_coeff = scene.get_emission_coefficient(sample_point)
        
        # Calculate transmittance from sample_point to p_end
        # This is a simplified approximation; full integration requires cumulative transmittance
        transmittance = scene.get_transmittance(sample_point, p_end)
        
        total_emission += emission_coeff * transmittance * (distance / num_samples)
        
    return total_emission

def mis(p1, p2):
    """
    Computes the balance heuristic MIS weight.
    """
    p1_sq = p1 * p1
    p2_sq = p2 * p2
    return p1_sq / (p1_sq + p2_sq)
```

### Parameters and Edge Cases
*   **Ray Intervals**: Shadow rays must use tight intervals (e.g., $0.001$ to $r-0.001$) to avoid self-intersection artifacts [S2, unofficial].
*   **PDF Normalization**: The term $r^{-2}p(\vec{\omega})$ is non-normalisable. Naive conversion using $r^{-2}\frac{\mathrm{dV}}{\mathrm{d\Omega}}$ yields incorrect results and is inconsistent with previous results without NEE [S1, unofficial].
*   **Performance**: Integrating the volume PDF over the ray length is significantly slower (20x-40x) due to required ray casts and raises OptiX warnings in CUDA modes [S1, unofficial].
*   **Mode Discrepancies**: In `llvm_*` modes, the implementation using NEE is approximately 5x slower than not using NEE. In `cuda_*` modes, it is approximately 1.5x slower [S1, unofficial].
*   **Intersection Backend**: Differences in ray intersection handling between Embree (llvm) and OptiX (cuda) can lead to different results for ray entry/exit points, potentially causing incorrect results in `llvm_*` modes if not carefully handled [S1, unofficial].

## Subtopics

### Path Tracing with Next Event Estimation
A general path tracing algorithm that includes direct lighting estimation via NEE. It recursively traces rays to compute pixel color. The base case returns black if depth is 0. It handles transmission materials by returning emitted light plus the recursive ray color scaled by transmittance. For opaque materials, it computes direct lighting by sampling a point on the light, checking for shadow, and calculating the contribution using the cosine term, light color, and PDF. Indirect lighting is computed recursively. This variant is a standard implementation of NEE in path tracers [S2, unofficial].

## Operating characteristics
*   **Time Complexity**: The method adds overhead due to shadow ray casting and emission integration along the segment. In `llvm_*` modes, it is ~5x slower than without NEE. In `cuda_*` modes, it is ~1.5x slower [S1, unofficial].
*   **Memory Complexity**: Requires storing ray intersection data and PDF values. No significant additional memory compared to standard path tracing.
*   **Performance**: Reduces noise for the same sample count by providing accurate direct lighting estimates [S1, unofficial].
*   **Strengths**: Ensures energy consistency and reduces noise in scenes with emissive volumes.
*   **Limitations**: Slower performance, especially in CPU-based renderers. Potential inconsistencies between CPU and GPU backends due to intersection differences [S1, unofficial].
*   **When to Use**: When accurate direct lighting from emissive volumes is required and noise reduction is a priority.
*   **When Not to Use**: In real-time applications where performance is critical and approximate lighting is acceptable.

## Background
The method was developed as part of the Mitsuba 3 renderer project, specifically in the context of implementing emissive volume rendering. It addresses the challenges of correctly sampling and integrating emission from volumes in a path tracing framework, ensuring consistency with Multiple Importance Sampling (MIS) [S1, unofficial]. The implementation details and discussions were documented in GitHub issues and pull requests related to the Mitsuba 3 repository [S1, unofficial].

## Sources
[S1] [✨ feature] Emissive Volume Rendering, https://github.com/mitsuba-renderer/mitsuba3/pull/678, fetched 2026-10-26 (unofficial)
[S2] TakTikFR/Volumetric-Path-Tracing, https://github.com/TakTikFR/Volumetric-Path-Tracing, fetched 2026-10-26 (unofficial)
