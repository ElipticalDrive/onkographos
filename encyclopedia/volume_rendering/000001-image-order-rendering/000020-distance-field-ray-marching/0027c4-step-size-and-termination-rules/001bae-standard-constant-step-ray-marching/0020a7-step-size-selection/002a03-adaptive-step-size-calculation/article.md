# Adaptive Step Size Calculation

Also called: Cone-angle based step size, Step Size Calculation

## Overview
Adaptive Step Size Calculation determines the distance between samples during ray marching by scaling the step size based on the ray's distance from the origin and a predefined cone angle. It addresses the trade-off between rendering speed and geometric accuracy, ensuring that samples are dense enough to resolve high-frequency details near the camera while reducing computation cost further away. It differs from constant step methods by dynamically adjusting granularity based on spatial context [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:**
  - $t$: Current ray time (distance from ray origin).
  - $\alpha_{cone}$: Cone angle for volume rendering.
  - $dt_{min}$: Minimum step size.
  - $dt_{max}$: Maximum step size.
- **Output:**
  - $dt$: The calculated step size for the current ray segment.

**Data Structures:**
No complex data structures are required. The method operates on scalar values.

**Steps:**
1. Calculate the unconstrained step size by multiplying the current ray distance $t$ by the cone angle $\alpha_{cone}$.
2. Clamp the result to the range $[dt_{min}, dt_{max}]$ to prevent steps from becoming too small (causing performance issues) or too large (causing aliasing or missed details).

**Formula:**
$$ dt = \text{clamp}(t \cdot \alpha_{cone}, dt_{min}, dt_{max}) $$

Where:
- $t \ge 0$ is the distance along the ray.
- $\alpha_{cone}$ is a constant parameter representing the angular resolution or cone angle.
- $dt_{min}$ and $dt_{max}$ are scalar bounds.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def calculate_adaptive_step_size(t, cone_angle, dt_min, dt_max):
    """
    Calculate adaptive step size for ray marching.
    
    Args:
        t (float): Current ray distance from origin.
        cone_angle (float): Cone angle parameter.
        dt_min (float): Minimum allowed step size.
        dt_max (float): Maximum allowed step size.
        
    Returns:
        float: The calculated step size.
    """
    # Calculate unconstrained step size
    raw_step = t * cone_angle
    
    # Clamp to valid range
    step_size = max(dt_min, min(dt_max, raw_step))
    
    return step_size
```

**Parameters:**
- $t$: Current ray time/distance. [S1, unofficial]
- $\alpha_{cone}$: Cone angle for volume rendering. [S1, unofficial]
- $dt_{min}$: Minimum step size. [S1, unofficial]
- $dt_{max}$: Maximum step size. [S1, unofficial]

**Edge Cases and Pitfalls:**
- If $t=0$, the step size is clamped to $dt_{min}$, preventing zero-length steps. [S1, unofficial]
- Precision issues may arise if $t$ is very large and $\alpha_{cone}$ is small, potentially resulting in $dt < dt_{min}$ due to floating-point underflow before clamping. [G]

## Subtopics

### [Ray Marching with Adaptive Step Count and Cone Sampling](4808-ray-marching-with-adaptive-step-count-and-cone-sampling.md)
Extends adaptive step size by adapting the number of steps based on cone sampling, offering higher fidelity for complex volumes.

### [Adaptive Ray Marching with Temporal Reprojection](4840-adaptive-ray-marching-with-temporal-reprojection.md)
Combines adaptive step size with temporal reprojection to reduce flickering and improve stability in animated scenes.

## Operating characteristics
- **Time Complexity:** $O(1)$ per step calculation. [S1, unofficial]
- **Memory Complexity:** $O(1)$ auxiliary space. [S1, unofficial]
- **Performance:** Reduces total sample count compared to constant step size for distant rays, improving performance. However, near-field rays may require more samples if $dt_{min}$ is small. [S1, unofficial]
- **Strengths:** Balances accuracy and performance; simple to implement. [S1, unofficial]
- **Limitations:** May still produce aliasing if $\alpha_{cone}$ is not tuned correctly for the scene's frequency content. [G]
- **When to Use:** Suitable for general volume rendering where performance is a concern and the scene has varying density scales. [S1, unofficial]
- **When Not to Use:** Not suitable for scenes requiring uniform resolution regardless of distance, or where strict error bounds are required. [G]

## Background
The method is derived from techniques used in efficient ray marching for volume rendering and occlusion grids. It is commonly associated with implementations that aim to optimize rendering speed by reducing sample counts at a distance. [S1, unofficial]

## Sources
[S1] Yosshi999/faster-occgrid, https://github.com/Yosshi999/faster-occgrid, fetched 2026-10-24 (unofficial)
