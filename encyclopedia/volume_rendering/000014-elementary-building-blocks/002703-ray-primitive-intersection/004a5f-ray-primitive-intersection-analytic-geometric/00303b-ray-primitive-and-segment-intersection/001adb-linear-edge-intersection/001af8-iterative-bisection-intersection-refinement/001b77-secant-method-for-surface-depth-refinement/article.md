# Secant Method for Surface Depth Refinement

Also called: Linear Interpolation Root Finding for SDF, Secant Refinement

## Overview
The Secant Method for Surface Depth Refinement is an iterative root-finding algorithm used to refine the depth value $d$ along a ray where a signed distance function (SDF) or occupancy logit crosses a specific threshold [S1, unofficial]. It solves the problem of locating the precise zero-crossing of a scalar field by using linear interpolation between two bracketing points to estimate the next candidate point [S1, unofficial]. It sets itself apart from its sibling, the [Bisection Method for Surface Depth Refinement](1033-bisection-method-for-surface-depth-refinement.md), by using the slope between the two bracketing points to predict the root, rather than simply halving the interval, which can converge faster for smooth fields but requires maintaining two distinct function evaluations [S1, unofficial].

## How it works
**Inputs:**
- Ray origin $ray0$ (3D vector).
- Ray direction $ray\_direction$ (normalized 3D vector).
- Initial bracketing depths $d_{low}$ and $d_{high}$ (scalars or vectors of depths) such that the function values $f_{low}$ and $f_{high}$ have opposite signs (or satisfy the bracketing condition for the specific implementation).
- A scalar field evaluation function $decoder(p, c, only\_occupancy=True)$ that returns occupancy logits or SDF values at point $p$ given camera parameters $c$ [S1, unofficial].
- A threshold $logit\_tau$ (scalar) [S1, unofficial].
- Number of refinement iterations $n_{secant\_steps}$ (integer) [S1, unofficial].

**Outputs:**
- Refined depth $d_{pred}$ (scalar or vector) representing the estimated intersection point.

**Data Structures:**
- Vectors $d_{low}, d_{high}, f_{low}, f_{high}, d_{pred}$ of size $N$ (where $N$ is the number of rays being processed in parallel).
- Indices $ind_{low}$ (boolean mask of size $N$) identifying rays where the current prediction fell below the threshold.

**Steps:**
1. **Initialization**: Compute the initial prediction $d_{pred}$ using linear interpolation (secant formula) between the initial bracketing points:
   $$ d_{pred} = - f_{low} \cdot \frac{d_{high} - d_{low}}{f_{high} - f_{low}} + d_{low} $$
   [S1, unofficial]

2. **Iteration Loop**: For $i$ from $0$ to $n_{secant\_steps} - 1$:
   a. **Sample Point**: Compute the 3D position $p_{mid}$ for each ray:
      $$ p_{mid} = ray0 + d_{pred} \cdot ray\_direction $$
      [S1, unofficial]
   b. **Evaluate Field**: Evaluate the scalar field at $p_{mid}$ without tracking gradients:
      $$ f_{mid} = decoder(p_{mid}, c, only\_occupancy=True) - logit\_tau $$
      [S1, unofficial]
   c. **Identify Bracketing Side**: Determine indices where the function value is negative:
      $$ ind_{low} = (f_{mid} < 0) $$
      [S1, unofficial]
   d. **Update Lower Bracket**: For all indices in $ind_{low}$, update the lower bound:
      $$ d_{low}[ind_{low}] = d_{pred}[ind_{low}] $$
      $$ f_{low}[ind_{low}] = f_{mid}[ind_{low}] $$
      [S1, unofficial]
   e. **Update Upper Bracket**: For all indices where $f_{mid} \ge 0$ (i.e., $ind_{low}$ is false):
      $$ d_{high}[ind_{low} == 0] = d_{pred}[ind_{low} == 0] $$
      $$ f_{high}[ind_{low} == 0] = f_{mid}[ind_{low} == 0] $$
      [S1, unofficial]
   f. **Recalculate Prediction**: Update $d_{pred}$ using the new bracketing values:
      $$ d_{pred} = - f_{low} \cdot \frac{d_{high} - d_{low}}{f_{high} - f_{low}} + d_{low} $$
      [S1, unofficial]

3. **Return**: The final $d_{pred}$ is the refined depth.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def secant_refine(ray0, ray_direction, d_low, d_high, f_low, f_high, 
                  decoder, c, logit_tau, n_secant_steps):
    """
    Refine surface depth using the Secant Method.
    
    Args:
        ray0: Tensor of shape (N, 3), ray origins.
        ray_direction: Tensor of shape (N, 3), normalized ray directions.
        d_low: Tensor of shape (N,), initial lower depth bounds.
        d_high: Tensor of shape (N,), initial upper depth bounds.
        f_low: Tensor of shape (N,), function values at d_low.
        f_high: Tensor of shape (N,), function values at d_high.
        decoder: Function that takes (point, camera_params, only_occupancy) and returns logits.
        c: Camera parameters.
        logit_tau: Threshold for zero-crossing.
        n_secant_steps: Number of iterations.
        
    Returns:
        d_pred: Tensor of shape (N,), refined depths.
    """
    # Step 1: Initialize prediction
    # Avoid division by zero if f_high == f_low, though bracketing implies opposite signs.
    # [G] Guard against division by zero: if f_high - f_low is near zero, use midpoint.
    denom = f_high - f_low
    safe_denom = torch.where(torch.abs(denom) > 1e-8, denom, torch.ones_like(denom))
    d_pred = - f_low * ((d_high - d_low) / safe_denom) + d_low
    
    for _ in range(n_secant_steps):
        # Step 2a: Compute 3D point
        p_mid = ray0 + d_pred.unsqueeze(1) * ray_direction
        
        # Step 2b: Evaluate function without gradient tracking
        # [G] Assuming decoder returns raw logits; subtract tau to get signed distance-like value
        f_mid = decoder(p_mid, c, only_occupancy=True) - logit_tau
        
        # Step 2c: Identify indices where f_mid < 0
        ind_low = f_mid < 0
        
        # Step 2d: Update lower bracket
        d_low[ind_low] = d_pred[ind_low]
        f_low[ind_low] = f_mid[ind_low]
        
        # Step 2e: Update upper bracket (where f_mid >= 0)
        mask_high = ~ind_low
        d_high[mask_high] = d_pred[mask_high]
        f_high[mask_high] = f_mid[mask_high]
        
        # Step 2f: Recalculate prediction
        denom = f_high - f_low
        safe_denom = torch.where(torch.abs(denom) > 1e-8, denom, torch.ones_like(denom))
        d_pred = - f_low * ((d_high - d_low) / safe_denom) + d_low
        
    return d_pred
```

**Parameters:**
- $n_{secant\_steps}$: Integer specifying the number of refinement iterations. Typical value: 8 [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Division by Zero**: If $f_{high} == f_{low}$, the denominator in the secant formula becomes zero. The implementation must handle this, typically by falling back to the midpoint or clamping the denominator [G].
- **Non-Bracketing Inputs**: The method assumes $f_{low}$ and $f_{high}$ have opposite signs (or appropriate bracketing). If the initial guess does not bracket the root, the method may diverge or converge to an incorrect root [G].
- **Discontinuous Fields**: If the field function is not continuous, the secant method may fail to converge or converge to an arbitrary point within the interval [S2, unofficial]. In such cases, the False Position Algorithm is preferred [S2, unofficial].

## Subtopics

### [Ray Marching with Secant Refinement](1042-ray-marching-with-secant-refinement.md)
A procedure to detect surface points along rays by evaluating a neural decoder at proposal points, identifying sign changes in occupancy logits, and refining the depth using a root-finding method. It distinguishes itself from its siblings by combining coarse uniform sampling with a specific sign-change detection logic to locate the first entry into the volume, followed by secant refinement. It builds on [Secant Method for Surface Depth Refinement](996-secant-method-for-surface-depth-refinement.md).

### [Parabolic Root-Finding with Gradients](8906-parabolic-root-finding-with-gradients.md)
Parabolic root-finding with gradients is an iterative refinement technique that estimates the zero-crossing of a scalar field $F(x)$ along a line segment by fitting a parabola to the function values and directional derivatives (gradients) at the endpoints. It solves for the root of this parabola to determine the intersection depth. It sets itself apart from its sibling, the [Secant Method for Surface Depth Refinement](996-secant-method-for-surface-depth-refinement.md), by utilizing gradient information to model curvature, thereby avoiding the need for an additional function evaluation to estimate the second derivative, while offering higher accuracy for non-linear fields.

### False Position Algorithm
A root-finding algorithm combining linear interpolation and the midpoint algorithm. It finds the exact intersection point even if the field function is not continuous [S2, unofficial]. Implementation is in `IsoMesher::intersect_xaxis()`, `IsoMesher::intersect_yaxis()`, and `IsoMesher::intersect_zaxis()`. The algorithm used is determined by the preprocessor symbol `FALSE_POSITION` [S2, unofficial]. It is no worse than the midpoint algorithm in terms of iterations and usually better, making it the preferred choice for finding exact intersection points [S2, unofficial].

### Secant Method for Grazing Point
Methods to find the grazing point (where the ray is tangent to the SDF surface) along a ray segment `[t_min, t_max]` using secant or bisection iteration on the directional derivative [S3, unofficial].
- **Secant Method**: Initialize `L = ray(t_min)`, `R = ray(t_max)`. Loop up to `utils.NUM_GRZ` iterations: compute directional derivatives `dL = dir_deriv(L, d)`, `dR = dir_deriv(R, d)`. Compute midpoint `M = R - dR * (R - L) / (dR - dL)`. Compute `dM = dir_deriv(M, d)`. Update `L[dM < 0] = M`, `R[dM > 0] = M`. Return `dr.detach(L)` [S3, unofficial].
- **Bisection Method**: Initialize `L = ray(t_min)`, `R = ray(t_max)`. Loop up to `utils.NUM_GRZ` iterations: compute `M = (L + R) / 2`. Compute `dM = dir_deriv(M, d)`. Update `L[dM < 0] = M[dM < 0]`, `R[dM > 0] = M[dM > 0]`. Return `dr.detach(L)` [S3, unofficial].
- **Selection**: Method is selected based on `utils.GRZ_MODE`: 'secant' or 'bisection' [S3, unofficial].
- **Parameter**: `utils.GRZ_MODE`: String specifying grazing point method, 'secant' or 'bisection' [S3, unofficial].
- **Pitfall**: Gradient computation is suspended during grazing point search using `dr.suspend_grad()` to prevent differentiability issues in the iterative root-finding process [S3, unofficial].

### Newton-Raphson SDF Refinement
Refines the rough intersection position obtained from ray marching by stepping along the ray direction by the SDF value at the current position, effectively performing one Newton-Raphson step or gradient descent step for surface finding [S4, unofficial].
- **Step 1**: Compute the SDF value at the rough intersection position using trilinear interpolation [S4, unofficial].
- **Step 2**: Update the intersection position: $P_{new} = P_{rough} + d \cdot SDF(P_{rough})$, where $d$ is the ray direction [S4, unofficial].
- **Step 3**: For pixels with no intersection (mask=0), preserve the original rough position or handle appropriately. The code adds $(1-mask)$ which implies adding 1 if no intersection, but this seems like a placeholder or specific handling for non-intersecting rays [S4, unofficial].

## Operating characteristics
- **Time Complexity**: $O(n_{secant\_steps})$ function evaluations per ray. Each step involves a 3D point evaluation and arithmetic operations.
- **Memory Complexity**: $O(N)$ for storing the depth and function value vectors for $N$ rays.
- **Performance**: The secant method typically converges faster than bisection for smooth, continuous functions because it uses slope information. However, it can be unstable for discontinuous fields [S2, unofficial].
- **Strengths**: Fast convergence for smooth SDFs; simple implementation; no gradient computation required for the field evaluation (unlike Newton-Raphson) [S1, unofficial][S4, unofficial].
- **Limitations**: Requires two bracketing points with opposite signs; can diverge if the function is not well-behaved; sensitive to discontinuities [S2, unofficial].
- **When to Use**: When the field is smooth and continuous, and computational efficiency is important. When gradients are expensive or unavailable.
- **When Not to Use**: When the field is discontinuous (use False Position) [S2, unofficial] or when high-order convergence is needed and gradients are available (use Newton-Raphson or Parabolic Root-Finding) [S4, unofficial][Parabolic Root-Finding].

## Background
The Secant Method is a classic root-finding algorithm that approximates the derivative using finite differences. In the context of volume rendering and neural implicit surfaces, it is used to refine the depth of surface intersections detected by coarse sampling. The specific implementation described here is found in the `autonomousvision/differentiable_volumetric_rendering` repository [S1, unofficial]. The False Position variant is discussed in the context of isosurface extraction in `IsoMesher` [S2, unofficial]. The grazing point refinement using secant methods is detailed in `zichenwang01/relaxed-boundary` [S3, unofficial]. The Newton-Raphson approach for SDF refinement is presented in `YueJiang-nj/CVPR2020-SDFDiff` [S4, unofficial].

## Sources
[S1] autonomousvision/differentiable_volumetric_rendering, https://github.com/autonomousvision/differentiable_volumetric_rendering, fetched 2026-10-26 (unofficial)
[S2] http://www.oocities.org/tzukkers/isosurf/isosurfaces.html, http://www.oocities.org/tzukkers/isosurf/isosurfaces.html, fetched 2026-10-30 (unofficial)
[S3] zichenwang01/relaxed-boundary, https://github.com/zichenwang01/relaxed-boundary, fetched 2026-10-27 (unofficial)
[S4] YueJiang-nj/CVPR2020-SDFDiff, https://github.com/YueJiang-nj/CVPR2020-SDFDiff, fetched 2026-10-27 (unofficial)
