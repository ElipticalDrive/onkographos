# Secant Method for Surface Depth Refinement

Also called: Linear Interpolation Root Finding for SDF, Secant Refinement

## Overview
The Secant Method for Surface Depth Refinement is an iterative root-finding algorithm used to refine the depth value $d$ where a signed distance function (SDF) or occupancy logit crosses a threshold [S1, unofficial]. It estimates the next intersection point using linear interpolation between two bracketing points [S1, unofficial].

## How it works
**Inputs:**
- $ray0$: Origin point of the ray (3D vector).
- $ray\_direction$: Direction vector of the ray (normalized).
- $d_{low}$: Array of lower bound depths (floats).
- $d_{high}$: Array of upper bound depths (floats).
- $f_{low}$: Array of function values at $d_{low}$ (floats).
- $f_{high}$: Array of function values at $d_{high}$ (floats).
- $n_{secant\_steps}$: Number of refinement iterations (integer).
- $decoder$: Function evaluating the neural field.
- $c$: Camera parameters or context for the decoder.
- $logit\_tau$: Threshold value for the occupancy logit.

**Outputs:**
- $d_{pred}$: Refined depth values where the root is estimated.

**Data Structures:**
- Arrays $d_{low}, d_{high}, f_{low}, f_{high}, d_{pred}$ are parallel vectors of size $N$ (number of rays), supporting batched processing [S1, unofficial].

**Steps:**
1. **Initialization**: Compute the initial prediction $d_{pred}$ for all rays using linear interpolation between the bracketing points:
   $$ d_{pred} = - f_{low} \cdot \frac{d_{high} - d_{low}}{f_{high} - f_{low}} + d_{low} $$
   [S1, unofficial]

2. **Iteration**: For each iteration $i$ from $0$ to $n_{secant\_steps} - 1$:
   a. **Ray Casting**: Compute the 3D point $p_{mid}$ for each ray:
      $$ p_{mid} = ray0 + d_{pred} \cdot ray\_direction $$
      [S1, unofficial]
   b. **Evaluation**: Evaluate the function $f_{mid}$ at $p_{mid}$ [S1, unofficial]. This involves calling the decoder with the point and context, subtracting the threshold, and disabling gradient tracking:
      $$ f_{mid} = decoder(p_{mid}, c, only\_occupancy=True) - logit\_tau $$
      [S1, unofficial]
   c. **Bracket Update**: Identify indices where $f_{mid} < 0$ as $ind_{low}$ [S1, unofficial].
      - If $ind_{low}$ contains any true values, update the lower bracket:
        $$ d_{low}[ind_{low}] = d_{pred}[ind_{low}] $$
        $$ f_{low}[ind_{low}] = f_{mid}[ind_{low}] $$
        [S1, unofficial]
      - Identify indices where $f_{mid} \ge 0$ (i.e., $ind_{low} == 0$) [S1, unofficial]. If these exist, update the upper bracket:
        $$ d_{high}[ind_{low}==0] = d_{pred}[ind_{low}==0] $$
        $$ f_{high}[ind_{low}==0] = f_{mid}[ind_{low}==0] $$
        [S1, unofficial]
   d. **Recalculation**: Update $d_{pred}$ using the new bracketing values:
      $$ d_{pred} = - f_{low} \cdot \frac{d_{high} - d_{low}}{f_{high} - f_{low}} + d_{low} $$
      [S1, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def secant_refinement(ray0, ray_direction, d_low, d_high, f_low, f_high, 
                      n_secant_steps, decoder, c, logit_tau):
    """
    Refine surface depth using the Secant Method.
    
    Args:
        ray0: List of (x, y, z) tuples, ray origins.
        ray_direction: List of (dx, dy, dz) tuples, ray directions.
        d_low: List of floats, lower depth bounds.
        d_high: List of floats, upper depth bounds.
        f_low: List of floats, function values at d_low.
        f_high: List of floats, function values at d_high.
        n_secant_steps: int, number of iterations.
        decoder: Function(p, c, only_occupancy=True) -> float.
        c: Context/camera parameters.
        logit_tau: float, threshold.
        
    Returns:
        d_pred: List of floats, refined depths.
    """
    N = len(d_low)
    # Step 1: Initialize prediction
    denom = [f_high[i] - f_low[i] for i in range(N)]
    # Guard against zero denominator [G]
    denom = [d if d != 0.0 else 1e-8 for d in denom]
    d_pred = [-f_low[i] * ((d_high[i] - d_low[i]) / denom[i]) + d_low[i] for i in range(N)]
    
    for _ in range(n_secant_steps):
        # Step 2a: Compute 3D points
        p_mid = [(ray0[i][0] + d_pred[i] * ray_direction[i][0],
                  ray0[i][1] + d_pred[i] * ray_direction[i][1],
                  ray0[i][2] + d_pred[i] * ray_direction[i][2]) for i in range(N)]
        
        # Step 2b: Evaluate function without gradients
        f_mid = [decoder(p_mid[i], c, only_occupancy=True) - logit_tau for i in range(N)]
            
        # Step 2c: Update brackets
        ind_low = [i for i in range(N) if f_mid[i] < 0]
        
        # Update lower bracket where f_mid < 0
        for i in ind_low:
            d_low[i] = d_pred[i]
            f_low[i] = f_mid[i]
            
        # Update upper bracket where f_mid >= 0
        ind_high = [i for i in range(N) if f_mid[i] >= 0]
        for i in ind_high:
            d_high[i] = d_pred[i]
            f_high[i] = f_mid[i]
            
        # Step 2d: Recalculate prediction
        denom = [f_high[i] - f_low[i] for i in range(N)]
        denom = [d if d != 0.0 else 1e-8 for d in denom]
        d_pred = [-f_low[i] * ((d_high[i] - d_low[i]) / denom[i]) + d_low[i] for i in range(N)]
        
    return d_pred
```

**Parameters:**
- $n_{secant\_steps}$: Integer specifying the number of refinement iterations. Typical value: 8 [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Division by Zero**: If $f_{high} == f_{low}$, the denominator in the interpolation formula becomes zero. A small epsilon or fallback to bisection is required [G].
- **Non-Bracketing**: The method assumes $f_{low} < 0$ and $f_{high} > 0$ (or vice versa). If the initial segment does not bracket a root, the method may diverge or converge to a non-root extremum [G].

## Subtopics

### [Ray Marching with Secant Refinement](1042-ray-marching-with-secant-refinement.md)
A procedure to detect surface points along rays by evaluating a neural decoder at proposal points, identifying sign changes in occupancy logits, and refining the depth using the Secant Method for Surface Depth Refinement. It sets itself apart from its siblings by combining coarse uniform sampling with a specific sign-change detection logic that identifies negative-to-positive transitions, followed by the Secant Method for Surface Depth Refinement to pinpoint the zero-crossing. It builds on [Secant Method for Surface Depth Refinement](996-secant-method-for-surface-depth-refinement.md) [G].

### False Position Algorithm
A root-finding algorithm combining linear interpolation and the midpoint algorithm [S2, unofficial]. It finds the exact intersection point even if the field function is not continuous [S2, unofficial]. Implementation is in `IsoMesher::intersect_xaxis()`, `IsoMesher::intersect_yaxis()`, and `IsoMesher::intersect_zaxis()` [S2, unofficial]. The algorithm used is determined by the preprocessor symbol `FALSE_POSITION` [S2, unofficial]. It is no worse than the midpoint algorithm in terms of iterations and usually better, making it the preferred choice for finding exact intersection points [S2, unofficial].

## Operating characteristics
- **Time Complexity**: $O(n_{secant\_steps} \cdot N)$, where $N$ is the number of rays [G]. Each step involves a decoder evaluation and vector operations [S1, unofficial].
- **Memory Complexity**: $O(N)$ for storing the bracketing arrays and intermediate points [G].
- **Strengths**: Simple to implement, robust for monotonic functions, no derivative calculation required [G].
- **Limitations**: Can fail if the function is not continuous or if the initial brackets do not contain a root [G].
- **When to Use**: When a fast, derivative-free root-finding method is needed for smooth implicit surfaces or neural fields [G].
- **When Not to Use**: When the function is discontinuous (use False Position or Bisection) or when high precision is required with minimal iterations (use Newton's method if derivatives are available) [G].

## Background
The Secant Method is a classical root-finding algorithm that approximates the derivative using finite differences [G]. In the context of volume rendering and neural fields, it is used to refine the depth of surface intersections detected during ray marching [S1, unofficial].

## Sources
[S1] autonomousvision/differentiable_volumetric_rendering, https://github.com/autonomousvision/differentiable_volumetric_rendering, fetched 2026-10-26 (unofficial)
[S2] http://www.oocities.org/tzukkers/isosurf/isosurfaces.html, http://www.oocities.org/tzukkers/isosurf/isosurfaces.html, fetched 2026-10-30 (unofficial)
