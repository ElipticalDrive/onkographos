# Discrete Ray-Marching with Forward Scattering Approximation

Also called: Ray-marching with light propagation approximation

## Overview
This method approximates the in-scattering integral for volumetric clouds by discretizing the continuous light propagation equation into a Riemann sum using fixed-step ray marching [S1]. It distinguishes itself from general in-scattering approximations by explicitly assuming that 50% of scattered light propagates in the forward direction within a small solid angle (~0.0001 steradians), ignoring the remaining 50% to reduce computational cost [S1]. It builds on [In-Scattering Integral Approximation via Riemann Sum](1585-in-scattering-integral-approximation-via-riemann-sum.md) by providing the specific physical assumptions and numerical simplifications (Maclaurin expansion) required for real-time GPU implementation.

## How it works

### Inputs and Outputs
**Inputs:**
*   Voxel grid lighting data, passed to the GPU as a 3D texture [S1].
*   Ray marching step size $\Delta s$ [S1].
*   Asymmetry factor $g$ for the phase function [S1].
*   Surface transmittance parameters [S1].

**Outputs:**
*   Final pixel color $I_{total}$ for each frame buffer point, comprising reflected and scattered light components [S1].

### Data Structures
*   **3D Texture:** Stores voxel grid lighting values. This structure enables hardware-accelerated trilinear interpolation to sample lighting values along the marching ray [S1].
*   **Ray State:** Contains current position $s$, accumulated transmittance, and accumulated incident light [S1].

### Formulas and Definitions
Let $s$ be the position along the ray, starting from the cloud surface. Let $L(s_i)$ be the illumination at intermediate point $i$. Let $T_i$ be the transmittance from the surface to point $i$ [S1].

1.  **Forward Scattering Assumption:** 50% of light scatters in the forward direction within a small solid angle of approximately 0.0001 steradians. The remaining 50% is ignored [S1].
2.  **Incident Light ($I_{inc}$):** Calculated as the summation of light emitted by scattering from each intermediate point $i$, attenuated by transmittance [S1]:
    $$ I_{inc} = \sum_i L(s_i) \cdot T_i \cdot \Delta s_i $$
    *(Note: Derived from the context of Equation 5 conversion to discrete form and Equation 7 in [S1])*
3.  **Total Light ($I_{total}$):** The sum of reflected and scattered light, both affected by transmittance from the surface to point $s$ [S1]:
    $$ I_{total} = I_{reflected} + I_{scattered} $$
    Where $I_{reflected}$ depends on illumination $L(s)$ and occluded area, and $I_{scattered}$ includes a phase function $p(\theta)$ [S1].
4.  **Segment Integral Approximation:** The continuous integral is converted to a summation of segment integrals [S1]:
    $$ \int_{s_i}^{s_{i+1}} \dots ds \approx \sum_i \dots $$
    The phase term is considered constant along the segment and factored out [S1].
5.  **Differential Transmittance ($dT$):** Denoted as the differential transmittance along the segment [S1].
6.  **Transmittance Approximation:** The exponential function $e^{-\tau}$ is approximated using the first two terms of its Maclaurin series expansion [S1]:
    $$ e^{-\tau} \approx 1 - \tau $$
    This simplifies the segment integral calculation [S1].
7.  **Phase Function:** The Henyey–Greenstein function is used to approximate refraction of light in water droplets [S1]:
    $$ p(\theta) = \frac{1-g^2}{(1+g^2-2g\cos\theta)^{3/2}} $$
    Where $\theta$ is the scattering angle and $g$ is the asymmetry factor [S1].

### Parameters
*   **Step Size ($\Delta s$):** The fixed distance between ray march steps [S1].
*   **Asymmetry Factor ($g$):** Parameter for the Henyey–Greenstein phase function [S1].
*   **Solid Angle:** Assumed to be ~0.0001 steradians for the forward scattering component [S1].

### Pseudocode
The following pseudocode implements the ray-marching pass described in [S1]. It assumes a 3D texture `lightingTex` is available for trilinear sampling.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_march_cloud(ray_origin, ray_direction, cloud_bounds, step_size, g):
    """
    Renders a cloud volume using discrete ray-marching with forward scattering approximation.
    
    Args:
        ray_origin: vec3, origin of the camera ray.
        ray_direction: vec3, normalized direction of the camera ray.
        cloud_bounds: Bounding volume for the cloud (e.g., AABB).
        step_size: float, distance between march steps.
        g: float, asymmetry factor for Henyey-Greenstein phase function.
        
    Returns:
        vec3, the final color for the pixel.
    """
    
    # 1. Intersect ray with cloud bounds to find start and end points
    t_min, t_max = intersect_aabb(ray_origin, ray_direction, cloud_bounds)
    if t_max < 0:
        return vec3(0.0) # No intersection
        
    s_start = ray_origin + ray_direction * t_min
    s_end = ray_origin + ray_direction * t_max
    
    # 2. Initialize accumulators
    total_reflected = vec3(0.0)
    total_scattered = vec3(0.0)
    current_transmittance = 1.0 # Transmittance from surface to current point
    
    # 3. Ray Marching Loop
    current_pos = s_start
    while current_pos < s_end:
        # Sample lighting from 3D texture using hardware-accelerated trilinear interpolation
        # [S1] "passes the voxel grid lighting in a 3D texture to the GPU... permits hardware accelerated trilinear interpolation"
        illumination = sample_3d_texture(lightningTex, current_pos)
        
        # Calculate scattering angle theta between light source direction and view direction
        # [S1] "angle between the light source and the observer"
        # Note: Light source direction is assumed to be known or constant for this approximation
        light_dir = normalize(light_source_position - current_pos)
        view_dir = -ray_direction
        cos_theta = dot(light_dir, view_dir)
        
        # Calculate Henyey-Greenstein phase function
        # [S1] p(theta) = (1-g^2) / (1 + g^2 - 2g*cos(theta))^(3/2)
        numerator = 1.0 - g * g
        denominator = 1.0 + g * g - 2.0 * g * cos_theta
        phase = numerator / (denominator ** 1.5)
        
        # Approximate transmittance using Maclaurin series: e^(-tau) approx 1 - tau
        # [S1] "exponential function e can be approximated using the first two terms of its McLaurin series expansion"
        # tau is the optical depth over the segment. 
        # Let sigma_t be the extinction coefficient. tau = sigma_t * step_size.
        # Since sigma_t is not explicitly given as a variable in the notes, we assume it is part of the 
        # "transmittance from surface to point i" calculation or embedded in the lighting sample.
        # However, the formula I_inc = sum L(s_i) * T_i * Delta s_i suggests T_i is pre-calculated or sampled.
        # We will use the provided formula structure.
        
        # Incident light contribution (Scattered Light)
        # [S1] I_inc = sum L(s_i) * T_i * Delta s_i
        # The forward scattering assumption implies we only take 50% of the scattering
        forward_scattering_factor = 0.5 
        
        # Differential transmittance dT is approximated. 
        # The note says "differential transmittance along segment is denoted as dT" and uses 1-tau.
        # We assume current_transmittance represents T_i in the sum.
        
        scattered_contribution = illumination * phase * forward_scattering_factor * current_transmittance * step_size
        total_scattered += scattered_contribution
        
        # Reflected Light
        # [S1] "light reflected at point s... depends on illumination L(s) and area occluded by particles"
        # "affected by the transmittance from point s to the surface"
        # Note: "area occluded by particles" is not explicitly defined as a variable in the notes.
        # We assume it is handled by the density/transmittance logic or is a constant factor.
        # For implementation completeness, we assume a simple reflection model proportional to illumination.
        reflected_contribution = illumination * current_transmittance * step_size # Placeholder for occlusion factor [G]
        total_reflected += reflected_contribution
        
        # Update transmittance for the next step
        # [S1] T_i is transmittance from surface to point i.
        # We need to update current_transmittance for the next iteration.
        # Using the approximation e^(-tau) approx 1 - tau.
        # tau = sigma_t * step_size. 
        # If we don't have sigma_t, we might need to sample it or assume it's 1 for simplicity in this abstract.
        # However, the note implies T_i is calculated. Let's assume a standard density sigma_t is available or 1.
        sigma_t = 1.0 # Not given in sources, assumed for validity of 1-tau [G]
        tau = sigma_t * step_size
        current_transmittance *= (1.0 - tau)
        
        # Advance ray
        current_pos += ray_direction * step_size
        
    # 4. Combine results
    # [S1] I_total = I_reflected + I_scattered
    final_color = total_reflected + total_scattered
    
    return final_color
```

### Edge Cases and Pitfalls
*   **Maclaurin Approximation Validity:** The approximation $e^{-\tau} \approx 1 - \tau$ is only valid for small $\tau$. If the step size $\Delta s$ or extinction coefficient is large, this approximation will introduce significant error or negative transmittance [S1].
*   **Forward Scattering Assumption:** Ignoring 50% of scattered light may lead to visually darker or less accurate clouds, particularly in dense regions where multiple scattering is significant [S1].
*   **Phase Function Singularity:** The Henyey–Greenstein function can become sharp for $g$ close to 1. Ensure numerical stability when calculating the denominator [S1].

## Operating characteristics

### Time and Memory Complexity
*   **Time Complexity:** Linear with respect to the number of ray march steps $N$. Each step involves a 3D texture lookup, phase function calculation, and accumulation [S1].
*   **Memory Complexity:** Dominated by the storage of the 3D texture for voxel grid lighting [S1].

### Measured Performance
*   The method is designed for real-time GPU rendering [S1].
*   Performance is improved by using hardware-accelerated trilinear interpolation for lighting samples [S1].
*   Specific frame rates or resolution metrics are not provided in the sources [S1].

### Strengths and Limitations
*   **Strengths:** Computationally efficient due to the forward scattering approximation (ignoring 50% of scattering) and the Maclaurin series approximation for transmittance [S1]. Suitable for real-time applications [S1].
*   **Limitations:** Less physically accurate due to the 50% scattering loss and the linear transmittance approximation [S1].

### When to Use
*   Use when real-time performance is critical and high physical accuracy is not required [S1].
*   Suitable for rendering volumetric clouds where forward scattering is the dominant visual effect [S1].

### When Not to Use
*   Avoid when high-fidelity physical accuracy is needed, especially in dense volumes where multiple scattering is significant [S1].
*   Avoid if the Maclaurin approximation for transmittance is invalid for the given density and step size [S1].

## Background

### Origin
The method is described in the paper "Efficient Algorithms for Real-Time GPU Volumetric Cloud Rendering with Enhanced Geometry" [S1].

### History and Development
The approach builds upon previous work on light propagation and scattering approximations, specifically referencing [13] for the forward scattering assumption and [33] for the segment integral conversion [S1].

### Usage
It is used for real-time GPU volumetric cloud rendering [S1].

## Sources

[S1] Efficient Algorithms for Real-Time GPU Volumetric Cloud Rendering with Enhanced Geometry, https://doi.org/10.3390/sym10040125, fetched 2026-10-26
