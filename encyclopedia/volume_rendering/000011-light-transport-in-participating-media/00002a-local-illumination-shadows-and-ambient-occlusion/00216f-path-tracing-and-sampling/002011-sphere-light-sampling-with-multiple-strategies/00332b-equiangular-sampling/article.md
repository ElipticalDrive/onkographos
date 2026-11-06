# EquiAngular Sampling
Also called: Equiangular distance sampling, Kulla and Fajardo sampling

## Overview
Equiangular sampling is a Monte Carlo integration strategy for single scattering in participating media that samples the distance domain along an eye ray to minimize variance when the light source is partially occluded by geometry [S2, S3, unofficial]. It differs from its siblings, [Importance Sampling for Sphere Surface Area](2223-importance-sampling-for-sphere-surface-area.md) and [Sphere Light Shading with Importance Solid Angle Sampling](2236-sphere-light-shading-with-importance-solid-angle-sampling.md), by avoiding spherical sampling of the light source entirely; instead, it distributes samples uniformly in the angular domain subtended by the incident beam, which effectively importance samples the geometry term (visibility) [S3, S4, unofficial].

## How it works
**Inputs:**
* Ray origin $\vec{x}$ and direction $\vec{\omega}$ [S3, unofficial].
* Light position $\vec{K}$ and intensity $\Phi$ [S2, unofficial].
* Homogeneous medium extinction coefficient $\sigma_t$ (where $\sigma_t = \sigma_s + \sigma_a$) [S3, unofficial].
* Scattering factor $\sigma_s$ [S3, unofficial].
* Phase function (e.g., Henyey-Greenstein) [S3, unofficial].
* Number of samples per ray `spr` [S3, unofficial].

**Outputs:**
* Estimated in-scattered radiance contribution from the light [S3, unofficial].

**Data Structures:**
* Standard scalar and vector types suffice [G].

**Steps:**
1. **Geometric Pre-computation:** Calculate the perpendicular distance $D$ from the ray origin $\vec{x}$ to the light position $\vec{K}$ by finding the orthogonal projection of the light onto the eye ray [S1][S3, unofficial]. Determine the angles $\theta_a$ and $\theta_b$ subtended by the light at the ray origin, corresponding to the start and end of the visible segment of the light along the ray direction [S1][S3, unofficial].
2. **Sample Generation:** Generate a uniform random number $u \sim U[0,1]$ [S2, S3, unofficial].
3. **Distance Sampling:** Compute the sampled distance $t$ along the eye ray using the equiangular distribution formula [S2, S3, unofficial]:
   $$ t = D \tan \left( (1-u)\theta_a + u \theta_b \right) $$
   where $t$ is the sampled distance along the eye ray [S2, S3, unofficial].
4. **PDF Evaluation:** Calculate the probability density function (PDF) for the sampled distance $t$ [S2, S3, unofficial]:
   $$ pdf(t) = \frac{D}{(\theta_b - \theta_a)(D^2 + t^2)} $$
   where $pdf(t)$ is the probability density function for the sampled distance $t$ [S2, S3, unofficial].
5. **Radiance Estimation:** Compute the contribution using the single scattering estimator [S3, unofficial]:
   $$ \text{contribution} = \sigma_s T_r(x, x_t) T_r(x_t, K) Vis(x_t, K) \frac{\Phi}{D^2 + t^2} \frac{1}{pdf(t)} $$
   where:
   * $x_t = \vec{x} + t\vec{\omega}$ is the point on the eye ray [S3, unofficial].
   * $T_r(x, x_t) = \exp(-\sigma_t t)$ is the transmittance from the ray origin to the sample point [S3, unofficial].
   * $T_r(x_t, K)$ is the transmittance from the sample point to the light [S3, unofficial].
   * $Vis(x_t, K)$ is the visibility term (1 if visible, 0 if occluded) [S3, unofficial].
   * $\frac{\Phi}{D^2 + t^2}$ represents the inverse square falloff of the light intensity at distance $\sqrt{D^2+t^2}$ [S3, unofficial].
6. **Aggregation:** Accumulate `spr` samples per ray [S3, unofficial]. For multiple lights, add a for-loop outside the sampling procedure, similar to Next Event Estimation (NEE) [S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def equiangular_sample(ray_origin, ray_direction, light_pos, light_intensity, sigma_t, sigma_s, spr, trace_shadow_ray):
    """
    Estimates single scattering radiance from a light using equiangular sampling.
    """
    total_radiance = 0.0
    
    # Pre-compute geometric terms
    to_light = light_pos - ray_origin
    projection_length = dot(to_light, ray_direction)
    projection_point = ray_origin + ray_direction * projection_length
    D = length(light_pos - projection_point)
    
    # Angles theta_a and theta_b are computed from the visible angular span of the light or occluding geometry [G].
    if theta_a == theta_b:
        return 0.0 # Degenerate case handled by fallback or zero contribution
    
    for _ in range(spr):
        u = random.random()
        
        # Sample distance t
        angle = (1.0 - u) * theta_a + u * theta_b
        t = D * math.tan(angle)
        
        # Calculate PDF
        pdf_val = D / ((theta_b - theta_a) * (D**2 + t**2))
        
        # Calculate transmittance
        T_r_x_xt = math.exp(-sigma_t * t)
        
        # Distance from sample point to light
        x_t = ray_origin + ray_direction * t
        dist_to_light = length(light_pos - x_t)
        
        # T_r(x_t, K)
        T_r_xt_K = math.exp(-sigma_t * dist_to_light)
        
        # Visibility Vis(x_t, K)
        vis = trace_shadow_ray(x_t, light_pos) # [G] Standard shadow ray trace
        
        # Light intensity term
        light_term = light_intensity / (D**2 + t**2)
        
        # Contribution
        contribution = sigma_s * T_r_x_xt * T_r_xt_K * vis * light_term / pdf_val
        
        total_radiance += contribution
        
    return total_radiance / spr
```

**Parameters:**
* `spr` (samples per ray): Typical value is around 5, which yields better results than free-path sampling with comparable runtime [S3, unofficial].
* `t_MAX`: Maximum distance along the ray to consider for sampling [S2, unofficial].

**Edge Cases and Pitfalls:**
* **Degenerate Angles:** If $\theta_a = \theta_b$, the PDF denominator becomes zero [S2, S3, unofficial]. This occurs if the light is a point source directly on the ray or if the angular extent is zero [G].
* **Visibility:** The method relies on the visibility term $Vis(x_t, K)$ [S3, unofficial]. If the light is occluded, the contribution is zero [G]. The equiangular distribution helps sample the visible portion more effectively than uniform distance sampling [S1][S3, unofficial].
* **Multiple Scattering:** This method is designed for single scattering [S3, unofficial]. Applying it to multiple scattering scenarios may produce incorrect results or high noise [S3, unofficial].

## Operating characteristics
* **Time Complexity:** $O(L \cdot spr)$, where $L$ is the number of lights and $spr$ is the samples per ray [S3, unofficial].
* **Memory Complexity:** $O(1)$ auxiliary memory per sample [G].
* **Measured Performance:** Renders in less than 10 minutes with $spp < 100$ and $spr = 4$ on a standard scene, leveraging libraries like Embree3 [S3, unofficial].
* **Strengths:**
  * Minimizes variance when the light is partially occluded [S2, unofficial].
  * Represents the peak of the scattering profile well even with 1 sample [S4, unofficial].
  * Lower noise compared to free-path sampling for single scattering [S3, unofficial].
* **Limitations:**
  * Cannot match the tail behavior of exponential sampling [S4, unofficial].
  * Not suitable for multiple scattering [S3, unofficial].
  * Requires calculation of angular bounds $\theta_a$ and $\theta_b$, which may depend on scene geometry [S1][S3, unofficial].
* **When to Use:** Single scattering in participating media where the light source is partially occluded by geometry [S2, S3, unofficial].
* **When Not to Use:** Multiple scattering scenarios [S3, unofficial].

## Background
Equiangular sampling was proposed by Kulla and Fajardo [KF12] and used by Novák et al. [NNDJ12] [S3, unofficial]. It is a strategy where samples are distributed uniformly in the angular domain subtended by the incident beam [S4, unofficial]. The method was adopted in volumetric path tracing implementations to address the noise issues associated with free-path sampling in single scattering scenarios [S3, unofficial].

## Sources
[S1] Joint importance sampling of low-order volumetric scattering, https://doi.org/10.1145/2508363.2508411, fetched 2026-10-28
[S2] wangyuehi/VolumeSingleScattering, https://github.com/wangyuehi/VolumeSingleScattering, fetched 2026-10-26 (unofficial)
[S3] l0exy1h/VolumetricPathTracing-public, https://github.com/l0exy1h/VolumetricPathTracing-public, fetched 2026-10-26 (unofficial)
[S4] Photon Beam Diffusion for Rendering, https://www.scribd.com/document/255860491/Photon-Beam-Diffusion, fetched 2026-10-29 (unofficial)
