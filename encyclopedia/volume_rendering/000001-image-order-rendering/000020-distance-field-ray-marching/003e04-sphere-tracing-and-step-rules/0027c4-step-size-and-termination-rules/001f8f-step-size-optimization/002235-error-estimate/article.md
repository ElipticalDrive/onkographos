# Error Estimate

Also called: Relative Error Metric, Adaptive Step Error Bound

## Overview
Error Estimate calculates a relative error metric to determine the adaptive step size for ray marching, ensuring the step length is proportional to the local field variation relative to the field magnitude [S1, unofficial]. It distinguishes itself from sibling methods like [Safe Raymarching Step Size](1462-safe-raymarching-step-size.md) by using a normalized ratio of the derivative (or error vector) to the field value, rather than relying solely on the absolute SDF value or fixed Lipschitz constants [S1, 1462-safe-raymarching-step-size.md, unofficial]. This approach prevents excessively small steps in regions of high field magnitude while maintaining precision where the field is small.

## How it works
**Inputs:**
*   $k$: The derivative vector (or error vector) at the current position $\mathbf{p}$. For SDFs, this is often the gradient $\nabla f(\mathbf{p})$. For general fields, it represents the local change or estimated error.
*   $y$: The field value vector at $\mathbf{p}$. For scalar fields, this is a single value $f(\mathbf{p})$. For vector fields, it is the vector $\mathbf{y}$.
*   $\epsilon_{min}$: A small floating-point minimum value (e.g., `FLOAT_MIN`) to prevent division by zero.

**Outputs:**
*   $r_v$: The computed relative error metric (scalar), used to scale the step size.

**Data Structures:**
*   Vectors $k$ and $y$ are typically 3-component floats (`vec3`) for spatial fields.

**Steps:**
1.  Compute the absolute value of the field vector $y$: $|y|$.
2.  Determine the maximum component magnitude of $|y|$: $M_y = \max(|y_x|, |y_y|, |y_z|)$.
3.  Apply a threshold to $M_y$ to avoid division by zero: $M_{safe} = \max(M_y, \text{threshold})$. The threshold is often a small epsilon or `FLOAT_MIN` [S1, unofficial].
4.  Compute the relative error components: $r_{v,i} = \frac{|k_i|}{M_{safe}}$ for each component $i \in \{x, y, z\}$.
5.  Return the maximum component of the relative error vector, clamped by a minimum float value:
    $$ r_v = \max(r_{v,x}, r_{v,y}, r_{v,z}) + \text{FLOAT\_MIN} $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_error_estimate(k, y, threshold=1e-7):
    """
    Computes the relative error metric for adaptive stepping.
    
    Args:
        k: Derivative or error vector (vec3).
        y: Field value vector (vec3).
        threshold: Minimum value to prevent division by zero.
        
    Returns:
        r_v: Scalar relative error metric.
    """
    # Step 1 & 2: Absolute values and max component
    abs_y_x = abs(y.x)
    abs_y_y = abs(y.y)
    abs_y_z = abs(y.z)
    
    max_abs_y = max(abs_y_x, max(abs_y_y, abs_y_z))
    
    # Step 3: Threshold application
    safe_max_y = max(max_abs_y, threshold)
    
    # Step 4: Relative error components
    # Note: k is treated as the numerator (derivative/error)
    r_v_x = abs(k.x) / safe_max_y
    r_v_y = abs(k.y) / safe_max_y
    r_v_z = abs(k.z) / safe_max_y
    
    # Step 5: Return max component + float min
    r_v = max(r_v_x, max(r_v_y, r_v_z))
    
    # Add FLOAT_MIN to ensure non-zero result for step size calculation
    return r_v + 1e-45  # FLOAT_MIN approximation
```

**Parameters:**
*   `threshold`: A small value to stabilize division. Typical: `1e-7` or machine epsilon [S1, unofficial].
*   `FLOAT_MIN`: The smallest positive normalized floating-point number. Typical: `1.17549435e-38` for IEEE 754 single precision [S1, unofficial].

**Edge Cases:**
*   **Zero Field Value:** If $y$ is zero, the denominator becomes the threshold, preventing NaN or infinity. This results in a large relative error, leading to smaller step sizes, which is appropriate for high-precision requirements near zero-crossings or singularities.
*   **Large Derivatives:** If $k$ is large relative to $y$, $r_v$ increases, reducing the step size to maintain accuracy.

## Subtopics

### [Data-Dependent Error Metric for Scalar Fields](6981-data-dependent-error-metric-for-scalar-fields.md)
This variant refines the mesh or sampling only in regions where the scalar field value is close to a chosen isovalue $v_{iso}$, ignoring regions far from the isosurface to reduce computation. It differs from the [Mean-Square Error Metric](7137-mean-square-error-metric-for-scalar-fields.md) by being local to the isovalue proximity, whereas the Mean-Square Error evaluates error globally within an element regardless of isovalue proximity, potentially causing unnecessary refinement.

### Embedded Error Norm Calculation
This method computes a norm for an error vector `err` and solution vector `y` given relative tolerance `rtol` and absolute tolerance `atol` [S2, unofficial]. For a single component, it collapses to $|err| / \text{scale}$. For multi-component vectors, it computes an RMS-like norm: $\sqrt{\frac{1}{N} \sum (\frac{err_i}{rtol \cdot |y_i| + atol})^2}$ [S2, unofficial]. Typical parameters are `rtol=1e-6` and `atol=0` [S2, unofficial].

### [Stepsize Annealing](7542-stepsize-annealing.md)
Stepsize Annealing is a training strategy for differentiable volume rendering and neural radiance fields that varies sampling density over training epochs. It starts with coarse sampling (large steps) for fast convergence and progressively refines (smaller steps) to recover details. It is distinct from static error-based adaptive stepping because the schedule is driven by training progress, not local field gradients at inference.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per step, involving basic arithmetic operations (abs, max, division).
*   **Memory Complexity:** $O(1)$, requiring only temporary storage for vector components.
*   **Strengths:** Adapts well to varying field magnitudes; prevents aliasing in high-gradient regions; robust against zero-field values via thresholding.
*   **Limitations:** Requires accurate derivative or error estimation; may produce overly conservative steps if the threshold is poorly chosen.
*   **When to Use:** When the field magnitude varies significantly across the scene, or when high precision is needed near zero-crossings or low-magnitude regions.

## Background
The Error Estimate method is derived from adaptive stepping techniques in numerical integration and ray marching, where the step size is controlled by local error bounds [S1, unofficial]. It is commonly used in real-time rendering engines and graphics APIs (e.g., N tsl) to balance performance and visual fidelity in distance-field ray marching [S1, unofficial]. The specific implementation using component-wise relative error is found in modern graphics repositories [S1, unofficial].

## Sources
[S1] tink-expo/ntsl-graphics, https://github.com/tink-expo/ntsl-graphics, fetched 2026-10-26 (unofficial)
[S2] rusaitis/webpic, https://github.com/rusaitis/webpic, fetched 2026-10-27 (unofficial)
