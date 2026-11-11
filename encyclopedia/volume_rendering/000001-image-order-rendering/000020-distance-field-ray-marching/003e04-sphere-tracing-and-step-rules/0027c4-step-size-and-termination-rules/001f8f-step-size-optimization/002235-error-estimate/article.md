# Error Estimate
Also called: Relative Error Metric for Adaptive Stepping

## Overview
Error Estimate calculates a relative error metric to guide adaptive stepping in distance-field ray marching [S1, unofficial]. It solves the problem of determining an appropriate step size by evaluating the local variation of the scalar field relative to its magnitude, ensuring that steps are small enough to capture detail where the field changes rapidly or is small, but large enough to maintain performance where the field is stable [S1, unofficial]. It sets itself apart from siblings like [Safe Raymarching Step Size](1462-safe-raymarching-step-size.md) by using a relative error bound based on the field value itself rather than a fixed geometric bound or derivative-based Lipschitz constant.

## How it works
**Inputs:**
- $k$: The maximum allowable relative error bound (scalar).
- $y$: The scalar field value at the current ray position (vector, typically 3D components $y_x, y_y, y_z$ representing values at neighboring points or a gradient approximation, though the source implies a vector input for the metric calculation).
- `threshold`: A small positive constant to prevent division by zero.
- `FLOAT_MIN`: The smallest positive normalized floating-point number.

**Outputs:**
- A scalar error metric value representing the maximum relative error across the vector components.

**Data Structures:**
- No complex data structures are required. The method operates on scalar and vector primitives.

**Steps:**
1. Compute the absolute value of each component of $y$.
2. Determine the maximum absolute value among the components of $y$.
3. Add the `threshold` to this maximum absolute value.
4. Divide $k$ by this sum to get the relative error for each component (conceptually, though the source formula applies the division to the vector result).
5. Compute the absolute value of the resulting vector $r_v$.
6. Return the maximum component of $r_v$ plus `FLOAT_MIN`.

**Formula:**
Let $y = (y_x, y_y, y_z)$.
$$ r_v = \frac{k}{\max(|y_x|, |y_y|, |y_z|) + \text{threshold}} $$
$$ \text{Error} = \max(|r_{v,x}|, |r_{v,y}|, |r_{v,z}|) + \text{FLOAT\_MIN} $$

Note: The source code snippet `abs(k / max(abs(y), threshold))` suggests that `max(abs(y), threshold)` computes the element-wise maximum of the absolute values of $y$ and the threshold, or more likely, the scalar maximum of the absolute values of $y$ components, clamped by the threshold. Given the context of "relative error", it is most likely:
$$ \text{denominator} = \max(|y_x|, |y_y|, |y_z|) + \text{threshold} $$
$$ r_v = \frac{k}{\text{denominator}} $$
However, the source says `abs(k / max(abs(y), threshold))`. If `max(abs(y), threshold)` is a vector operation (element-wise max), then:
$$ r_v = \left| \frac{k}{\max(|y_x|, \text{threshold}), \max(|y_y|, \text{threshold}), \max(|y_z|, \text{threshold})} \right| $$
Then return $\max(r_v)$.
Let's stick to the literal interpretation of the code snippet provided in [S1, unofficial]:
1. Calculate `max_abs_y` = $\max(|y_x|, |y_y|, |y_z|)$.
2. Calculate `denom` = $\max(\text{max\_abs\_y}, \text{threshold})$.
3. Calculate `r_v` = $k / \text{denom}$.
4. Return $\max(|r_v|) + \text{FLOAT\_MIN}$.
Wait, the code is `abs(k / max(abs(y), threshold))`. In GLM/C++, `max` of a vector and a scalar usually broadcasts. So `max(abs(y), threshold)` creates a vector where each component is $\max(|y_i|, \text{threshold})$.
Then $k$ is divided by this vector.
Then `abs` is taken.
Then `max` of the resulting vector components is taken.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def error_estimate(k, y, threshold, FLOAT_MIN):
    # y is a vector with components y.x, y.y, y.z
    # Calculate absolute values of y components
    abs_y_x = abs(y.x)
    abs_y_y = abs(y.y)
    abs_y_z = abs(y.z)
    
    # Element-wise maximum of abs(y) and threshold
    # This ensures the denominator is at least 'threshold'
    denom_x = max(abs_y_x, threshold)
    denom_y = max(abs_y_y, threshold)
    denom_z = max(abs_y_z, threshold)
    
    # Calculate relative error vector r_v
    # k is a scalar, division broadcasts to vector
    r_v_x = abs(k / denom_x)
    r_v_y = abs(k / denom_y)
    r_v_z = abs(k / denom_z)
    
    # Return the maximum component of r_v plus FLOAT_MIN
    max_r_v = max(r_v_x, max(r_v_y, r_v_z))
    return max_r_v + FLOAT_MIN
```

**Parameters:**
- $k$: Maximum relative error. Typical value: Not given in the sources.
- `threshold`: Small constant to avoid division by zero. Typical value: Not given in the sources.
- `FLOAT_MIN`: Smallest positive float. Typical value: $1.17549435 \times 10^{-38}$ for IEEE 754 single precision.

**Edge Cases and Pitfalls:**
- If $y$ is zero and threshold is zero, division by zero occurs. The threshold prevents this.
- If $k$ is very large, the error estimate will be large, potentially leading to very large step sizes and aliasing.
- Precision issues may arise if $y$ components vary significantly in magnitude.

## Subtopics

### [Data-Dependent Error Metric for Scalar Fields](6981-data-dependent-error-metric-for-scalar-fields.md)
This method refines the mesh only in regions with values close to a chosen isovalue $v_{iso}$, reducing computation time by ignoring regions far from the isovalue. It sets itself apart from its sibling, the Mean-Square Error Metric, which evaluates error globally within an element regardless of isovalue proximity, potentially leading to unnecessary refinement in regions far from the target isosurface.

### Stepsize Annealing
**Definition:** A training strategy that varies the sampling rate $s$ in a range $[s_l, s_h]$ across epochs to benefit from fast convergence at low sampling rates and refinement at high sampling rates [S2, unofficial]. It is a training acceleration method for raymarching models that starts with a high rendering step size in early epochs and decreases it over time to achieve similar performance as training with a constant small step size [S2, unofficial].

**Formula:**
$$ s(e) = s_l \left(1 - \left(\frac{e}{E}\right)^2\right) + \left(\frac{e}{E}\right)^2 s_h $$
where $s(e)$ is the sampling rate at epoch $e$, $s_l$ is the lower bound, $s_h$ is the upper bound, $e$ is the current epoch, and $E$ is the total number of epochs [S2, unofficial].

**Parameters:**
- $s_l$: Lower bound of sampling rate range. Typical value: 0.1 [S2, unofficial].
- $s_h$: Upper bound of sampling rate range. Typical value: 2.0 [S2, unofficial].

**Comparison:**
Stepsize annealing produces visual results comparable to training with a fixed sampling rate of $s=2.0$ but reduces training time by 33.4% on average compared to constant $s=2.0$ [S2, unofficial]. Compared to training with a constant small step size, stepsize annealing achieves similar performance but with accelerated training time [S2, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
The error estimate calculation itself is $O(1)$ in time and memory, involving basic arithmetic operations on a fixed-size vector [S1, unofficial].

**Measured Performance:**
Not given in the sources.

**Scaling:**
Not given in the sources.

**Strengths and Limitations:**
- Strengths: Simple to implement, adapts to local field magnitude, prevents division by zero [S1, unofficial].
- Limitations: Relies on the choice of $k$ and threshold, which may not be optimal for all scenes [S1, unofficial].

**When to Use:**
Use when adaptive stepping is required based on relative field variation [S1, unofficial].

**When Not to Use:**
Not given in the sources.

## Background
**Origin:**
The method originates from the NTSL Graphics project, specifically in the context of marching cubes and adaptive stepping [S1, unofficial].

**History and Development:**
The method was implemented in the `ntsl-graphics` repository [S1, unofficial].

**Where it is used:**
Used in distance-field ray marching for volume rendering [S1, unofficial].

## Sources
[S1] tink-expo/ntsl-graphics, https://github.com/tink-expo/ntsl-graphics, fetched 2026-10-26 (unofficial)
[S2] Deep Direct Volume Rendering: Learning Visual Feature Mappings From Exemplary Images, https://doi.org/10.48550/arxiv.2106.05429, fetched 2026-10-26 (unofficial)
