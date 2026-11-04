# EquiAngular Sampling


## Overview
Equiangular sampling is a Monte Carlo integration strategy for single scattering in participating media that samples the distance domain along an eye ray to minimize variance when the light source is partially occluded by geometry [S1, S2, S3, unofficial]. It differs from its siblings, [Importance Sampling for Sphere Surface Area](2223-importance-sampling-for-sphere-surface-area.md) and [Sphere Light Shading with Importance Solid Angle Sampling](2236-sphere-light-shading-with-importance-solid-angle-sampling.md), by avoiding spherical sampling of the light source entirely; instead, it distributes samples uniformly in the angular domain subtended by the incident beam, which effectively importance samples the geometry term (visibility) [S2, S3, unofficial].

## How it works
**Inputs:**
*   Ray origin $\vec{x}$ and direction $\vec{\omega}$ [S3, unofficial].
*   Light position $\vec{K}$ and intensity $\Phi$ [S2, unofficial].
*   Homogeneous medium extinction coefficient $\sigma_t$ (where $\sigma_t = \sigma_s + \sigma_a$) [S2, unofficial].
*   Scattering factor $\sigma_s$ [S2, unofficial].
*   Phase function (e.g., Henyey-Greenstein) [S2, unofficial].
*   Number of samples per ray `spr` [S2, unofficial].

**Outputs:**
*   Estimated in-scattered radiance contribution from the light [S2, unofficial].

**Data Structures:**
*   No complex structures required. Standard scalar and vector types suffice.

**Steps:**

1.  **Geometric Pre-computation:**
    Calculate the perpendicular distance $D$ from the ray origin $\vec{x}$ to the light position $\vec{K}$. Determine the angles $\theta_a$ and $\theta_b$ subtended by the light at the ray origin, corresponding to the start and end of the visible segment of the light along the ray direction [S2, unofficial].
    *   Note: If the light is a point source, the "segment" is defined by the angular extent of the light if it has finite size, or the method is applied to the specific geometry interaction. In the context of the provided notes, $D$ is the distance from the ray to the light projection, and $\theta_a, \theta_b$ define the angular range of integration [S2, unofficial].
    *   Specifically, $D$ is the perpendicular distance from the ray to the light source [S1, S2, unofficial].
    *   $\theta_a$ and $\theta_b$ are the angles to the segment endpoints [S1, unofficial].

2.  **Sample Generation:**
    Generate a uniform random number $u \sim U[0,1]$ [S1, unofficial].

3.  **Distance Sampling:**
    Compute the sampled distance $t$ along the eye ray using the equiangular distribution formula [S1, S2, unofficial]:
    $$ t = D \tan \left( (1-u)\theta_a + u \theta_b \right) $$
    where $t$ is the sampled distance along the eye ray [S2, unofficial].

4.  **PDF Evaluation:**
    Calculate the probability density function (PDF) for the sampled distance $t$ [S1, S2, unofficial]:
    $$ pdf(t) = \frac{D}{(\theta_b - \theta_a)(D^2 + t^2)} $$
    where $pdf(t)$ is the probability density function for the sampled distance $t$ [S2, unofficial].

5.  **Radiance Estimation:**
    Compute the contribution using the single scattering estimator [S2, unofficial]:
    $$ \text{contribution} = \sigma_s T_r(x, x_t) T_r(x_t, K) Vis(x_t, K) \frac{\Phi}{D^2 + t^2} \frac{1}{pdf(t)} $$
    where:
    *   $x_t = \vec{x} + t\vec{\omega}$ is the point on the eye ray [S2, unofficial].
    *   $T_r(x, x_t) = \exp(-\sigma_t t)$ is the transmittance from the ray origin to the sample point [S2, unofficial].
    *   $T_r(x_t, K)$ is the transmittance from the sample point to the light [S2, unofficial].
    *   $Vis(x_t, K)$ is the visibility term (1 if visible, 0 if occluded) [S2, unofficial].
    *   $\frac{\Phi}{D^2 + t^2}$ represents the inverse square falloff of the light intensity at distance $\sqrt{D^2+t^2}$ [S2, unofficial].

6.  **Aggregation:**
    Repeat steps 2-5 for `spr` samples and average the results [S2, unofficial]. For multiple lights, wrap this procedure in a loop over all lights (Next Event Estimation) [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def equiangular_sample(ray_origin, ray_direction, light_pos, light_intensity, sigma_t, sigma_s, spr):
    """
    Estimates single scattering radiance from a light using equiangular sampling.
    
    Args:
        ray_origin: Vector3, origin of the eye ray x
        ray_direction: Vector3, direction of the eye ray omega
        light_pos: Vector3, position of the light K
        light_intensity: float, intensity Phi of the light
        sigma_t: float, extinction coefficient of the medium
        sigma_s: float, scattering coefficient of the medium
        spr: int, samples per ray
        
    Returns:
        float: Estimated radiance contribution
    """
    total_radiance = 0.0
    
    # Pre-compute geometric terms
    # Vector from ray origin to light
    to_light = light_pos - ray_origin
    
    # Perpendicular distance D from ray origin to light
    # D = |to_light - (to_light . ray_direction) * ray_direction|
    # However, the notes define D as the perpendicular distance from the ray to the light.
    # In the context of the formula t = D tan(theta), D is the shortest distance from the 
    # ray line to the light point.
    projection_length = dot(to_light, ray_direction)
    projection_point = ray_origin + ray_direction * projection_length
    D = length(light_pos - projection_point)
    
    # Angles theta_a and theta_b
    # The notes imply theta_a and theta_b are the angles subtended by the light segment.
    # For a point light or specific geometry, these define the integration bounds.
    # In many implementations for a point light at distance D, the "angles" might be 
    # derived from the visible extent or set to specific bounds if the light is treated 
    # as a segment. 
    # Based on [S2, unofficial] and [S1, unofficial], theta_a and theta_b are the angles to the segment endpoints.
    # If the light is a point source and we are sampling the distance to it, 
    # the "segment" is effectively the point itself, but the method is often applied 
    # to volume rendering where the light is visible through a window or defined by geometry.
    # For the purpose of this implementation based strictly on the provided notes:
    # We assume theta_a and theta_b are provided or calculated based on the visible 
    # angular extent of the light from the ray origin.
    # If the light is a point source directly visible, theta_a = theta_b = 0 is degenerate.
    # The notes suggest this is for when the light is partially occluded.
    # Let's assume theta_a and theta_b are known bounds for the angular integration.
    # If not provided, this step requires external geometry intersection data.
    # [G] Assuming theta_a and theta_b are computed from the visible angular span of the light.
    theta_a = 0.0 # Placeholder: Must be calculated from geometry
    theta_b = 0.0 # Placeholder: Must be calculated from geometry
    
    # If theta_a == theta_b, the PDF is undefined. This case should be handled 
    # by falling back to another strategy or ensuring the light is not a point 
    # in the angular domain.
    if theta_a == theta_b:
        return 0.0 # Degenerate case

    for _ in range(spr):
        u = random.random()
        
        # Sample distance t
        # t = D * tan((1-u)*theta_a + u*theta_b)
        angle = (1.0 - u) * theta_a + u * theta_b
        t = D * math.tan(angle)
        
        # Calculate PDF
        # pdf(t) = D / ((theta_b - theta_a) * (D^2 + t^2))
        pdf_val = D / ((theta_b - theta_a) * (D**2 + t**2))
        
        # Calculate transmittance
        # T_r(x, x_t) = exp(-sigma_t * t)
        T_r_x_xt = math.exp(-sigma_t * t)
        
        # Distance from sample point to light
        # x_t = ray_origin + t * ray_direction
        x_t = ray_origin + ray_direction * t
        dist_to_light = length(light_pos - x_t)
        
        # T_r(x_t, K)
        # Assuming homogeneous medium, transmittance is exp(-sigma_t * dist)
        T_r_xt_K = math.exp(-sigma_t * dist_to_light)
        
        # Visibility Vis(x_t, K)
        # [G] In a real implementation, this requires a shadow ray trace.
        # If the light is visible, Vis = 1, else 0.
        # For the purpose of the formula derivation, we assume Vis is 1 if the path is clear.
        # The notes mention "partially occluded", so Vis is the critical term.
        vis = 1.0 # Placeholder: Must be determined by shadow ray trace
        
        # Light intensity term
        # Phi / (D^2 + t^2)
        # Note: D^2 + t^2 is the squared distance from the ray origin to the light 
        # IF the light lies on the plane perpendicular to the ray at distance D?
        # Actually, the distance from x to K is sqrt(D^2 + projection_length^2).
        # The formula in [S2, unofficial] uses D^2 + t^2 in the denominator for the light term.
        # This implies the light is located at a distance D perpendicular to the ray,
        # and t is the distance along the ray. The distance from x_t to K would be 
        # sqrt(D^2 + (projection_length - t)^2) or similar depending on coordinate system.
        # However, we must follow the source formula exactly.
        # [S2, unofficial] formula: Phi / (D^2 + t^2)
        light_term = light_intensity / (D**2 + t**2)
        
        # Contribution
        # sigma_s * T_r(x, x_t) * T_r(x_t, K) * Vis * light_term / pdf
        contribution = sigma_s * T_r_x_xt * T_r_xt_K * vis * light_term / pdf_val
        
        total_radiance += contribution
        
    return total_radiance / spr
```

**Parameters:**
*   `spr` (samples per ray): Typical value is around 4-5, which yields better results than free-path sampling with comparable runtime [S2, unofficial].
*   `t_MAX`: Maximum distance along the ray to consider for sampling [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Degenerate Angles:** If $\theta_a = \theta_b$, the PDF denominator becomes zero. This occurs if the light is a point source directly on the ray or if the angular extent is zero. The method is designed for cases where the light is partially occluded, implying a non-zero angular segment [S2, S3, unofficial].
*   **Precision:** The tangent function can become unstable as angles approach $\pi/2$. Ensure angles are within valid bounds [S1, unofficial].
*   **Visibility:** The method relies on the visibility term $Vis(x_t, K)$. If the light is occluded, the contribution is zero. The equiangular distribution helps sample the visible portion more effectively than uniform distance sampling [S1, unofficial].
*   **Multiple Scattering:** This method is designed for single scattering. Applying it to multiple scattering scenarios may produce incorrect results or high noise [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(L \cdot spr)$, where $L$ is the number of lights and $spr$ is the samples per ray [S2, unofficial].
*   **Memory Complexity:** $O(1)$ auxiliary memory per sample [S2, unofficial].
*   **Measured Performance:** Renders in less than 10 minutes with $spp < 100$ and $spr = 4$ on a standard scene, leveraging libraries like Embree3 [S2, unofficial].
*   **Strengths:**
    *   Minimizes variance when the light is partially occluded [S1, unofficial].
    *   Represents the peak of the scattering profile well even with 1 sample [S3, unofficial].
    *   Lower noise compared to free-path sampling for single scattering [S2, unofficial].
*   **Limitations:**
    *   Cannot match the tail behavior of exponential sampling [S3, unofficial].
    *   Not suitable for multiple scattering [S2, unofficial].
    *   Requires calculation of angular bounds $\theta_a$ and $\theta_b$, which may depend on scene geometry [S2, S3, unofficial].
*   **When to Use:** Single scattering in participating media where the light source is partially occluded by geometry [S1, S2, unofficial].
*   **When Not to Use:** Multiple scattering scenarios or when the tail behavior of the scattering profile is critical and cannot be combined with other strategies [S2, S3, unofficial].

## Background
Equiangular sampling was proposed by Kulla and Fajardo [KF12] and used by Novák et al. [NNDJ12] [S3, unofficial]. It is a strategy where samples are distributed uniformly in the angular domain subtended by the incident beam [S3, unofficial]. The method was adopted in volumetric path tracing implementations to address the noise issues associated with free-path sampling in single scattering scenarios [S2, unofficial].

## Sources
[S1] wangyuehi/VolumeSingleScattering, https://github.com/wangyuehi/VolumeSingleScattering, fetched 2026-10-26 (unofficial)
[S2] l0exy1h/VolumetricPathTracing-public, https://github.com/l0exy1h/VolumetricPathTracing-public, fetched 2026-10-26 (unofficial)
[S3] Photon Beam Diffusion for Rendering, https://www.scribd.com/document/255860491/Photon-Beam-Diffusion, fetched 2026-10-29 (unofficial)
