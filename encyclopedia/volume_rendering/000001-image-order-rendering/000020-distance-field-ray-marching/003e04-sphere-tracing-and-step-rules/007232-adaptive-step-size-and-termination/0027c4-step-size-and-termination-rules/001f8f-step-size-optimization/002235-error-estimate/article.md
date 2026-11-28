# Error Estimate


## Overview
Error Estimate calculates a relative error metric for adaptive stepping in ray marching [S1, unofficial]. It determines step sizes by normalizing a derivative or error vector against the local field magnitude, contrasting with methods that rely on absolute field values or fixed Lipschitz constants [G].

## How it works
**Inputs:**
*   $k$: Derivative or error vector at the current position $\mathbf{p}$ [S1, unofficial].
*   $y$: Field value vector at $\mathbf{p}$ [S1, unofficial].
*   `threshold`: Minimum value to prevent division by zero [S1, unofficial].

**Outputs:**
*   $r_v$: Scalar relative error metric used to scale the step size [S1, unofficial].

**Data Structures:**
*   Vectors $k$ and $y$ are 3-component floats (`vec3`) for spatial fields [G].

**Steps:**
1.  Compute the component-wise absolute value of the field vector $y$: $|y|$ [S1, unofficial].
2.  Determine the maximum component magnitude of $|y|$: $M_y = \max(|y_x|, |y_y|, |y_z|)$ [S1, unofficial].
3.  Apply a threshold to $M_y$ to avoid division by zero: $M_{\text{safe}} = \max(M_y, \text{threshold})$ [S1, unofficial].
4.  Compute the relative error components: $r_{v,i} = \frac{|k_i|}{M_{\text{safe}}}$ for each component $i \in \{x, y, z\}$ [S1, unofficial].
5.  Return the maximum component of the relative error vector, clamped by a minimum float value:
    $$ r_v = \max(r_{v,x}, \max(r_{v,y}, r_{v,z})) + \text{FLOAT\_MIN} $$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
FLOAT_MIN = 1.17549435e-38  # IEEE 754 single-precision minimum [G]

def compute_error_estimate(k, y, threshold):
    # Step 1 & 2: Absolute values and max component
    abs_y_x = abs(y.x)
    abs_y_y = abs(y.y)
    abs_y_z = abs(y.z)
    max_abs_y = max(abs_y_x, max(abs_y_y, abs_y_z))
    
    # Step 3: Threshold application
    safe_max_y = max(max_abs_y, threshold)
    
    # Step 4: Relative error components
    r_v_x = abs(k.x) / safe_max_y
    r_v_y = abs(k.y) / safe_max_y
    r_v_z = abs(k.z) / safe_max_y
    
    # Step 5: Return max component + FLOAT_MIN
    return max(r_v_x, max(r_v_y, r_v_z)) + FLOAT_MIN
```

**Parameters:**
*   `threshold`: Not given in the sources.
*   `FLOAT_MIN`: Standard IEEE 754 single-precision minimum is $1.17549435 \times 10^{-38}$ [G].

**Edge Cases:**
*   **Zero Field Value:** If $y = \mathbf{0}$, the denominator falls back to the threshold, preventing division by zero and yielding a large relative error that forces smaller steps [S1, unofficial].
*   **Large Derivatives:** If $k$ is large relative to $y$, $r_v$ increases, reducing the step size to maintain accuracy [G].

## Subtopics

### [Data-Dependent Error Metric for Scalar Fields](6981-data-dependent-error-metric-for-scalar-fields.md)
This variant refines sampling or mesh only in regions where the scalar field value is close to a chosen isovalue $v_{\text{iso}}$, ignoring regions far from the isosurface to reduce computation. It differs from the [Mean-Square Error Metric](7137-mean-square-error-metric-for-scalar-fields.md) by being local to isovalue proximity, whereas the Mean-Square Error evaluates error globally within an element regardless of isovalue proximity, potentially causing unnecessary refinement.

### Embedded Error Norm Calculation
This method computes a norm for an error vector `err` and solution vector `y` given relative tolerance `rtol` and absolute tolerance `atol` [S2, unofficial]. For a single component, it collapses to $|\text{err}| / \text{scale}$ [S2, unofficial]. For multi-component vectors, it computes an RMS-like norm: $\sqrt{\frac{1}{N} \sum \left(\frac{\text{err}_i}{\text{rtol} \cdot |y_i| + \text{atol}}\right)^2}$ [S2, unofficial]. Typical parameters are `rtol = 1e-6` and `atol = 0` [S2, unofficial]. Example: `err=[1e-6, 2e-6]`, `y=[1, 2]`, `rtol=1e-6`, `atol=0` yields a norm of $\approx 1.581139$ [S2, unofficial].

### [Stepsize Annealing](7542-stepsize-annealing.md)
Stepsize Annealing is a training strategy for differentiable volume rendering and neural radiance fields that varies sampling density over training epochs. It starts with coarse sampling (large steps) for fast convergence and progressively refines (smaller steps) to recover details. It is distinct from static error-based adaptive stepping because the schedule is driven by training progress, not local field gradients at inference.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Not given in the sources.
*   **Strengths:** Not given in the sources.
*   **Limitations:** Not given in the sources.
*   **When to Use:** Not given in the sources.

## Background
The method originates from adaptive stepping techniques in numerical integration and ray marching, where step size is controlled by local error bounds [S1, unofficial]. It is implemented in the N tsl graphics framework [S1, unofficial]. Related adaptive methods estimate error by comparing solutions of different numerical orders, such as Euler and Runge-Kutta 2, to adjust step size dynamically [S3].

## Sources
[S1] tink-expo/ntsl-graphics, https://github.com/tink-expo/ntsl-graphics, fetched 2026-10-26 (unofficial)
[S2] rusaitis/webpic, https://github.com/rusaitis/webpic, fetched 2026-10-27 (unofficial)
[S3] Differential Ray Marching, https://doi.org/10.2312/localchapterevents/ceig/ceig12/091-098, fetched 2026-10-26
