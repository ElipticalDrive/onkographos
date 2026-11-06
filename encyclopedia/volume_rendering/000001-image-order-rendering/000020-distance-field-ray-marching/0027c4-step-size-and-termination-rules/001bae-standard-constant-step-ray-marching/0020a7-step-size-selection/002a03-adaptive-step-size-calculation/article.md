# Adaptive Step Size Calculation

Also called: Cone-angle based step size, Step Size Calculation

## Overview
Adaptive Step Size Calculation determines the distance between samples during ray marching by scaling the step size based on the viewing angle relative to the volume, specifically using a cone angle parameter. It solves the problem of balancing rendering speed and visual quality in atmospheric clouds, where top-down views require fewer steps than horizon views to maintain performance without sacrificing detail. It differs from standard constant step ray marching by varying the step count dynamically based on geometry orientation, and from sibling methods like [Raycasting Step Size & Aspect Ratio Normalization](3734-raycasting-step-size-aspect-ratio-normalization.md) by relying on a cone-angle heuristic rather than aspect ratio normalization for step determination.

## How it works
**Inputs and Outputs:**
- Input: `t` (current ray time/distance along ray), `cone_angle` (cone angle for volume rendering), `dt_min` (minimum step size), `dt_max` (maximum step size).
- Output: `dt` (the calculated step size for the current sample).

**Data Structures:**
No complex data structures are required; the method operates on scalar values.

**Steps:**
1. Calculate the raw step size by multiplying the current ray time `t` by the `cone_angle`.
2. Clamp the result to the range `[dt_min, dt_max]` to prevent steps that are too small (causing performance issues) or too large (causing aliasing or missing details).

**Formula:**
$$ dt = \text{clamp}(t \times \text{cone\_angle}, dt\_min, dt\_max) $$

Where:
- $t$: Current ray time.
- $\text{cone\_angle}$: Cone angle for volume rendering.
- $dt\_min$: Minimum step size.
- $dt\_max$: Maximum step size.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def calculate_adaptive_step_size(t, cone_angle, dt_min, dt_max):
    """
    Calculate adaptive step size based on cone angle.
    
    Args:
        t (float): Current ray time.
        cone_angle (float): Cone angle for volume rendering.
        dt_min (float): Minimum step size.
        dt_max (float): Maximum step size.
        
    Returns:
        float: The calculated step size.
    """
    raw_step = t * cone_angle
    # Clamp the step size to the valid range
    step_size = max(dt_min, min(raw_step, dt_max))
    return step_size
```

**Parameters:**
- `cone_angle`: Not given in the sources.
- `dt_min`: Not given in the sources.
- `dt_max`: Not given in the sources.

**Edge Cases and Pitfalls:**
- If `t` is zero, the step size will be clamped to `dt_min`.
- If `cone_angle` is zero, the step size will be clamped to `dt_min`.
- Precision issues may arise if `t` is very large or very small, but clamping mitigates extreme values.

## Subtopics

### [Ray Marching with Adaptive Step Count and Cone Sampling](4808-ray-marching-with-adaptive-step-count-and-cone-sampling.md)
This method performs volume rendering of volumetric clouds by adaptively selecting the number of ray march steps based on the viewing angle and using cone sampling to approximate light transport within the volume. It solves the problem of balancing rendering performance and visual quality in real-time atmospheric clouds, where top-down views require fewer steps than horizon views, and single-sample lighting causes banding artifacts. It differs from standard constant step ray marching by varying the step count dynamically and from sibling methods by using a fixed kernel of 6 offset vectors rotated towards the light source for density estimation along the light path.

### [Adaptive Ray Marching with Temporal Reprojection](4840-adaptive-ray-marching-with-temporal-reprojection.md)
This method performs volume rendering by combining adaptive ray marching with temporal accumulation to reduce noise and flickering in animated scenes. It builds on [Adaptive Step Size Calculation](4671-adaptive-step-size-calculation.md) by using density thresholds to vary step size, and extends standard ray marching by reprojection of the previous frame's color and alpha into the current frame's screen space. It is distinct from sibling methods like [Ray Marching with Adaptive Step Count and Cone Sampling](4808-ray-marching-with-adaptive-step-count-and-cone-sampling.md) by relying on spatial reprojection rather than cone-based filtering for temporal stability.

## Operating characteristics
- **Time Complexity:** $O(1)$ per sample, as it involves a constant number of arithmetic operations.
- **Memory Complexity:** $O(1)$, as it requires no additional memory beyond the input parameters.
- **Measured Performance:** Not given in the sources.
- **Scaling:** The method scales linearly with the number of samples, but the adaptive nature reduces the total number of samples needed for high-quality rendering in certain viewing angles.
- **Strengths:** Improves performance by reducing sample count in favorable viewing angles (e.g., top-down) while maintaining quality in challenging angles (e.g., horizon).
- **Limitations:** Relies on the accuracy of the `cone_angle` parameter; incorrect values can lead to suboptimal step sizes.
- **When to Use:** Suitable for real-time volume rendering of atmospheric clouds where performance is critical and viewing angles vary.
- **When Not to Use:** Not suitable for scenes where viewing angles are fixed or where high precision is required regardless of angle.

## Background
The method originates from the work of Yosshi999, as documented in the `faster-occgrid` repository. It is an unofficial implementation, likely derived from practical experience in optimizing volume rendering for real-time applications.

## Sources
[S1] Yosshi999/faster-occgrid, https://github.com/Yosshi999/faster-occgrid, fetched 2026-10-24 (unofficial)
