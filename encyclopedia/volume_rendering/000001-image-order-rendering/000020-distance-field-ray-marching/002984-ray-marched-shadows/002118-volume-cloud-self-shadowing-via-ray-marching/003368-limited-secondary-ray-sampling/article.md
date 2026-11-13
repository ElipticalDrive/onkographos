# Limited Secondary Ray Sampling

Also called: Limited Secondary Light Sampling, Secondary Ray Optimization

## Overview
Limited Secondary Ray Sampling is an optimization strategy for computing volumetric self-shadowing in distance-field ray marching. It reduces the computational cost of secondary ray traces by limiting their maximum travel distance and adaptively reducing the number of samples based on primary ray depth and local density [S1, S1, unofficial]. Unlike global shadowing methods, it restricts self-shadowing to a local neighborhood, trading long-range accuracy for performance [S1, unofficial][S2].

## How it works
The method augments standard primary ray marching (view direction) with secondary ray marching (light direction) to estimate transmittance and lighting.

**Inputs and Outputs:**
- Input: Primary ray origin, direction, and accumulated density along the primary ray.
- Input: Light direction and scene size parameters.
- Output: Shadowed radiance contribution for the current primary sample.

**Data Structures:**
- Primary Ray State: Accumulated density $\tau_p$, current position $P$, step size $h_p$.
- Secondary Ray State: Accumulated density $\tau_s$, current position $Q$, step size $h_s$.
- Parameters:
  - $D_{max}$: Maximum secondary ray distance. Typical value: 1000 meters [S1, unofficial].
  - $N_{sec}$: Number of secondary samples. Adaptive based on primary distance and density [S1, unofficial].
  - $\tau_{thresh}$: Primary ray termination threshold. When $\tau_p \ge \tau_{thresh}$, the cloud is treated as fully opaque [S1, unofficial].

**Algorithm Steps:**
1. **Primary Ray Marching:**
   - March along the view direction.
   - At each primary sample point $P_i$:
     - If accumulated density $\tau_p \ge \tau_{thresh}$, terminate primary ray. The point is fully opaque; return black or base color with no transmission [S1, unofficial].
     - Calculate adaptive secondary sample count $N_{sec}$:
       - Reduce $N_{sec}$ for primary samples far from the camera (distance $d$). A logarithmic reduction is suggested: $N_{sec} \propto \log(d)$ or similar decay [S1, unofficial].
       - Reduce $N_{sec}$ for primary samples with low density $\rho(P_i)$. Low density implies transparency, so lower lighting accuracy is acceptable [S1, unofficial].
     - Initialize secondary ray at $P_i$ towards light direction $L$.
     - Set secondary ray max distance $D_{max}$ [S1, unofficial].
     - Perform $N_{sec}$ secondary steps.

2. **Secondary Ray Marching (Light Sampling):**
   - For each secondary step $j$ from $0$ to $N_{sec}-1$:
     - Move secondary ray position $Q_j$ along $L$ by step size $h_s$.
     - If distance from $P_i$ exceeds $D_{max}$, stop secondary ray [S1, unofficial].
     - Sample density $\rho(Q_j)$ at current position.
     - Accumulate optical depth: $\tau_s += \rho(Q_j) \cdot h_s$.
     - Apply importance weighting $w_j$ to fade out distant samples. This avoids hard margins in the lighting calculation. A linear or exponential fade based on distance from $P_i$ is used [S1, unofficial].
       - Example weight: $w_j = \max(0, 1 - \frac{\|Q_j - P_i\|}{D_{max}})$ or similar smooth decay [S1, unofficial].

3. **Lighting Calculation:**
   - Compute transmittance $T = e^{-\tau_s}$.
   - Weighted sum of light contributions using weights $w_j$ and transmittance.
   - Combine with primary ray color.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def limited_secondary_ray_sampling(primary_origin, primary_dir, light_dir, 
                                   density_field, D_max=1000.0, tau_thresh=5.0):
    """
    Perform primary ray march with adaptive secondary light sampling.
    """
    # Primary ray parameters
    primary_pos = primary_origin
    primary_dir = normalize(primary_dir)
    primary_step = 1.0  # Example step size
    accumulated_density_p = 0.0
    
    # Secondary ray parameters
    secondary_step = 1.0  # Example step size
    
    while accumulated_density_p < tau_thresh:
        # Sample density at current primary position
        rho_p = density_field.sample(primary_pos)
        
        # Adaptive secondary sample count
        # Distance from camera (assume primary_origin is camera)
        dist_from_cam = length(primary_pos - primary_origin)
        
        # Base number of samples, reduced logarithmically with distance
        # and linearly with low density
        base_samples = 16
        dist_factor = max(1.0, log(dist_from_cam + 1.0))
        density_factor = max(0.1, rho_p / 10.0) # Normalize density
        num_secondary_samples = max(1, int(base_samples / (dist_factor * density_factor)))
        
        # Secondary ray march towards light
        secondary_pos = primary_pos
        accumulated_density_s = 0.0
        weighted_light = 0.0
        
        for i in range(num_secondary_samples):
            # Move secondary ray
            secondary_pos += light_dir * secondary_step
            
            # Check max distance
            dist_from_primary = length(secondary_pos - primary_pos)
            if dist_from_primary > D_max:
                break
            
            # Sample density
            rho_s = density_field.sample(secondary_pos)
            accumulated_density_s += rho_s * secondary_step
            
            # Importance weighting to fade out distant samples
            # Linear fade from 1.0 at primary_pos to 0.0 at D_max
            weight = max(0.0, 1.0 - (dist_from_primary / D_max))
            
            # Accumulate weighted transmittance contribution
            # Simplified: weight * exp(-accumulated_density_s)
            # Note: In practice, this might be integrated more carefully
            transmittance = exp(-accumulated_density_s)
            weighted_light += weight * transmittance
        
        # Normalize weighted light by sum of weights if necessary
        # For simplicity, assume weight sum is normalized or used as is
        shadow_factor = weighted_light / num_secondary_samples if num_secondary_samples > 0 else 1.0
        
        # Compute color contribution (example)
        color = rho_p * shadow_factor * light_color
        
        # Advance primary ray
        accumulated_density_p += rho_p * primary_step
        primary_pos += primary_dir * primary_step
        
        yield color

    # If terminated by opacity
    return black_color
```

**Edge Cases and Pitfalls:**
- **Hard Margins:** Failing to fade out secondary samples can cause visible artifacts at the edge of the shadow volume [S1, unofficial].
- **Precision:** Accumulated density can overflow if not clamped or if step sizes are too large [G].
- **Boundary:** Secondary rays may exit the volume prematurely. Ensure density field returns 0 outside bounds [G].

## Subtopics

### Ray Marched Volume Shadow
A specific implementation mode in Unreal Engine's Volumetric Cloud system that uses secondary ray marching for shadowing. It produces sharp, colored shadows but is limited in trace distance due to sample constraints [S2]. It is more expensive than Beer Shadow Maps but supports ground-to-sky-to-space transitions and provides volumetric self-shadow color [S2]. The cost is driven by the `Shadow View Sample Count Scale` parameter on the Volumetric Cloud component [S2]. A key pitfall is the limited distance shadows can be traced [S2].

## Operating characteristics
- **Time Complexity:** $O(N_p \cdot N_s)$, where $N_p$ is the number of primary samples and $N_s$ is the average number of secondary samples. $N_s$ is adaptive and typically much smaller than a full secondary march [S1, unofficial].
- **Memory Complexity:** $O(1)$ per sample, assuming density field access is $O(1)$ [G].
- **Performance:** Significantly faster than full secondary ray marching due to distance and sample count limits [S1, unofficial]. On console platforms (Xbox One, PS4), Beer Shadow Maps are often recommended over Ray Marched shadows due to cost, though Ray Marched provides better quality [S2].
- **Strengths:** Local self-shadowing is accurate and colored; performance is manageable for real-time [S1, unofficial][S2].
- **Limitations:** Shadows do not extend beyond $D_{max}$ (e.g., 1000m); long-range shadows require alternative methods like Beer Shadow Maps [S1, unofficial][S2].

## Background
The method originates from practical implementations of efficient volumetric clouds in real-time rendering, particularly in the context of three.js and Unreal Engine's Volumetric Cloud system [S1, unofficial][S2]. It addresses the high cost of secondary ray marching for self-shadowing by introducing locality and adaptivity [S1, unofficial].

## Sources
[S1] Efficient volumetric clouds - Resources - three.js forum, https://discourse.threejs.org/t/efficient-volumetric-clouds/66067, fetched 2026-10-29 (unofficial)
[S2] Volumetric Cloud Component in Unreal Engine  | Unreal Engine 5.8 Documentation | Epic Developer Community, https://dev.epicgames.com/documentation/en-us/unreal-engine/volumetric-cloud-component-in-unreal-engine, fetched 2026-10-30
