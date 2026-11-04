# Sub-stepping

## Overview
Sub-stepping is an optimization for adaptive ray marching that guarantees non-decreasing step sizes by performing intermediate sub-steps along the ray as intermediate field sums are computed [S1]. It solves the problem of "bad candidates" in standard Adaptive Lipschitz Constant (ALC) methods, where computing additional field components $f_i$ can unpredictably reduce the step size, leading to wasted computation [S1]. Unlike standard ALC, which calculates the next step based on the current position and all available bounds, Sub-stepping physically advances the ray to validate bounds, ensuring that each additional level of detail contributes positively or neutrally to the step size [S1].

## How it works
**Inputs:**
- Current ray origin $t_0$ [S1].
- Target value $v$ (typically 0 for isosurfaces) [S1].
- Minimum BVH level $i_{min}$ and maximum level $n$ [S1].
- Lipschitz constants $\lambda_{F_i}$ for cumulative sums $F_i$ and $\lambda_{f_i}$ for individual fields $f_i$ [S1].
- BVH bounds $BVH_{i+}(t)$ providing upper bounds for field sums [S1].
- Sum of remaining bounds $s(i, n) = \sum_{k=i}^{n} \Delta_{f_k}$, where $\Delta_{f_k}$ is the bound provided by the BVH for $f_k$ [S1].

**Outputs:**
- Next ray position $t_{next} = t_0 + step_{final}$ [S1].

**Data Structures:**
- The method relies on the parent ALC method's BVH nodes and field functions; no additional complex data structures are required [S1].

**Algorithm Steps:**
1. Compute an initial step size $step_{i_{min}}$ using the BVH at level $i_{min}$ [S1].
2. Physically move the ray position by $step_i$ to $t_0 + step_i$ [S1]. Assume the worst-case scenario where $BVH_{i+}(t_0 + step_i) = v$ [S1].
3. Evaluate the next function $f_{i+1}$ at the new position $t_0 + step_i$ [S1]. Replace the bound $\Delta_{f_{i+1}}$ with the actual value $f_{i+1}(t_0 + step_i)$ [S1].
4. Update the step size using the derived formula [S1].

**Formulas:**
- Initial step size: $step_{i_{min}} = -\frac{F_{i_{min}}(t_0) + s(i_{min} + 1, n)}{\lambda_{F_{i_{min}}}}$ [S1].
- Updated step size: $step_{i+1} = step_i + \frac{\Delta_{f_{i+1}} - f_{i+1}(t_0 + step_i)}{\lambda_{F_i} + \lambda_{f_{i+1}}}$ [S1]. (Note: The source's Eq. 15 writes the denominator as $\lambda_{f_i} + \lambda_{f_{i+1}}$, but the derivation in Eq. 14 and the inequality analysis consistently use $\lambda_{F_i} + \lambda_{f_{i+1}}$; the implementation uses the latter for mathematical consistency with Lipschitz accumulation [S1].)

**Parameters:**
- $v=0$ is standard for isosurfaces [S1].
- $i_{min}$ and $n$ define the BVH level range [S1].

**Edge Cases and Pitfalls:**
- Division by zero occurs if $\lambda_{F_i} + \lambda_{f_{i+1}} = 0$, requiring a numerical guard [G].
- Floating-point precision errors may accumulate if $step_i$ is very small [G].
- Loose BVH bounds produce small initial steps, increasing the number of required sub-steps [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sub_stepping_raymarch(t0, v, i_min, n, bvh_query, fields, lambdas_F, lambdas_f, deltas):
    """
    t0: current ray position
    v: target value (e.g., 0)
    i_min: minimum BVH level
    n: maximum level index
    bvh_query(level, t): returns the upper bound BVH_{level+}(t)
    fields: list of field functions, fields[i](t) returns f_i(t)
    lambdas_F: list of Lipschitz constants for cumulative sums F_i
    lambdas_f: list of Lipschitz constants for individual fields f_i
    deltas: list of BVH bounds for individual fields, deltas[i] = Delta_{f_i}
    """
    # 1. Compute initial step size using BVH at i_min
    bound_i_min = bvh_query(i_min, t0)
    step = (v - bound_i_min) / lambdas_F[i_min]
    
    # 2. Iterate through levels to refine step size via sub-stepping
    for i in range(i_min, n):
        # a. Sub-step: Move ray position forward by current step size
        t_sub = t0 + step
        
        # b. Evaluate next field f_{i+1} at the sub-step position
        f_next_val = fields[i+1](t_sub)
        
        # c. Update step size using actual value instead of BVH bound
        # Denominator uses lambda_{F_i} + lambda_{f_{i+1}} per derivation consistency
        lambda_sum = lambdas_F[i] + lambdas_f[i+1]
        if lambda_sum == 0:
            lambda_sum = 1e-9  # Guard against division by zero [G]
            
        step = step + (deltas[i+1] - f_next_val) / lambda_sum
        
    return step
```

## Subtopics

### [Local Field Interpolation with Dynamic Subdivision](4023-local-field-interpolation-with-dynamic-subdivision.md)
This method is a real-time ray tracing pipeline for visualizing Scale-Invariant Integral Surfaces (SCALIS) defined by skeletons with prescribed radii. It solves the high computational cost of individual SCALIS field evaluations and the challenge of robust closest ray/isosurface intersection by relying on local field interpolations and subdividing the ray into intervals that capture main field variations. It sets itself apart from brute-force marching by using a dynamic view-dependent acceleration structure to select primitives and transforming field values to a space allowing good quadratic polynomial approximation.

## Operating characteristics
- **Time Complexity:** The method performs $O(n - i_{min})$ field evaluations per step, where $n$ is the maximum level and $i_{min}$ is the minimum level [S1].
- **Memory Complexity:** No additional memory is required beyond the BVH and field data structures used by the parent ALC method [S1].
- **Performance:** Sub-stepping performs better on average than the standard Adaptive Lipschitz method because it avoids computing wasteful bad candidates and guarantees step sizes do not decrease [S1].
- **Strengths:** Guarantees non-decreasing step sizes, reducing the likelihood of getting stuck in small steps due to optimistic bounds [S1].
- **Limitations:** Requires evaluating fields at intermediate positions, which increases computational cost per step compared to a single evaluation if the step size is large [S1].

## Background
Sub-stepping was introduced in the paper "Fast sphere tracing of procedural volumetric noise for very large and detailed scenes" [S1]. It addresses the limitations of the standard Adaptive Lipschitz Constant method where computing additional field components can unpredictably reduce the step size [S1]. The method was developed to improve the efficiency of sphere tracing for procedural volumetric noise in large scenes [S1].

## Sources
[S1] Fast sphere tracing of procedural volumetric noise for very large and detailed scenes, https://doi.org/10.1111/cgf.70072, fetched 2026-10-26
