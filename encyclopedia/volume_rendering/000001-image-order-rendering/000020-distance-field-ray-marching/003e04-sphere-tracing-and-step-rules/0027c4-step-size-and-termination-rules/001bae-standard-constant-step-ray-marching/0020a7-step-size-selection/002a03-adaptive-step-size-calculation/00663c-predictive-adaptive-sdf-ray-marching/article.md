# Predictive Adaptive SDF Ray Marching
Also called: Raytracer 30
## Overview
Predictive Adaptive SDF Ray Marching is a ray marching algorithm for Signed Distance Fields (SDFs) that determines step size using a linear prediction based on the previous two distance estimates, scaled by a reduction factor to prevent overshooting [S1, unofficial]. It differs from standard constant step methods by adapting to local geometry curvature and from other adaptive methods by relying on temporal continuity of distance estimates rather than cone sampling or external data structures for step prediction [S1, unofficial].
## How it works
**Inputs and Outputs:**
- Input: Ray $r$ (origin $O$, direction $D$), SDF function $SDF(P)$, maximum step count `stepcount`, reduction factor `STEP_SIZE_REDUCTION` (typically 0.87) [S1, unofficial].
- Output: A `vec2` containing the hit distance $t$ and the final step size used [S1, unofficial].

**Data Structures:**
- Variables:
  - $t$: Current ray distance along the ray (float).
  - $ft$: SDF value at current position $t$ (float).
  - $pft$: Previous SDF value $ft$ from the prior iteration (float).
  - $dt$: Calculated step size for the current iteration (float).
  - $nft$: New SDF value at the predicted position (float).

**Steps:**
1. Initialize $t = 0$.
2. Evaluate the SDF at the ray origin: $ft = SDF(RAY(r, t))$.
3. Initialize previous distance $pft = 0$.
4. Initialize step size $dt = 0$.
5. Loop for $i$ from 0 to `stepcount`:
   a. Calculate the predicted step size $dt$ using the formula:
      $$dt = ft \times \left(1 - \text{STEP\_SIZE\_REDUCTION} \times \frac{pft - ft - dt}{pft - ft + dt}\right)$$
      [S1, unofficial]
   b. Evaluate the SDF at the new position: $nft = SDF(RAY(r, t + dt))$ [S1, unofficial].
   c. Check for overshooting or invalid step:
      - If $dt < ft + nft$:
        - Update $ft = nft$.
        - Update $t = t + dt$.
      - Else:
        - Reduce step size to the current distance estimate: $dt = ft$.
        - Update $t = t + dt$.
        - Re-evaluate SDF at the new position: $ft = SDF(RAY(r, t))$.
   d. Update $pft$ for the next iteration: $pft = ft$ (Note: The source implies $pft$ tracks the previous $ft$; in the loop, $pft$ should be updated to the $ft$ value *before* it was potentially updated in step c, or effectively the $ft$ from the start of the iteration. Based on the formula structure $pft - ft$, $pft$ is the SDF value from the previous step's end, and $ft$ is the current step's start. The source code logic suggests $pft$ is the $ft$ from the previous iteration. Let's trace: Start: $ft_0$. Iter 1: $pft=0, ft=ft_0$. Calc $dt$. Get $nft$. If valid, $ft \leftarrow nft, t \leftarrow t+dt$. Next iter: $pft$ should be old $ft$. The source says "pft = 0" init. It does not explicitly show $pft$ update in the loop text, but logically $pft$ must be the $ft$ from the previous iteration. We will assume $pft$ is updated to the $ft$ value at the end of the successful step or the re-evaluated $ft$).
      *Correction based on standard predictive algorithms and source context*: The variable $pft$ represents the SDF value at the position of the previous step. In the first iteration, it is 0. After a step, $pft$ becomes the $ft$ of that step.
6. Return `vec2(t, dt)` [S1, unofficial].

**Parameters:**
- `STEP_SIZE_REDUCTION`: A constant float value of 0.87 [S1, unofficial].

**Edge Cases and Pitfalls:**
- Division by zero: The denominator $pft - ft + dt$ could be zero if $pft + dt = ft$. This requires a guard or epsilon check [G].
- Overshooting: The condition $dt < ft + nft$ handles cases where the linear prediction would jump past the surface or into a region where the distance estimate is invalid. If this condition fails, the step is clamped to $ft$ [S1, unofficial].
- Initial Step: The first step uses $pft=0$, which may result in a large initial step if $ft$ is large, potentially skipping small features [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def predictive_adaptive_sdf_ray_marching(ray_origin, ray_direction, sdf_func, step_count=100, reduction_factor=0.87):
    t = 0.0
    ft = sdf_func(ray_origin + ray_direction * t)
    pft = 0.0
    dt = 0.0
    
    for i in range(step_count):
        # Calculate predicted step size
        # Guard against division by zero
        denominator = pft - ft + dt
        if abs(denominator) < 1e-6:
            dt = ft
        else:
            dt = ft * (1.0 - reduction_factor * (pft - ft - dt) / denominator)
        
        # Evaluate SDF at predicted position
        nft = sdf_func(ray_origin + ray_direction * (t + dt))
        
        # Check for overshooting
        if dt < ft + nft:
            # Valid step
            pft = ft
            ft = nft
            t += dt
        else:
            # Overshoot or invalid, clamp step to current distance estimate
            dt = ft
            t += dt
            ft = sdf_func(ray_origin + ray_direction * t)
            pft = ft # Update pft for next iteration as the current ft
            
    return t, dt
```
## Subtopics
## Operating characteristics
**Time and Memory Complexity:**
- Time: $O(N)$ where $N$ is the number of steps, which is typically fewer than constant step marching for smooth surfaces due to adaptive sizing, but more expensive per step due to the prediction calculation [S1, unofficial].
- Memory: $O(1)$ auxiliary memory [S1, unofficial].

**Measured Performance:**
- Not given in the sources.

**Strengths and Limitations:**
- Strengths: Adapts to local geometry, potentially reducing step count for smooth surfaces [S1, unofficial].
- Limitations: Relies on the assumption that distance estimates change linearly, which may fail for highly curved or complex geometries [G]. The reduction factor is hardcoded [S1, unofficial].

**When to Use:**
- Use for smooth SDFs where step size adaptation improves performance [S1, unofficial].
- Not suitable for highly discontinuous or noisy SDFs [G].

## Background
The method originates from the "Interactive-SDF-Renderer" project by Csabix, presented as an unofficial implementation [S1, unofficial]. It is also known as "Raytracer 30" [S1, unofficial].
## Sources
[S1] Csabix/Interactive-SDF-Renderer, https://github.com/Csabix/Interactive-SDF-Renderer, fetched 2026-10-27 (unofficial)
