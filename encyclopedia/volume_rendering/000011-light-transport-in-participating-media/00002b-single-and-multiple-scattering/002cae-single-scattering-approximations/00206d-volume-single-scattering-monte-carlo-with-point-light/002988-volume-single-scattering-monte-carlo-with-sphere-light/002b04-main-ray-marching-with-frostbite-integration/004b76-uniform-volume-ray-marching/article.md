# Uniform Volume Ray Marching

Also called: Backward Ray Marching, Forward Ray Marching

## Overview
Uniform Volume Ray Marching is a rendering technique for participating media with constant opacity and density, using ray marching to estimate the integral of light scattering [S1, unofficial]. It assumes a single light source and constant volume properties, allowing precomputation of attenuation factors [S1, unofficial]. Forward ray marching is preferred over backward ray marching for uniform volumes because it allows for early termination when transmission becomes very small, optimizing performance [S1, unofficial].

## How it works
**Inputs:**
- Camera ray origin $O$ and direction $D$ [G].
- Volume density $\sigma_t$ (constant) [S1, unofficial].
- Light source position $L$ and color $C_L$ [G].
- Step size $\Delta s$ [S1, unofficial].
- Early termination threshold $\epsilon$ [G].
- Maximum number of steps $N_{max}$ [G].

**Outputs:**
- Final pixel color $C_{out}$ (RGBA) [G].

**Data Structures:**
- `overall_transmission`: Floating-point scalar, initialized to $1.0$ [S1, unofficial].
- `color_result`: Vector (RGB), initialized to $0.0$ [G].
- `current_position`: Vector, initialized to $O$ [G].

**Steps:**
1. Initialize `overall_transmission` to $1.0$ and `color_result` to $0.0$ [S1, unofficial].
2. Iterate along the ray direction $D$ for up to $N_{max}$ steps:
   a. Compute the in-scattered light $L_i(x)$ at the current sample point $x$, representing the light attenuated from the source to the sample point [S1, unofficial].
   b. Accumulate the final color: `color_result` += $L_i(x) \times$ `overall_transmission` [S1, unofficial].
   c. Update `overall_transmission` by multiplying it by the current sample's transmission value $T_{sample}$ [S1, unofficial]. For a uniform volume, $T_{sample} = e^{-\sigma_t \Delta s}$ [S1, unofficial].
   d. Advance `current_position` by $\Delta s \cdot D$ [G].
   e. If `overall_transmission` drops below a small threshold close to zero, terminate the loop early [S1, unofficial].
3. Return `color_result` and `overall_transmission` as the final RGBA pixel value [G].

**Formulas:**
- Sample transmission: $T_{sample} = e^{-\sigma_t \Delta s}$ [S1, unofficial].
- In-scattered light: $L_i(x)$ is computed based on the attenuation of light from the source to $x$ [S1, unofficial]. The exact formula depends on the light model, but the accumulation term is $L_i(x) \cdot T_{overall}$ [S1, unofficial].
- Pixel width estimation for step size: $\text{step\_size} \approx 2 \cdot \tan(\frac{\pi}{180} \cdot \frac{FOV}{2}) \cdot \text{image\_width}$ [S1, unofficial]. Note: The source provides a garbled formula, but this is the standard interpretation for estimating pixel width to inform step size selection [S1, unofficial].

**Parameters:**
- Step size $\Delta s$: A constant distance between march steps. Trade-off between speed and accuracy [S1, unofficial].
- Early termination threshold $\epsilon$: A small value close to zero [S1, unofficial].

**Pitfalls:**
- If the step size is too large, the ray may overstep small solid objects casting shadows, completely ignoring the shadow and creating visual artifacts [S1, unofficial].
- If the volume is not uniform, a constant step size may miss small features or shadows (aliasing/filtering issue) [S1, unofficial].
- A source typo claimed that a step size that is 'too small' misses detail; the correct interpretation is that a step size that is 'too large' misses detail [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def uniform_volume_ray_marching(ray_origin, ray_direction, volume_density, light_position, light_color, step_size, threshold, max_steps):
    overall_transmission = 1.0
    color_result = [0.0, 0.0, 0.0]
    current_position = list(ray_origin)
    
    for _ in range(max_steps):
        if overall_transmission < threshold:
            break
            
        # Compute in-scattered light Li(x) [G]
        light_dir = [light_position[i] - current_position[i] for i in range(3)]
        dist = math.sqrt(sum(d*d for d in light_dir))
        dist = max(dist, 1e-6)
        li_x = [light_color[i] / (dist * dist) for i in range(3)]
        
        # Accumulate color before updating transmission
        color_result[0] += li_x[0] * overall_transmission
        color_result[1] += li_x[1] * overall_transmission
        color_result[2] += li_x[2] * overall_transmission
        
        # Update overall transmission
        sample_transmission = math.exp(-volume_density * step_size)
        overall_transmission *= sample_transmission
        
        # Advance position
        current_position[0] += ray_direction[0] * step_size
        current_position[1] += ray_direction[1] * step_size
        current_position[2] += ray_direction[2] * step_size
        
    return [color_result[0], color_result[1], color_result[2], overall_transmission]
```

## Subtopics

### [Main Ray Marching with Frostbite Integration](4931-main-ray-marching-with-frostbite-integration.md)
Handles non-uniform volumes and complex integration schemes. Choose when volume properties vary.

### [Volume Single Scattering (Monte Carlo with Point Light)](52-volume-single-scattering-monte-carlo-with-point-light.md)
Uses Monte Carlo methods for single scattering. Choose for stochastic accuracy or complex light interactions.

### [Volume Single Scattering (Monte Carlo with Sphere Light)](53-volume-single-scattering-monte-carlo-with-sphere-light.md)
Extends Monte Carlo to sphere lights. Choose for soft shadows from area lights.

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the number of steps [G]. Early termination can reduce effective $N$ [S1, unofficial].
- **Memory Complexity:** $O(1)$, as it only requires scalar and vector accumulators [G].
- **Strengths:** Simple to implement; early termination optimizes performance for opaque regions; precomputable attenuation for uniform volumes [S1, unofficial].
- **Limitations:** Assumes uniform volume properties; step size trade-off can lead to aliasing or missed details if not tuned correctly; not suitable for non-uniform volumes without modification [S1, unofficial].
- **When to use:** Uniform volumes with constant density and opacity; single light source; performance-critical applications where early termination is beneficial [S1, unofficial].
- **When not to use:** Non-uniform volumes; multiple light sources; high-frequency details requiring adaptive step sizes [S1, unofficial].

## Background
Originated in the context of volume rendering for developers, specifically discussed in "Volume Rendering for Developers" Group Read Part 1 [S1, unofficial]. The technique is a simplified version of ray marching tailored for uniform media, allowing for precomputed attenuation factors [S1, unofficial].

## Sources
[S1] "Volume Rendering for Developers" Group Read Part 1 [2023-03-30], https://www.youtube.com/watch?v=mZzMVD4LQvk, fetched 2026-10-26 (unofficial)
