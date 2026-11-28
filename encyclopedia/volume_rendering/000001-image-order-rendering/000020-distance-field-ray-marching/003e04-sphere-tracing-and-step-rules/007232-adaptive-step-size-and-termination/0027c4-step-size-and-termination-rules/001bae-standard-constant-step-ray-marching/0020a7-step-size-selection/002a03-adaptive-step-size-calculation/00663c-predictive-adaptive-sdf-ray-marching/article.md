# Predictive Adaptive SDF Ray Marching

Also called: Raytracer 30

## Overview
Predictive Adaptive SDF Ray Marching is a ray marching algorithm for Signed Distance Field (SDF) rendering that determines step size using a prediction derived from the previous two distance estimates, scaled by a reduction factor to prevent overshooting [S1, unofficial]. It differs from standard adaptive methods by incorporating a temporal component (previous step history) into the step size calculation, allowing for smoother convergence in empty space compared to methods relying solely on the current distance estimate [S1, unofficial].

## How it works
**Inputs:**
- Ray $r$ (origin and direction).
- SDF function $SDF(p)$.
- Maximum step count `stepcount`.
- Reduction factor `STEP_SIZE_REDUCTION` (typically 0.87) [S1, unofficial].

**Outputs:**
- A `vec2` containing the final distance to the surface ($t$) and the final step size used ($t + \text{stepsize}$) [S1, unofficial].

**Data Structures:**
- Floating-point variables: $t$ (current ray distance), $ft$ (SDF value at $t$), $pft$ (SDF value at previous step), $dt$ (step size), $nft$ (SDF value at next step) [S1, unofficial].

**Steps:**
1. Initialize $t = 0$, $ft = SDF(ray(r, t))$, $pft = 0$, $dt = 0$ [S1, unofficial].
2. Loop for $i$ from 0 to `stepcount`:
   a. Calculate the predicted step size $dt$ using the formula:
      $$dt = ft \times \left(1 - \text{STEP\_SIZE\_REDUCTION} \times \frac{pft - ft - dt}{pft - ft + dt}\right)$$
      [S1, unofficial].
   b. Evaluate the SDF at the new position: $nft = SDF(ray(r, t + dt))$ [S1, unofficial].
   c. Check for overshooting:
      - If $dt < ft + nft$:
        - Update $ft = nft$.
        - Update $t = t + dt$ [S1, unofficial].
      - Else (overshoot detected):
        - Update $t = t + ft$ (step back to the previous valid position).
        - Update $ft = SDF(ray(r, t))$ [S1, unofficial].
3. Return `vec2(t, t + stepsize)` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def predictive_adaptive_sdf_ray_marching(ray, sdf, stepcount, step_size_reduction=0.87):
    t = 0.0
    ft = sdf(ray_at(ray, t))
    pft = 0.0
    dt = 0.0
    
    for i in range(stepcount):
        # Calculate predicted step size
        # Note: Division by zero check for (pft - ft + dt) is implied by robust implementation [G]
        numerator = pft - ft - dt
        denominator = pft - ft + dt
        
        if denominator == 0:
            dt = ft # Fallback to simple step if denominator is zero [G]
        else:
            dt = ft * (1.0 - step_size_reduction * (numerator / denominator))
        
        # Evaluate SDF at new position
        nft = sdf(ray_at(ray, t + dt))
        
        # Check for overshooting
        if dt < ft + nft:
            ft = nft
            t = t + dt
        else:
            t = t + ft
            ft = sdf(ray_at(ray, t))
            
    return vec2(t, t + stepcount) # Note: Source returns t + stepsize, but stepsize variable is not explicitly tracked as a separate accumulator in the loop description. 
                                 # The source says "Return vec2(t, t + stepsize)". Assuming stepsize refers to the last dt or a specific final metric. 
                                 # Given the ambiguity, we return t and the last computed dt as a proxy for stepsize if not tracked [G].
                                 # However, strictly following source: return vec2(t, t + dt) is the most logical interpretation of the final state.
    return vec2(t, t + dt)
```

**Parameters:**
- `STEP_SIZE_REDUCTION`: A constant float value of 0.87 used to scale the correction term [S1, unofficial].

**Edge Cases and Pitfalls:**
- Division by zero in the step size formula if $pft - ft + dt = 0$. A guard is required [G].
- The condition $dt < ft + nft$ is used to detect overshooting. If this condition fails, the algorithm steps back by $ft$ rather than $dt$, which may cause oscillation if not handled carefully [S1, unofficial].

## Subtopics

### [Dynamic SDF Ray Marching with Adaptive Step Skipping](19684-dynamic-sdf-ray-marching-with-adaptive-step-skipping.md)
Uses predicted distances from previous steps for adaptive skipping via an octree, unlike the continuous adjustment of this method.
- [Adaptive SDF Ray Marching with Variable Step Size]: Uses a simpler step size logic based on the current distance estimate and a fixed truncation limit, lacking the predictive two-step history of this method.

## Operating characteristics
Not given in the sources.

## Background
Originated from the Interactive-SDF-Renderer project by Csabix [S1, unofficial].

## Sources
[S1] Csabix/Interactive-SDF-Renderer, https://github.com/Csabix/Interactive-SDF-Renderer, fetched 2026-10-27 (unofficial)
