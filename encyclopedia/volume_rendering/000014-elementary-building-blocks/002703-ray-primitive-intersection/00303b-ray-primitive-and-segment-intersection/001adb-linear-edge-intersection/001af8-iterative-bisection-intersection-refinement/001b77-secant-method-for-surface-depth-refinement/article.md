# Secant Method for Surface Depth Refinement

Also called: Linear Interpolation Root Finding for SDF, Secant Refinement

## Overview
The Secant Method for Surface Depth Refinement is an iterative root-finding algorithm used to refine the depth value $d$ along a ray where a signed distance function (SDF) or occupancy logit crosses a specific threshold [S1, unofficial]. It distinguishes itself from its sibling [Bisection Method for Surface Depth Refinement](1033-bisection-method-for-surface-depth-refinement.md) by using linear interpolation between two bracketing points to estimate the next probe point, rather than simply halving the interval [S1, unofficial].

## How it works
**Inputs:**
- Ray origin $ray0$ and direction $ray\_direction$ [S1, unofficial].
- Initial bracketing depths $d_{low}$ and $d_{high}$ with corresponding function values $f_{low}$ and $f_{high}$ [S1, unofficial].
- Function $f(d)$ representing the signed distance or occupancy logit minus a threshold $logit\_tau$ [S1, unofficial].
- Number of refinement iterations $n_{secant\_steps}$ [S1, unofficial].

**Outputs:**
- Refined depth $d_{pred}$ [S1, unofficial].

**Data Structures:**
- Vectors or scalars for $d_{low}, d_{high}, f_{low}, f_{high}, d_{pred}$ [S1, unofficial].
- 3D point $p_{mid}$ computed from the ray equation [S1, unofficial].

**Steps:**
1. **Initialization**: Compute the initial prediction $d_{pred}$ using linear interpolation between the bracketing points:
   $$d_{pred} = - f_{low} \cdot \frac{d_{high} - d_{low}}{f_{high} - f_{low}} + d_{low}$$
   [S1, unofficial]

2. **Iteration Loop**: For each iteration $i$ from $0$ to $n_{secant\_steps} - 1$:
   a. Compute the 3D point at the predicted depth:
      $$p_{mid} = ray0 + d_{pred} \cdot ray\_direction$$
      [S1, unofficial]
   b. Evaluate the function at $p_{mid}$. For neural implicit surfaces, this involves querying the decoder without gradient tracking:
      $$f_{mid} = decoder(p_{mid}, c, only\_occupancy=True) - logit\_tau$$
      [S1, unofficial]
   c. Update the bracketing interval based on the sign of $f_{mid}$:
      - Identify indices where $f_{mid} < 0$ as $ind_{low}$ [S1, unofficial].
      - If $ind_{low}$ has any true values, update the lower bound:
        $$d_{low}[ind_{low}] = d_{pred}[ind_{low}]$$
        $$f_{low}[ind_{low}] = f_{mid}[ind_{low}]$$
        [S1, unofficial]
      - Identify indices where $f_{mid} \ge 0$ (i.e., $ind_{low} == 0$) [S1, unofficial].
      - If these exist, update the upper bound:
        $$d_{high}[ind_{low}==0] = d_{pred}[ind_{low}==0]$$
        $$f_{high}[ind_{low}==0] = f_{mid}[ind_{low}==0]$$
        [S1, unofficial]
   d. Recalculate $d_{pred}$ using the updated bracketing values:
      $$d_{pred} = - f_{low} \cdot \frac{d_{high} - d_{low}}{f_{high} - f_{low}} + d_{low}$$
      [S1, unofficial]

3. **Return**: The final $d_{pred}$ as the refined intersection depth [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def secant_refine(ray0, ray_direction, d_low, d_high, f_low, f_high, decoder, logit_tau, n_secant_steps, c=None):
    """
    Refine surface depth using the Secant Method.
    
    Args:
        ray0: Array of ray origins.
        ray_direction: Array of ray directions.
        d_low: Array of lower bound depths.
        d_high: Array of upper bound depths.
        f_low: Array of function values at d_low.
        f_high: Array of function values at d_high.
        decoder: Function to evaluate occupancy/logits.
        logit_tau: Threshold for occupancy.
        n_secant_steps: Number of refinement iterations.
        c: Optional context parameter passed to the decoder.
        
    Returns:
        d_pred: Refined depth array.
    """
    # Step 1: Initialize prediction
    denom = f_high - f_low
    # Guard against zero denominator [G]
    denom = np.where(denom == 0, 1.0, denom)
    d_pred = -f_low * (d_high - d_low) / denom + d_low
    
    for _ in range(n_secant_steps):
        # Step 2a: Compute 3D point
        p_mid = ray0 + d_pred * ray_direction
        
        # Step 2b: Evaluate function without gradients
        if c is not None:
            f_mid = decoder(p_mid, c, only_occupancy=True) - logit_tau
        else:
            f_mid = decoder(p_mid, only_occupancy=True) - logit_tau
            
        # Step 2c: Update brackets
        ind_low = f_mid < 0
        
        # Update lower bound where f_mid < 0
        d_low[ind_low] = d_pred[ind_low]
        f_low[ind_low] = f_mid[ind_low]
        
        # Update upper bound where f_mid >= 0
        ind_high = ~ind_low
        d_high[ind_high] = d_pred[ind_high]
        f_high[ind_high] = f_mid[ind_high]
        
        # Step 2d: Recalculate prediction
        denom = f_high - f_low
        # Guard against zero denominator [G]
        denom = np.where(denom == 0, 1.0, denom)
        d_pred = -f_low * (d_high - d_low) / denom + d_low
        
    return d_pred
```

**Parameters:**
- $n_{secant\_steps}$: Integer specifying the number of refinement iterations. Typical value: 8 [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Division by Zero**: If $f_{high} == f_{low}$, the denominator in the linear interpolation formula becomes zero. A guard must be implemented to handle this case, typically by falling back to the midpoint or keeping the current estimate [G].

## Subtopics

### [Ray Marching with Secant Refinement](1042-ray-marching-with-secant-refinement.md)
A procedure to detect surface points along rays by evaluating a neural decoder at proposal points, identifying sign changes in occupancy logits, and refining the depth using a root-finding method. It distinguishes itself from its siblings by combining coarse uniform sampling with a specific sign-change detection logic to locate the first entry into the volume, followed by secant refinement. It builds on [Secant Method for Surface Depth Refinement](996-secant-method-for-surface-depth-refinement.md).

### False Position Algorithm
A root-finding algorithm combining linear interpolation and the midpoint algorithm [S2, unofficial]. It finds the exact intersection point even if the field function is not continuous [S2, unofficial]. Implementation is in `IsoMesher::intersect_xaxis()`, `IsoMesher::intersect_yaxis()`, and `IsoMesher::intersect_zaxis()` [S2, unofficial]. The algorithm used is determined by the preprocessor symbol `FALSE_POSITION` [S2, unofficial]. It is no worse than the midpoint algorithm in terms of iterations and usually better, making it the preferred choice for finding exact intersection points [S2, unofficial].

### Secant Method for Grazing Point
Methods to find the grazing point (where the ray is tangent to the SDF surface) along a ray segment `[t_min, t_max]` using secant or bisection iteration on the directional derivative [S3, unofficial].
- **Secant method**: Initialize `L = ray(t_min)`, `R = ray(t_max)` [S3, unofficial]. Loop up to `utils.NUM_GRZ` iterations: compute directional derivatives `dL = dir_deriv(L, d)`, `dR = dir_deriv(R, d)` [S3, unofficial]. Compute midpoint `M = R - dR * (R - L) / (dR - dL)` [S3, unofficial]. Compute `dM = dir_deriv(M, d)` [S3, unofficial]. Update `L[dM < 0] = M`, `R[dM > 0] = M` [S3, unofficial]. Return `dr.detach(L)` [S3, unofficial].
- **Bisection method**: Initialize `L = ray(t_min)`, `R = ray(t_max)` [S3, unofficial]. Loop up to `utils.NUM_GRZ` iterations: compute `M = (L + R) / 2` [S3, unofficial]. Compute `dM = dir_deriv(M, d)` [S3, unofficial]. Update `L[dM < 0] = M[dM < 0]`, `R[dM > 0] = M[dM > 0]` [S3, unofficial]. Return `dr.detach(L)` [S3, unofficial].
- Select method based on `utils.GRZ_MODE`: 'secant' or 'bisection' [S3, unofficial].
- **Pitfall**: Gradient computation is suspended during grazing point search using `dr.suspend_grad()` to prevent differentiability issues in the iterative root-finding process [S3, unofficial].

## Operating characteristics
- **Time Complexity**: $O(n_{secant\_steps})$ function evaluations per ray [S1, unofficial].
- **Memory Complexity**: $O(1)$ additional memory per ray for storing bracketing values [S1, unofficial].
- **Strengths**: Fast convergence for smooth SDFs; simple implementation [S1, unofficial].
- **Limitations**: Requires two bracketing values; potential for division by zero if function values are equal [S1, unofficial].
- **When to Use**: When the SDF is smooth and continuous, and fast convergence is desired [S1, unofficial].
- **When Not to Use**: When the field function is discontinuous; in this case, the False Position Algorithm is preferred [S2, unofficial].

## Background
The Secant Method is a classic root-finding algorithm that approximates the root of a function by using linear interpolation between two points [S1, unofficial]. It is widely used in computer graphics for surface intersection refinement, particularly in differentiable rendering and neural implicit surface reconstruction [S1, unofficial][S3, unofficial].

## Sources
[S1] autonomousvision/differentiable_volumetric_rendering, https://github.com/autonomousvision/differentiable_volumetric_rendering, fetched 2026-10-26 (unofficial)
[S2] http://www.oocities.org/tzukkers/isosurf/isosurfaces.html, http://www.oocities.org/tzukkers/isosurf/isosurfaces.html, fetched 2026-10-30 (unofficial)
[S3] zichenwang01/relaxed-boundary, https://github.com/zichenwang01/relaxed-boundary, fetched 2026-10-27 (unofficial)
[S4] YueJiang-nj/CVPR2020-SDFDiff, https://github.com/YueJiang-nj/CVPR2020-SDFDiff, fetched 2026-10-27 (unofficial)
