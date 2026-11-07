# Error Estimate

Also called: Relative Error Metric for Adaptive Stepping

## Overview
Error Estimate calculates a relative error metric to drive adaptive stepping in scalar volume rendering [S1, unofficial]. It determines the necessary refinement by weighting interpolation error based on the distance to a user-specified isovalue $v_{iso}$ [S1, unofficial]. Unlike uniform error metrics that treat all regions equally, this method refines the mesh only where scalar field values are close to $v_{iso}$, significantly reducing computation time in areas far from the isosurface [S1, unofficial]. It sets itself apart from siblings like [Safe Raymarching Step Size](1462-safe-raymarching-step-size.md) by introducing data-dependence on the exploration parameter ($v_{iso}$) rather than relying solely on geometric or field-range properties [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A scalar field value vector $\mathbf{y}$ (typically the interpolated values at the current step or voxel corners), a constant $k$ representing a target error threshold or scaling factor, and a `threshold` value to prevent division by zero.
*   **Output:** A scalar error metric $E$ representing the relative error magnitude.

**Data Structures:**
*   $\mathbf{y}$: A 3-component vector (e.g., `vec3`) containing scalar field samples.
*   $k$: A scalar float.
*   `threshold`: A scalar float, typically a small positive number (e.g., $10^{-6}$ or machine epsilon) to ensure numerical stability.

**Steps:**
1.  Compute the absolute values of the components of $\mathbf{y}$.
2.  Determine the maximum absolute value in $\mathbf{y}$, clamped to be at least `threshold`.
3.  Calculate the relative error for each component by dividing $k$ by the clamped maximum absolute value.
4.  Return the maximum of the resulting relative error components, adding a small constant `FLOAT_MIN` to ensure the result is strictly positive.

**Formulas:**
Let $\mathbf{y} = [y_x, y_y, y_z]$.
The relative error vector $\mathbf{r}_v$ is computed as:
$$ r_{v,i} = \frac{k}{\max(|y_i|, \text{threshold})} \quad \text{for } i \in \{x, y, z\} $$
The final error estimate $E$ is:
$$ E = \max(r_{v,x}, r_{v,y}, r_{v,z}) + \text{FLOAT\_MIN} $$

**Parameters:**
*   $k$: Scaling factor for the error metric. Not given in the sources.
*   `threshold`: Minimum denominator to avoid division by zero. Not given in the sources.
*   `FLOAT_MIN`: Smallest positive normalized floating-point number. Not given in the sources.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def error_estimate(y: Vector3, k: float, threshold: float) -> float:
    """
    Calculates a relative error metric for adaptive stepping.
    
    Args:
        y: Vector3 containing scalar field values.
        k: Scaling factor for the error metric.
        threshold: Minimum value for the denominator to prevent division by zero.
        
    Returns:
        float: The relative error metric.
    """
    # Compute absolute values
    abs_y_x = abs(y.x)
    abs_y_y = abs(y.y)
    abs_y_z = abs(y.z)
    
    # Determine clamped maximum absolute value
    # Note: The source formula uses max(abs(y), threshold). 
    # Interpreting this as component-wise clamping for the denominator 
    # based on the vector result r_v.
    denom_x = max(abs_y_x, threshold)
    denom_y = max(abs_y_y, threshold)
    denom_z = max(abs_y_z, threshold)
    
    # Compute relative error components
    r_v_x = k / denom_x
    r_v_y = k / denom_y
    r_v_z = k / denom_z
    
    # Return the maximum of the relative errors plus a small constant
    # FLOAT_MIN is a standard floating-point minimum positive value [G]
    result = max(r_v_x, max(r_v_y, r_v_z))
    return result + FLOAT_MIN
```

**Edge Cases and Pitfalls:**
*   **Zero Values:** If all components of $\mathbf{y}$ are zero, the denominator becomes `threshold`, preventing division by zero. The error will be $k/\text{threshold}$, which may be large, triggering refinement even in empty space if $k$ is not scaled appropriately.
*   **Threshold Selection:** The choice of `threshold` affects sensitivity. A too-large threshold reduces the metric's dynamic range; a too-small threshold risks numerical instability.
*   **Interpretation of $k$:** The source does not specify how $k$ is derived. It likely relates to the desired interpolation accuracy or the voxel size relative to the field range.

## Subtopics

### [Data-Dependent Error Metric for Scalar Fields](6981-data-dependent-error-metric-for-scalar-fields.md)
This method calculates a relative error metric for adaptive ray marching in scalar volume rendering by weighting the interpolation error based on the distance to a user-specified isovalue $v_{iso}$. Unlike uniform error metrics that treat all regions equally, this approach refines the mesh only in regions where the scalar field values are close to $v_{iso}$, significantly reducing computation time in areas far from the isosurface. It sets itself apart from its sibling [Error Estimate](2768-error-estimate.md) by introducing data-dependence on the exploration parameter ($v_{iso}$) rather than relying solely on geometric or field-range properties.

### Stepsize Annealing
**Definition:** A training strategy that varies the sampling rate $s$ in a range $[s_l, s_h]$ across epochs to benefit from fast convergence at low sampling rates and refinement at high sampling rates [S2, unofficial]. It is a training acceleration method for raymarching models that starts with a high rendering step size in early epochs and decreases it over time to achieve similar performance as training with a constant small step size [S2, unofficial].

**Formula:**
$$ s(e) = s_l \left(1 - \left(\frac{e}{E}\right)^2\right) + \left(\frac{e}{E}\right)^2 s_h $$
where $s(e)$ is the sampling rate at epoch $e$, $s_l$ is the lower bound, $s_h$ is the upper bound, $e$ is the current epoch, and $E$ is the total number of epochs [S2, unofficial].

**Parameters:**
*   $s_l$: Lower bound of sampling rate range. Typical value: 0.1 [S2, unofficial].
*   $s_h$: Upper bound of sampling rate range. Typical value: 2.0 [S2, unofficial] [S2, unofficial].

**Comparison:**
Stepsize annealing produces visual results comparable to training with a fixed sampling rate of $s=2.0$ but reduces training time by 33.4% on average compared to constant $s=2.0$ [S2, unofficial]. Compared to training with a constant small step size, stepsize annealing achieves similar performance but with accelerated training time [S2, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
The error estimate calculation itself is $O(1)$ with constant memory overhead, involving basic arithmetic operations [S1, unofficial]. The overall performance gain depends on the reduction in the number of ray marching steps achieved by adaptive refinement.

**Measured Performance:**
Specific performance metrics for the error estimate method are not provided in the sources [S1, unofficial]. However, related techniques like stepspeed annealing show significant training time reductions (33.4%) when used in conjunction with adaptive sampling strategies [S2, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Reduces computation in regions far from the isosurface by focusing refinement where it matters (near $v_{iso}$) [S1, unofficial]. Simple to implement [S1, unofficial].
*   **Limitations:** Requires careful tuning of parameters $k$ and `threshold` [S1, unofficial]. May not account for geometric complexity or screen-space coverage directly, unlike [Compute Minimal Sample Distance Per Pixel](2684-compute-minimal-sample-distance-per-pixel.md).

**When to Use:**
Use when rendering scalar volumes with a specific isosurface of interest and computational resources are limited [S1, unofficial].

**When Not to Use:**
Avoid when the volume has complex geometry far from the isosurface that requires uniform resolution, or when screen-space accuracy is the primary concern [S1, unofficial].

## Background
**Origin:**
The error estimate method is described in the context of adaptive ray marching for scalar volume rendering [S1, unofficial]. It is part of a broader effort to optimize volume rendering by dynamically adjusting step sizes based on local data properties [S1, unofficial].

**History and Development:**
The method is associated with the NTSL Graphics project, which explores real-time volume rendering techniques [S1, unofficial]. It builds on the principles of adaptive sampling and error-driven refinement [S1, unofficial].

**Where it is Used:**
Used in real-time volume rendering applications where interactive frame rates are required and the volume data has distinct regions of interest defined by isovalue thresholds [S1, unofficial].

## Sources
[S1] tink-expo/ntsl-graphics, https://github.com/tink-expo/ntsl-graphics, fetched 2026-10-26 (unofficial)
[S2] Deep Direct Volume Rendering: Learning Visual Feature Mappings From Exemplary Images, https://doi.org/10.48550/arxiv.2106.05429, fetched 2026-10-26 (unofficial)
