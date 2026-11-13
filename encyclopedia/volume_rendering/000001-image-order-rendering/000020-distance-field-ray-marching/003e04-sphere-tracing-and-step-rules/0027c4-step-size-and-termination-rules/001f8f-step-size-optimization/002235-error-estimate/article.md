# Error Estimate

Also called: Relative Error Metric, Adaptive Step Error Norm

## Overview
Error Estimate calculates a relative error metric to determine the validity of a step in adaptive ray marching or integration [S1, S2, unofficial]. It solves the problem of determining whether the current step size is sufficient by comparing the local error against the magnitude of the solution, scaled by tolerances. It sets itself apart from sibling methods like [Safe Raymarching Step Size](1462-safe-raymarching-step-size.md) by focusing on the *resultant* error magnitude relative to the field value, rather than solely on geometric safety constraints or Lipschitz bounds [S1, S2, unofficial].

## How it works
**Inputs:**
- `k`: A scalar error value or step-related metric [S1, unofficial].
- `y`: The current solution vector (e.g., field value or state vector) [S1, S2, unofficial].
- `threshold`: A minimum floor for the denominator to prevent division by zero [S1, unofficial].
- `rtol`: Relative tolerance parameter [S2, unofficial].
- `atol`: Absolute tolerance parameter [S2, unofficial].

**Outputs:**
- `error_norm`: A scalar value representing the normalized error. If `error_norm > 1`, the step is typically rejected or refined [S2, unofficial].

**Data Structures:**
- `y`: Vector of size $N$ (typically 1 for scalar fields, 3 for vector fields) [S1, S2, unofficial].
- `err`: Error vector of size $N$ [S2, unofficial].

**Steps:**
1.  **Scalar/Component-wise Calculation (S1 variant):**
    For a scalar field or component-wise check, compute the relative error $r_v$:
    $$ r_v = \frac{|k|}{\max(|y|, \text{threshold})} $$
    The final error estimate is the maximum component value plus a small floating-point minimum to ensure positivity:
    $$ \text{error\_norm} = \max(r_v) + \text{FLOAT\_MIN} $$
    [S1, unofficial]

2.  **Vector Norm Calculation (S2 variant):**
    For multi-component fields, compute a norm of the error vector `err` relative to the solution `y`. The scale factor $S$ for each component $i$ is:
    $$ S_i = \text{atol} + \text{rtol} \cdot |y_i| $$
    The norm is computed as an RMS-like value:
    $$ \text{error\_norm} = \sqrt{\frac{1}{N} \sum_{i=1}^{N} \left( \frac{\text{err}_i}{S_i} \right)^2} $$
    [S2, unofficial]

**Parameters:**
- `threshold`: Not given in the sources for S1, but implied as a small positive constant [S1, unofficial].
- `rtol`: Relative tolerance. Typical value: $10^{-6}$ [S2, unofficial].
- `atol`: Absolute tolerance. Typical value: $0$ [S2, unofficial].
- `FLOAT_MIN`: Smallest positive normalized floating-point number (e.g., $1.17549435 \times 10^{-38}$ for IEEE 754 single precision) [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def error_estimate_scalar(k, y, threshold=1e-12):
    """
    Calculates relative error for scalar field stepping [S1, unofficial].
    """
    # Compute denominator with floor to avoid division by zero
    denom = max(abs(y), threshold)
    
    # Relative error
    r_v = abs(k) / denom
    
    # Return max component (scalar) plus float min
    return max(r_v) + 1e-38

def error_estimate_vector(err, y, rtol=1e-6, atol=0.0):
    """
    Calculates RMS-like relative error norm for vector fields [S2, unofficial].
    """
    n = len(y)
    if n == 0:
        return 0.0
    
    sum_sq = 0.0
    for i in range(n):
        # Scale factor for component i
        scale = atol + rtol * abs(y[i])
        
        # Avoid division by zero if scale is zero
        if scale == 0:
            scale = 1e-38 # [G] fallback
            
        sum_sq += (err[i] / scale) ** 2
        
    # RMS norm
    return math.sqrt(sum_sq / n)
```

**Edge Cases:**
- **Zero Solution:** If $|y|$ is near zero, the denominator is clamped by `threshold` (S1) or `atol` (S2) to prevent infinite error estimates [S1, S2, unofficial].
- **Oscillation:** If the error estimate oscillates around 1.0, the step size may need annealing or reduction [Parent].

## Subtopics

### [Data-Dependent Error Metric for Scalar Fields](6981-data-dependent-error-metric-for-scalar-fields.md)
This method refines the mesh only in regions with values close to a chosen isovalue $v_{iso}$, reducing computation time by ignoring regions far from the isovalue. It sets itself apart from its sibling, the Mean-Square Error Metric, which evaluates error globally within an element regardless of isovalue proximity, potentially leading to unnecessary refinement in regions far from the target isosurface.

### [Stepsize Annealing](7542-stepsize-annealing.md)
Step size annealing is a training acceleration strategy for differentiable volume rendering and neural radiance fields (NeRF) that varies the sampling rate (inverse of step size) across training epochs. It solves the problem of high computational cost during early training stages by starting with coarse sampling (large step size) and progressively refining to fine sampling (small step size) as the model converges. It sets itself apart from static sampling strategies by dynamically adjusting the resolution of the volume exploration based on training progress, achieving comparable visual quality to fixed high-resolution training with significantly reduced time.

### Embedded Error Norm Calculation
This variant computes the error norm given an error vector `err`, solution vector `y`, relative tolerance `rtol`, and absolute tolerance `atol` [S2, unofficial]. For a single component, it collapses to $|\text{err}| / \text{scale}$. For multi-component vectors, it computes an RMS-like norm [S2, unofficial]. Example: `err=[1e-6, 2e-6]`, `y=[1, 2]`, `rtol=1e-6`, `atol=0` yields a norm of $\approx 1.581139$ [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of components in the solution vector [S2, unofficial]. For scalar fields, $O(1)$ [S1, unofficial].
- **Memory Complexity:** $O(1)$ auxiliary space [S1, S2, unofficial].
- **Strengths:** Provides a normalized metric that adapts to the magnitude of the field, preventing false positives in high-value regions and false negatives in low-value regions [S1, S2, unofficial].
- **Limitations:** Requires careful tuning of `rtol` and `atol` [S2, unofficial]. The scalar variant (S1) may be less robust for vector fields compared to the norm-based variant (S2) [S1, S2, unofficial].

## Background
The concept of relative error estimation is fundamental in numerical integration and adaptive stepping. The specific implementation in [S1, unofficial] appears in the context of NTSL (NVIDIA Texture Synthesis Language) graphics programming, used for adaptive marching cubes initialization [S1, unofficial]. The vector norm variant in [S2, unofficial] is derived from adaptive trace algorithms used in WebGPU-based ray tracing benchmarks, ensuring step acceptance based on local error bounds [S2, unofficial].

## Sources
[S1] tink-expo/ntsl-graphics, https://github.com/tink-expo/ntsl-graphics, fetched 2026-10-26 (unofficial)
[S2] rusaitis/webpic, https://github.com/rusaitis/webpic, fetched 2026-10-27 (unofficial)
