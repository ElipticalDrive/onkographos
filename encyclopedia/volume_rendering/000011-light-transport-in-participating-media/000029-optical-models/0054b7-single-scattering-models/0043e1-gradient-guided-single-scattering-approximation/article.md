# Gradient-guided single scattering approximation

Also called: GPU Based On-the-fly Light Emission-absorption Approximation, On-the-fly scatter approximation

## Overview
A GPU-based direct volume rendering method that approximates single scattering and shadowing on-the-fly for multiple intersecting volumetric datasets [S1, unofficial]. It improves visual quality over local illumination models by using local gradients to guide scatter sample positions and in-scatter coefficients, while maintaining interactive frame rates [S1, unofficial]. It differs from pre-computed ambient occlusion methods by requiring no parameter buffers and performing all computations on-the-fly [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** Multiple registered volumetric datasets (e.g., $256^3$ grids), transfer functions defining opacity/scattering ratios, and a view ray.
*   **Output:** Final radiance $L_o$ for each pixel.

**Data Structures**
*   **Voxel Grid:** Standard 3D texture or volume grid storing density/extinction coefficients.
*   **Gradient Field:** Not pre-computed; approximated on-the-fly from the voxel grid neighborhood [S1, unofficial].

**Parameters**
*   $g$: Average cosine of the scattering angle, ranging from $-1$ to $1$ [S1, unofficial].
*   $N_{max}$: Maximum number of scatter sample positions, set to 15 [S1, unofficial].

**Algorithm Steps**
1.  **Ray Marching:** Cast a ray through the volume. For each step position $\mathbf{p}$:
2.  **Gradient Approximation:** Compute the gradient $\nabla V(\mathbf{p})$ of the volume data at the current sampling point [S1, unofficial].
3.  **Determine Scatter Samples:**
    *   Select up to $N_{max} = 15$ scatter sample positions $\mathbf{s}_i$ in the local neighborhood of $\mathbf{p}$ [S1, unofficial].
    *   The selection and weighting depend on the strength of the gradient at $\mathbf{p}$ [S1, unofficial].
    *   Assume a strong gradient indicates a less scattering, more reflecting surface, and that the gradient points in the direction of less dense, more scattering media [S1, unofficial].
4.  **Calculate Scattering Angle:**
    *   For each scatter sample $\mathbf{s}_i$, compute its gradient $\nabla V(\mathbf{s}_i)$ [S1, unofficial].
    *   Calculate the cosine of the scattering angle $\theta$ between the gradient at the current point $\nabla V(\mathbf{p})$ and the gradient at the scatter sample $\nabla V(\mathbf{s}_i)$ [S1, unofficial].
    *   The cosine is zero only if the direction of the scatter sample gradient is identical to the current sampling point’s gradient vector [S1, unofficial].
5.  **Weighting:**
    *   Weight the influence of each scatter sample position by:
        1.  The distance to the actual sampling point $\mathbf{p}$ [S1, unofficial].
        2.  The gradient magnitude at the given scatter sampling position $\|\nabla V(\mathbf{s}_i)\|$ [S1, unofficial].
6.  **Phase Function:**
    *   Compute the phase function $p(\theta)$ using the Henyey-Greenstein function [S1, unofficial]:
        $$ p(\theta) = \frac{1-g^2}{4\pi (1+g^2 +2g \cos\theta)^{3/2}} $$
    *   Where $g < 0$ indicates backward scattering, $g > 0$ indicates forward scattering, and $g = 0$ results in isotropic scattering [S1, unofficial].
7.  **Light Transfer:**
    *   Calculate light transfer through the volumes using standard lighting absorption calculation as described in [Max95] [S1, unofficial].
    *   Combine the in-scattered light (weighted by $p(\theta)$ and sample weights) with the direct illumination and absorption [S1, unofficial].
8.  **Compositing:** Accumulate the result along the ray.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gradient_guided_single_scattering(ray_origin, ray_direction, volume_data, g_param):
    """
    Approximates single scattering for a single ray.
    volume_data: 3D array of voxel values.
    g_param: Phase function parameter (average cosine of scattering angle).
    """
    total_transmittance = 1.0
    accumulated_radiance = 0.0
    
    # Standard ray marching parameters
    step_size = 0.1
    max_steps = 1000
    
    for step in range(max_steps):
        current_pos = ray_origin + ray_direction * step * step_size
        current_val = trilinear_interpolate(volume_data, current_pos)
        
        # 1. Gradient Approximation at current position
        grad_current = compute_gradient(volume_data, current_pos)
        
        # 2. Determine scatter samples (up to 15)
        # Note: Exact selection logic for "up to 15" based on gradient strength 
        # is not fully specified in sources beyond "depending on the strength" [S1, unofficial].
        # We assume a fixed set of neighbors or a dynamic selection based on gradient magnitude.
        scatter_samples = select_scatter_samples(current_pos, grad_current, volume_data)
        
        in_scatter = 0.0
        
        for sample_pos in scatter_samples:
            # 3. Gradient at scatter sample
            grad_sample = compute_gradient(volume_data, sample_pos)
            
            # 4. Scattering Angle Cosine
            # Cosine is zero if directions are identical (parallel gradients)
            # Note: The source states "cosine ... is zero only if ... identical". 
            # This implies the angle is between gradient vectors, not view directions.
            cos_theta = dot_product(normalize(grad_current), normalize(grad_sample))
            
            # 5. Phase Function
            phase = heney_greenstein(cos_theta, g_param)
            
            # 6. Weighting
            dist = distance(current_pos, sample_pos)
            grad_mag_sample = magnitude(grad_sample)
            weight = (1.0 / (dist + epsilon)) * grad_mag_sample
            
            # 7. Light Transfer (Simplified Absorption)
            # Using standard absorption calculation from Max95 [S1, unofficial]
            # Assume sigma_t is derived from volume_data or transfer function
            sigma_t = get_extinction(volume_data, sample_pos)
            transmittance_to_sample = exp(-sigma_t * dist)
            
            # In-scatter contribution
            # Source implies in-scatter amount is defined by the angle and weights
            in_scatter += phase * weight * transmittance_to_sample
            
        # 8. Accumulate
        # Standard emission-absorption model update [S1, unofficial]
        # L_o = L_i * T + L_e * (1 - T) + InScatter
        # Here, InScatter is added to the radiance before absorption of the next step
        accumulated_radiance += in_scatter * total_transmittance
        
        # Update transmittance for the next step
        sigma_t_current = get_extinction(volume_data, current_pos)
        total_transmittance *= exp(-sigma_t_current * step_size)
        
        if total_transmittance < epsilon:
            break
            
    return accumulated_radiance

def heney_greenstein(cos_theta, g):
    """
    Henyey-Greenstein phase function.
    """
    numerator = 1.0 - g**2
    denominator = 4.0 * pi * (1.0 + g**2 + 2.0 * g * cos_theta)**1.5
    return numerator / denominator

def compute_gradient(volume_data, pos):
    """
    Approximate gradient using central differences or finite differences.
    """
    # Implementation detail [G]: Standard finite difference gradient calculation
    dx = volume_data[pos.x+1, pos.y, pos.z] - volume_data[pos.x-1, pos.y, pos.z]
    dy = volume_data[pos.x, pos.y+1, pos.z] - volume_data[pos.x, pos.y-1, pos.z]
    dz = volume_data[pos.x, pos.y, pos.z+1] - volume_data[pos.x, pos.y, pos.z-1]
    return Vector3(dx, dy, dz)

def select_scatter_samples(current_pos, grad_current, volume_data):
    """
    Select up to 15 scatter sample positions based on gradient strength.
    Note: Exact algorithm for selection is not detailed in sources [S1, unofficial].
    We assume a heuristic based on gradient magnitude.
    """
    # Placeholder for specific selection logic not provided in [S1, unofficial]
    # Typically involves sampling neighbors in the direction of the gradient
    samples = []
    # ... implementation depends on specific neighbor sampling strategy ...
    return samples
```

## Operating characteristics
*   **Time Complexity:** Interactive. Performance depends on the number of scatter samples (up to 15) and the cost of gradient computation per voxel [S1, unofficial].
*   **Memory Complexity:** Low. No pre-computed parameter buffers or ambient occlusion maps are required; computations are performed on-the-fly [S1, unofficial].
*   **Measured Performance:** Tested on Intel QuadCore 3.16 GHz and two Nvidia Geforce 480 GPUs rendering two registered $256^3$ datasets in a $1024 \times 768$ viewport [S1, unofficial].
    *   T1 (both opaque): L1 (no scattering) 45 fps, L2 (with scattering) 21 fps.
    *   T2 (1 opaque, 1 translucent): L1 11 fps, L2 9 fps.
    *   T3 (both half-opaque/half-scattering): L1 41 fps, L2 19 fps.
    *   T4 (both translucent): L1 41 fps, L2 11 fps.
    *   L1 = light transport only; L2 = light transport including scattering [S1, unofficial].
*   **Strengths:** Improves visual quality over local illumination models (e.g., Phong) by capturing single scattering effects; maintains interactive frame rates; suitable for multi-volume rendering.
*   **Limitations:** Approximation; relies on gradient direction which may not always accurately represent light transport paths; performance degrades with higher scattering ratios and translucent datasets [S1, unofficial].

## Background
*   **Origin:** Presented by Rostislav Khlebnikov, Bernhard Kainz, Bernhard Roth, Judith Muehl, and Dieter Schmalstieg at EUROGRAPHICS 2011 [S1, unofficial].
*   **History:** The method builds on the emission-absorption model from Max [Max95] and uses the Henyey-Greenstein phase function from [HG41] [S1, unofficial]. It was influenced by algorithms from Ropinski et al. [RDRS10] and Šoltészová et al. [ŠPBV10], which were not directly applicable to multi-volume rendering due to memory requirements or pre-computation needs [S1, unofficial].
*   **Usage:** Used for direct multi-volume rendering of intersecting datasets, such as registered MRI and DTI-MRI of the brain [S1, unofficial].

## Sources
[S1] GPU Based On-the-fly Light Emission-absorption Approximation for Direct Multi-volume Rendering, https://doi.org/10.2312/eg2011/posters/011-012, fetched 2026-10-26 (unofficial)
