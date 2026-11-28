# Limited Secondary Ray Sampling

Also called: Limited Secondary Light Sampling, Secondary Ray Optimization

## Overview
Limited Secondary Ray Sampling is an optimization strategy for computing volumetric self-shadowing in distance-field ray marching [S1, unofficial]. It reduces computational cost by limiting the maximum travel distance of secondary rays and adaptively reducing sample counts based on primary ray depth and local density [S1, unofficial]. Unlike global shadowing methods, it restricts self-shadowing to a local neighborhood, trading long-range accuracy for performance [S1, unofficial][S2].

## How it works
The method augments standard primary ray marching (view direction) with secondary ray marching (light direction) to estimate transmittance and lighting [S1, unofficial][S2].

**Inputs and Outputs:**
- Input: Primary ray origin, direction, accumulated density along the primary ray, light direction, and density field access function [S1, unofficial][G].
- Output: Shadowed radiance contribution for the current primary sample [S1, unofficial].

**Data Structures:**
- Primary Ray State: Accumulated density $\tau_p$, current position $P$, step size $h_p$ [G].
- Secondary Ray State: Accumulated density $\tau_s$, current position $Q$, step size $h_s$ [G].
- Parameters:
  - $D_{max}$: Maximum secondary ray distance. Typical value: 1000 meters [S1, unofficial].
  - $N_{sec}$: Number of secondary samples. Adaptive based on primary distance and density [S1, unofficial].
  - $\tau_{thresh}$: Primary ray termination threshold. When $\tau_p \ge \tau_{thresh}$, the cloud is treated as fully opaque [S1, unofficial].

**Algorithm Steps:**
1. **Primary Ray Marching:** March along the view direction. At each primary sample point $P_i$, if accumulated density $\tau_p \ge \tau_{thresh}$, terminate the primary ray and treat the point as fully opaque [S1, unofficial].
2. **Adaptive Sampling:** Calculate $N_{sec}$. Reduce $N_{sec}$ logarithmically as the primary sample distance from the camera increases, which is effective with minimal quality impact [S1, unofficial]. Reduce $N_{sec}$ for primary samples with low density $\rho(P_i)$, as transparency tolerates lower lighting accuracy [S1, unofficial].
3. **Secondary Ray Marching:** Initialize the secondary ray at $P_i$ towards the light direction $L$. Limit the maximum travel distance to $D_{max}$ to reduce computational cost [S1, unofficial]. Use a linear scale for the sample distribution along the secondary ray [S1, unofficial].
4. **Weighting & Accumulation:** For each secondary step, sample the density, accumulate optical depth $\tau_s$, and apply a linear importance weight to fade out distant samples. This avoids hard margins in the lighting calculation [S1, unofficial].
5. **Lighting Calculation:** Compute transmittance $T = e^{-\tau_s}$ [G]. Combine the weighted transmittance with the primary ray color to produce the final shadowed radiance [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def limited_secondary_ray_sampling(primary_origin, primary_dir, light_dir, 
                                   density_field, D_max=1000.0, tau_thresh=5.0, 
                                   base_secondary_samples=16, primary_step=1.0):
    """
    Perform primary ray march with adaptive secondary light sampling.
    """
    primary_pos = primary_origin
    primary_dir = normalize(primary_dir)
    light_dir = normalize(light_dir)
    tau_p = 0.0
    
    # Standard lighting constants [G]
    light_color = vec3(1.0)
    base_color = vec3(0.0)
    
    while tau_p < tau_thresh:
        # Sample density at current primary position
        rho_p = density_field.sample(primary_pos)
        
        # Adaptive secondary sample count [G]
        dist_from_cam = length(primary_pos - primary_origin)
        # Logarithmic reduction with distance from camera
        dist_factor = max(1.0, log(dist_from_cam + 1.0))
        # Linear reduction with low density
        density_factor = max(0.1, rho_p / 10.0)
        
        num_secondary = max(1, int(base_secondary_samples / (dist_factor * density_factor)))
        
        # Secondary ray march towards light
        secondary_pos = primary_pos
        tau_s = 0.0
        weighted_transmittance = 0.0
        weight_sum = 0.0
        
        # Linear scale for sample distribution [S1, unofficial]
        step = D_max / num_secondary
        
        for i in range(num_secondary):
            secondary_pos += light_dir * step
            
            # Check max distance limit
            dist_from_primary = length(secondary_pos - primary_pos)
            if dist_from_primary > D_max:
                break
            
            # Sample density at current position
            rho_s = density_field.sample(secondary_pos)
            
            # Accumulate optical depth
            tau_s += rho_s * step
            
            # Linear importance weighting to fade out distant samples
            # Avoids hard margins in lighting calculation [S1, unofficial]
            w = max(0.0, 1.0 - (dist_from_primary / D_max))
            
            # Compute transmittance
            transmittance = exp(-tau_s)
            
            # Accumulate weighted contribution
            weighted_transmittance += w * transmittance
            weight_sum += w
        
        # Normalize weighted light by sum of weights
        shadow_factor = weighted_transmittance / weight_sum if weight_sum > 0 else 1.0
        
        # Combine with primary ray color
        color = base_color + shadow_factor * light_color
        
        # Advance primary ray
        tau_p += rho_p * primary_step
        primary_pos += primary_dir * primary_step
        
        yield color

    # Terminate when accumulated density reaches threshold [S1, unofficial]
    return vec3(0.0)
```

**Edge Cases and Pitfalls:**
- **Hard Margins:** Failing to fade out secondary samples causes visible artifacts at the edge of the shadow volume [S1, unofficial].
- **Precision:** Accumulated density can overflow if step sizes are too large or not clamped [G].
- **Boundary:** Secondary rays may exit the volume prematurely; ensure the density field returns zero outside bounds [G].

## Subtopics

### Ray Marched Volume Shadow
A specific implementation mode in Unreal Engine's Volumetric Cloud system that uses secondary ray marching for shadowing [S2]. It produces sharp, colored shadows but is limited in trace distance due to sample constraints [S2]. It is more expensive than Beer Shadow Maps but supports ground-to-sky-to-space transitions and provides volumetric self-shadow color [S2]. The cost is driven by the `Shadow View Sample Count Scale` parameter on the Volumetric Cloud component [S2]. A key pitfall is the limited distance shadows can be traced [S2].

## Operating characteristics
- **Time and Memory Complexity:** Not given in the sources.
- **Performance:** Significantly faster than full secondary ray marching due to distance and sample count limits [S1, unofficial]. On console platforms (Xbox One, PS4), Beer Shadow Maps are often recommended over Ray Marched shadows due to cost, though Ray Marched provides better quality [S2].
- **Strengths:** Local self-shadowing is accurate and colored; performance is manageable for real-time [S1, unofficial][S2].
- **Limitations:** Shadows do not extend beyond $D_{max}$ (e.g., 1000m); long-range shadows require alternative methods like Beer Shadow Maps [S1, unofficial][S2].

## Background
The method originates from practical implementations of efficient volumetric clouds in real-time rendering, particularly in community-driven discussions on density thresholds and secondary ray limits [S1, unofficial], and is formally documented in Unreal Engine's Volumetric Cloud system [S2]. It addresses the high cost of secondary ray marching for self-shadowing by introducing locality and adaptivity [S1, unofficial].

## Sources
[S1] Efficient volumetric clouds - Resources - three.js forum, https://discourse.threejs.org/t/efficient-volumetric-clouds/66067, fetched 2026-10-29 (unofficial)
[S2] Volumetric Cloud Component in Unreal Engine  | Unreal Engine 5.8 Documentation | Epic Developer Community, https://dev.epicgames.com/documentation/en-us/unreal-engine/volumetric-cloud-component-in-unreal-engine, fetched 2026-10-30
