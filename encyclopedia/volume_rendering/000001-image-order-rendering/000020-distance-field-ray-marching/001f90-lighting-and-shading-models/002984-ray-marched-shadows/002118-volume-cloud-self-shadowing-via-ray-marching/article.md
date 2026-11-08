# Volume Cloud Self-Shadowing via Ray Marching

Also called: Self-Shadowing via Ray Marching, Transmittance-based Self-Shadowing

## Overview
This method calculates self-shadowing for volumetric clouds by performing an additional ray march towards the light source at each sample point to compute transmittance [S1, unofficial]. It is a direct, image-order technique that avoids the precision limitations of shadow maps and the approximation errors of transmittance function mapping [S1, unofficial]. While it produces richer details than map-based alternatives, it incurs higher computational costs due to the secondary ray traversal [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** Current sample position $P$, view direction, light direction $L$, accumulated density along the view ray, and cloud density field.
- **Output:** Self-shadowing factor (transmittance) $T$ for the current sample.

**Data Structures:**
- **Shadow Ray State:** Contains current position, step size, and accumulated transmittance.
- **Step Count:** Integer $N_{shadow}$, typically 6 steps on PC/Console platforms [S1, unofficial].

**Algorithm:**
1. **Initialize Shadow Ray:** Start a secondary ray at the current volume sample position $P$ in the direction of the light source $L$. Initialize accumulated transmittance $T_{acc} = 1.0$ [S1, unofficial].
2. **Iterate Steps:** For $i$ from 0 to $N_{shadow}-1$:
   - Calculate step position: $P_{step} = P + i \cdot \Delta s \cdot L$, where $\Delta s$ is the step size derived from the primary ray's step count or a fixed value [S1, unofficial].
   - Sample the cloud density $\sigma(P_{step})$ from the volume texture [S1, unofficial].
   - Update transmittance: $T_{acc} = T_{acc} \cdot e^{-\sigma(P_{step}) \cdot \Delta s}$ [S1, unofficial].
   - If $T_{acc}$ drops below a negligible threshold (e.g., $10^{-4}$), break the loop early [G].
3. **Finalize:** The self-shadowing factor is $T = T_{acc}$ [S1, unofficial].

**Parameters:**
- **Shadow Steps ($N_{shadow}$):** Approximately 6 steps on PC/Console [S1, unofficial].
- **Step Size:** Determined by the primary ray's step size or scene scale.

**Pitfalls:**
- **Insufficient Steps:** Cause noise in the shadowed regions [S1, unofficial].
- **Too Short Steps:** Cause brightness artifacts (overly bright shadows) due to undersampling the density field [S1, unofficial].
- **Too Long Steps:** Cause noise due to aliasing in the density sampling [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_self_shadow(position, light_direction, density_field, num_steps=6):
    """
    Computes self-shadowing transmittance by ray marching towards the light.
    
    Args:
        position: Current 3D sample position.
        light_direction: Normalized vector towards the light source.
        density_field: Function or texture lookup for cloud density at a 3D point.
        num_steps: Number of steps for the shadow ray march (default 6).
        
    Returns:
        float: Transmittance value between 0.0 (fully shadowed) and 1.0 (fully lit).
    """
    # Estimate step size based on primary ray or scene scale. 
    # Not given in sources; assume a fixed step size or derived from primary ray step.
    step_size = 1.0  # Placeholder: In practice, this should match the primary ray's spatial resolution or be scaled appropriately.
    
    transmittance = 1.0
    current_pos = position
    
    for i in range(num_steps):
        # Move along the light direction
        current_pos = position + light_direction * (step_size * (i + 1))
        
        # Sample density at the new position
        density = density_field.sample(current_pos)
        
        # Update transmittance using Beer-Lambert law
        # T_new = T_old * exp(-density * step_size)
        transmittance = transmittance * math.exp(-density * step_size)
        
        # Early exit if fully shadowed
        if transmittance < 1e-4:
            return 0.0
            
    return transmittance
```

## Subtopics

### Limited Secondary Ray Sampling
A variant that optimizes the cost of self-shadowing by limiting the distance and number of samples for secondary rays [S2, unofficial].
- **Step:** Use a linear scale for sample distribution along secondary rays [S2, unofficial]. Limit the maximum distance these rays travel to reduce computational cost [S2, unofficial]. Fade out the importance of samples further away along the secondary ray to avoid hard margins in the lighting calculation [S2, unofficial]. Terminate the primary ray sampling when the accumulated density reaches a threshold, treating the cloud as fully opaque [S2, unofficial].
- **Parameter:** Secondary ray max distance is approximately 1000 meters, depending on scene size, limiting self-shadowing to a local area [S2, unofficial].
- **Optimization:** Reduce the number of secondary samples for primary samples that are far away; this is effective with minimal quality impact [S2, unofficial]. Reduce the number of secondary samples for primary samples with low density, as the lighting accuracy is lower but acceptable because the sample is transparent [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N_{primary} \cdot N_{shadow})$, where $N_{primary}$ is the number of samples along the view ray and $N_{shadow}$ is the number of shadow steps [S1, unofficial].
- **Memory Complexity:** $O(1)$ additional memory per sample, assuming the density field is accessed via texture lookup [S1, unofficial].
- **Performance:** Significantly more expensive than shadow map or transmittance function mapping methods [S1, unofficial]. On PC/Console, ~6 steps are typical [S1, unofficial].
- **Strengths:** Richer details than shadow maps due to lack of precision limits; no artifacts from shadow map resolution [S1, unofficial].
- **Limitations:** High computational cost; prone to noise if steps are insufficient or too long [S1, unofficial].
- **When to Use:** When visual fidelity of self-shadowing is critical and performance budget allows for secondary ray marching [S1, unofficial].
- **When Not to Use:** When performance is constrained or when global shadows (clouds casting shadows on other objects) are required, in which case shadow mapping or transmittance function mapping may be preferred [S1, unofficial].

## Background
Originated from techniques used in real-time volumetric cloud rendering, particularly in games like *Red Dead Redemption 2* and *Horizon Zero Dawn* which use ray marching for single scattering [S1, unofficial]. The specific approach of using ~6 steps for self-shadowing is documented in industry implementations [S1, unofficial]. Transmittance Function Mapping (TFM) and Beer's Shadow Map are alternative approaches discussed in the same context [S1, unofficial].

## Sources
[S1] 体积云效果的实现，游戏世界的云合雾集, https://leihuo.163.com/institution/news/20230719/38423_1099592.html, fetched 2026-10-30 (unofficial)
[S2] Efficient volumetric clouds - Resources - three.js forum, https://discourse.threejs.org/t/efficient-volumetric-clouds/66067, fetched 2026-10-29 (unofficial)
