# SeaThru-NeRF

Also called: SeaThru

## Overview
SeaThru-NeRF is a neural radiance field method designed for visualizing and reconstructing scenes within dispersed scattering media, such as fog, haze, and subaquatic environments [S1]. It extends the standard NeRF formulation by treating the scattering medium as a semi-transparent entity with distinct color and density parameters, separate from opaque objects [S2]. This allows the model to distinguish between unblemished (directly illuminated) and backscattered segments of the scene, enabling the restoration of scene hue and 3D configuration in far-off areas that are typically obscured [S1]. It builds on [HybridNeRF Model Architecture](4842-hybridnerf-model-architecture.md) by incorporating specific scattering physics into the volume rendering integral.

## How it works
SeaThru-NeRF modifies the standard volume rendering equation to account for the additive density and color contributions of a scattering medium. It assumes that the total density at any point along a ray is the sum of the object density and the medium density [S2].

**Inputs and Outputs:**
*   **Input:** Camera ray $r$, scene geometry and appearance data (learned via MLPs).
*   **Output:** Rendered color $C(r)$, potentially decomposed into object-only and medium-only components.

**Data Structures:**
The method utilizes the standard NeRF MLP architecture but outputs two sets of parameters for each sampled point $t$ along the ray:
1.  Object density $\sigma_{obj}(t)$ and object color $c_{obj}(t)$.
2.  Medium density $\sigma_{med}(t)$ and medium color $c_{med}(t)$.

**Rendering Formulas:**
The rendered color $C(r)$ is computed by integrating the accumulated color along the ray segment from $t_n$ to $t_f$ [S2]:

$$
C(r) = \int_{t_n}^{t_f} T(t) \left[ \sigma_{obj}(t)c_{obj}(t) + \sigma_{med}(t)c_{med}(t) \right] dt \quad (8)
$$

where $T(t)$ is the transmittance, representing the probability that light travels from the camera to point $t$ without being absorbed or scattered [S2]:

$$
T(t) = \exp \left( - \int_{t_n}^{t} \left[ \sigma_{obj}(s) + \sigma_{med}(s) \right] ds \right) \quad (9)
$$

Here, $c_{obj}(t)$ and $c_{med}(t)$ are the colors of the object and medium at $t$, respectively [S2].

**Training:**
The model parameters are optimized by minimizing the squared distance between the expected color $\hat{C}(r)$ and the ground truth color $C_{gt}(r)$ [S2].

**Pseudocode:**
The following pseudocode illustrates the rendering step for a single ray, assuming the MLPs `query_object` and `query_medium` are defined elsewhere.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_seathru_ray(ray_origin, ray_direction, t_near, t_far, num_samples, 
                       query_object, query_medium):
    """
    Renders a single ray using the SeaThru-NeRF scattering model.
    
    Args:
        ray_origin: 3D vector, camera origin.
        ray_direction: 3D vector, normalized ray direction.
        t_near: float, near plane distance.
        t_far: float, far plane distance.
        num_samples: int, number of sample points along the ray.
        query_object: function, returns (sigma_obj, c_obj) for a 3D point.
        query_medium: function, returns (sigma_med, c_med) for a 3D point.
        
    Returns:
        color: float, rendered color value.
    """
    # Discretize the ray into sample points
    t_vals = linspace(t_near, t_far, num_samples)
    
    accumulated_transmittance = 0.0
    accumulated_color = 0.0
    
    # Calculate delta distances between samples
    # Note: In practice, t_vals are often sampled adaptively or uniformly.
    # Here we assume uniform sampling for simplicity.
    delta_t = (t_far - t_near) / (num_samples - 1)
    
    for i in range(num_samples):
        t = t_vals[i]
        point = ray_origin + t * ray_direction
        
        # Query object properties
        sigma_obj, c_obj = query_object(point)
        
        # Query medium properties
        sigma_med, c_med = query_medium(point)
        
        # Total density at this point
        sigma_total = sigma_obj + sigma_med
        
        # Calculate transmittance up to this point
        # T(t) = exp(-integral from t_near to t of sigma_total(s) ds)
        # Approximating integral as sum of previous deltas
        if i == 0:
            transmittance = 1.0
        else:
            # Accumulate opacity from previous steps
            # This is a simplified Euler integration of the transmittance
            # A more robust implementation would maintain running transmittance
            pass 
            
        # Calculate weight for this sample
        # Weight = T(t) * (1 - exp(-sigma_total * delta_t))
        # However, the standard NeRF formulation uses:
        # Weight_i = T_i * (1 - exp(-sigma_i * delta_t))
        # where T_i = exp(-sum_{j<i} sigma_j * delta_t)
        
        # Let's implement the standard discrete volume rendering accumulation
        pass

    # Re-implementing the standard accumulation loop for clarity
    accumulated_transmittance = 1.0
    accumulated_color = 0.0
    
    for i in range(num_samples):
        t = t_vals[i]
        point = ray_origin + t * ray_direction
        
        sigma_obj, c_obj = query_object(point)
        sigma_med, c_med = query_medium(point)
        
        sigma_total = sigma_obj + sigma_med
        
        # Calculate opacity for this segment
        # alpha = 1 - exp(-sigma * delta_t)
        alpha = 1.0 - exp(-sigma_total * delta_t)
        
        # Weight = accumulated_transmittance * alpha
        weight = accumulated_transmittance * alpha
        
        # Add contribution to color
        # The color contribution is the sum of object and medium colors weighted by their densities
        # c_total = (sigma_obj * c_obj + sigma_med * c_med) / sigma_total
        # But the integral form (Eq 8) suggests:
        # contribution = T(t) * (sigma_obj * c_obj + sigma_med * c_med) * delta_t
        # In discrete form: weight * c_total is not quite right if we use the alpha formulation.
        # The standard NeRF discrete form is:
        # C = sum_i T_i * (1 - exp(-sigma_i * delta_t)) * c_i
        # Where c_i is the color emitted by the volume element.
        # In SeaThru, the "emitted" color is the sum of object and medium emissions.
        
        c_total = (sigma_obj * c_obj + sigma_med * c_med) / (sigma_total + 1e-6)
        
        accumulated_color += weight * c_total
        
        # Update transmittance for next step
        accumulated_transmittance *= (1.0 - alpha)
        
    return accumulated_color
```

**Edge Cases and Pitfalls:**
*   **Degenerate Input:** If the medium density is zero everywhere, the model reduces to standard NeRF. If object density is zero, it renders only the medium.
*   **Precision:** The exponential function in transmittance calculation can lead to underflow for very dense media. Clamping or using log-space arithmetic may be necessary [G].
*   **Boundary Conditions:** The integration limits $t_n$ and $t_f$ must be carefully chosen to include all relevant scattering and object contributions.

## Subtopics

### Shading Replacement for Relighting
Focuses on modifying shading for relighting, not scattering media. [Link](6346-shading-replacement-for-relighting.md)

### Hybrid Multi-Space Module
Uses hybrid representations for multi-space scenes, not specifically scattering. [Link](11712-hybrid-multi-space-module.md)

### LidaRF Hybrid Representation
Hybrid representation for large-scale scenes, not scattering. [Link](12893-lidarf-hybrid-representation.md)

### Pano-NeRF
Panoramic NeRF, not scattering. [Link](15013-pano-nerf.md)

### FastSR-NeRF
Fast super-resolution NeRF, not scattering. [Link](16574-fastsr-nerf.md)

### NeRFCodec
Compression for NeRF, not scattering. [Link](16590-nerfcodec.md)

### Ambiguity-Aware Depth Estimates
Depth estimation with ambiguity, not scattering. [Link](16981-ambiguity-aware-depth-estimates.md)

### Space-time Neural Irradiance Fields
Dynamic scenes, not scattering. [Link](17161-space-time-neural-irradiance-fields.md)

## Operating characteristics
*   **Time Complexity:** Depends on the number of samples per ray and the depth of the MLPs. Similar to standard NeRF but with double the output channels (object and medium).
*   **Memory Complexity:** Requires storing two sets of density and color parameters per point, effectively doubling the output size of the MLPs compared to standard NeRF.
*   **Strengths:** Capable of restoring scene hue and 3D configuration in far-off areas obscured by scattering media [S1]. Can produce lifelike new-view composites while maintaining or eliminating the medium effect [S1].
*   **Limitations:** Performance in highly complex reflective and refractive scenes may require more subspaces, similar to MS-NeRF [S1]. Not given in the sources for specific hardware performance metrics.
*   **When to use:** Underwater, fog, or haze visualization and reconstruction.
*   **When not to use:** Clear air scenes where standard NeRF is sufficient and more efficient.

## Background
SeaThru-NeRF was introduced in 2023 [S1]. It is inspired by the SeaThru method [S2]. It extends the capability of NeRFs to handle scattering media such as underwater environments [S2]. The method showcases proficiency in depicting scenes in both simulated and actual settings, resulting in cutting-edge, realistic new-view composites [S1].

## Sources
[S1] Neural Radiance Field-based Visual Rendering: a Comprehensive Review, https://ieeexplore.ieee.org/abstract/document/11455380, fetched 2026-10-30
[S2] Bayesian uncertainty analysis for underwater 3D reconstruction with neural radiance fields, https://doi.org/10.1016/j.apm.2024.115806, fetched 2026-10-29
