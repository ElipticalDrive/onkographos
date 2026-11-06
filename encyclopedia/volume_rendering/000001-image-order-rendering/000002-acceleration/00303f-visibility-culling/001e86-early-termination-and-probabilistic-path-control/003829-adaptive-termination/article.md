# Adaptive Termination

Also called: Color-difference termination, Significant color change termination

## Overview
Adaptive termination is a visibility culling technique that terminates a volume ray early once the accumulated color change between consecutive samples falls below a user-defined threshold [S1, unofficial][S2, unofficial]. It differs from standard opacity-based early termination by using the magnitude of the color contribution ($C_{out} - C_{in}$) as the primary stopping criterion, aiming to skip samples where the visual impact is negligible [S1, unofficial][S2, unofficial].

## How it works
**Inputs:**
- Ray parameters (origin, direction, step size).
- Volume data and transfer functions (opacity $\alpha$, color $C$).
- Termination threshold $\epsilon > 0$ [S1, unofficial][S2, unofficial].

**Outputs:**
- Final pixel color $C_{final}$.
- Boolean flag indicating if the ray was terminated early.

**Data Structures:**
- Accumulated color $C_{acc}$ (initialized to 0 or background color).
- Accumulated opacity $\alpha_{acc}$ (initialized to 0).
- Previous sample color $C_{prev}$ (initialized to background or first sample).

**Steps:**
1. Initialize $C_{acc} = 0$, $\alpha_{acc} = 0$, and $C_{prev} = C_{background}$ [S1, unofficial][S2, unofficial].
2. Traverse the ray in front-to-back order [1019-early-ray-termination.md].
3. For each sample point $i$:
   a. Sample the volume to get color $C_i$ and opacity $\alpha_i$.
   b. Compute the color difference $\Delta C = |C_i - C_{prev}|$ [S1, unofficial][S2, unofficial].
   c. Update accumulated color and opacity using standard front-to-back compositing:
      $$ C_{acc} = C_{acc} + C_i \cdot \alpha_i \cdot (1 - \alpha_{acc}) $$
      $$ \alpha_{acc} = \alpha_{acc} + \alpha_i \cdot (1 - \alpha_{acc}) $$
   d. Check termination condition:
      - If $\alpha_{acc} > 1 - \epsilon$, terminate [S1, unofficial][S2, unofficial].
      - If $\Delta C < \epsilon$, terminate [S1, unofficial][S2, unofficial].
   e. Set $C_{prev} = C_i$ for the next iteration.
4. Return $C_{acc}$ as the final color.

**Parameters:**
- $\epsilon$: Small positive value controlling sensitivity. Higher values reduce rendering time but may increase artifacts; lower values reduce artifacts and produce higher quality images [S1, unofficial][S2, unofficial].

**Edge Cases:**
- If $\epsilon$ is too large, significant color transitions may be missed, leading to visual artifacts [S1, unofficial][S2, unofficial].
- If $\epsilon$ is too small, the performance benefit is reduced as more samples are processed [S1, unofficial][S2, unofficial].
- The method assumes that once opacity is very high ($> 1-\epsilon$), further color changes are insignificant [S1, unofficial][S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def adaptive_ray_cast(ray, volume, epsilon):
    c_acc = vec3(0.0)
    alpha_acc = 0.0
    c_prev = vec3(0.0)  # Background color
    
    for sample in ray.traverse(volume):
        c_sample = volume.sample_color(sample)
        alpha_sample = volume.sample_opacity(sample)
        
        # Front-to-back compositing
        weight = alpha_sample * (1.0 - alpha_acc)
        c_acc = c_acc + c_sample * weight
        alpha_acc = alpha_acc + weight
        
        # Check termination conditions
        color_diff = abs(c_sample - c_prev)
        if alpha_acc > 1.0 - epsilon or color_diff < epsilon:
            break
            
        c_prev = c_sample
        
    return c_acc
```

## Operating characteristics
- **Time Complexity:** Depends on the volume density and $\epsilon$. Higher $\epsilon$ leads to earlier termination and faster rendering [S1, unofficial][S2, unofficial].
- **Memory Complexity:** $O(1)$ per ray, as only current and previous sample data are needed.
- **Strengths:** Can skip samples where color contribution is minimal, even if opacity is not yet saturated [S1, unofficial][S2, unofficial].
- **Limitations:** May introduce artifacts if $\epsilon$ is not chosen carefully [S1, unofficial][S2, unofficial]. Relies on the assumption that small color changes are visually insignificant [S1, unofficial][S2, unofficial].

## Background
Adaptive termination was proposed as a method to optimize volume ray casting by identifying the last sample that significantly changes the ray's color [S1, unofficial][S2, unofficial]. It is part of a broader set of techniques aimed at reducing computational cost in direct volume rendering [S1, unofficial][S2, unofficial].

## Sources
[S1] Direct Volume Rendering Techniques, https://www.scribd.com/presentation/380929057/10-raycasting, fetched 2026-10-25 (unofficial)
[S2] Direct Volume Rendering (DVR): Ray-casting | PPT - SlideShare, https://www.slideshare.net/slideshow/direct-volume-rendering-dvr-raycasting/265104120, fetched 2026-10-26 (unofficial)
