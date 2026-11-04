# Piecewise Linear Interpolation (Lookup Table)

Also called: Linear Interpolation on 1D LUT, Piecewise Linear Transfer Function Evaluation, 1D Transfer Function Control Point Interpolation, Linear Interpolation between Control Points, TFN Linear Interpolation

## Overview
Piecewise Linear Interpolation (Lookup Table) evaluates scalar-to-color and scalar-to-opacity transfer functions by linearly interpolating between adjacent control points [S1, S3, S4, unofficial]. It solves the problem of mapping discrete scalar values from a volume dataset to continuous RGBA output values for volume rendering [S4, unofficial]. Unlike [Cubic Spline Transfer Function](1811-cubic-spline-transfer-function.md), which ensures $C^1$ continuity, this method produces $C^0$ continuous functions with sharp changes in derivative at control points, offering lower computational cost and simpler implementation [S4, unofficial]. It differs from [Linear Look-up Table Classification](2636-linear-look-up-table-classification.md) by explicitly supporting arbitrary control point placement and binary search for interval identification rather than fixed uniform grid indexing [S4, unofficial].

## How it works
**Inputs:**
- A list of control points sorted by position (scalar value). Each point contains a position $p$ and a value $v$. For color transfer functions, $v$ is a 4-component vector (RGBA); for scalar curves, $v$ is a scalar [S1, S3, S4, unofficial].
- A query scalar value $P$ [S4, unofficial].

**Outputs:**
- Interpolated value $V$ (scalar or RGBA) at position $P$ [S1, S4, unofficial].

**Data Structures:**
- `ControlPoint`: A struct or tuple containing:
  - `position`: float, the scalar value where this control point is defined.
  - `value`: float (for scalar curves) or `vec4` (for color/opacity curves).
- The list of control points must be sorted by `position` in ascending order [S1, S4, unofficial].

**Steps:**

1. **Boundary Clamping:** If the query value $P$ is less than the position of the first control point, return the value of the first control point. If $P$ is greater than the position of the last control point, return the value of the last control point [S1, S3, S4, unofficial].
   - *Note:* Some implementations clamp $P$ to the range $[0.0, 1.0]$ or the min/max of control points before lookup [S3, unofficial].

2. **Interval Search:** Find the index $i$ such that $points[i].position \le P < points[i+1].position$ [S4, unofficial].
   - This is typically done using binary search (e.g., `std::lower_bound` or a custom `nearestn` function) [S4, unofficial].
   - *Pitfall:* Binary search implementations may return -1 if $P$ is below all control points; ensure boundary checks handle this [S4, unofficial]. If $P$ exactly equals a control point position, the interval starting at that point is selected [S4, unofficial].

3. **Weight Calculation:** Compute the interpolation weight $w$ within the interval $[points[i].position, points[i+1].position]$ [S1, S4, unofficial]:
   $$
   w = \frac{P - points[i].position}{points[i+1].position - points[i].position}
   $$
   - Let $A = points[i].position$ and $B = points[i+1].position$. Then $w = \frac{P - A}{B - A}$ [S4, unofficial].

4. **Value Interpolation:** Compute the interpolated value $V$ using linear interpolation between the values at $points[i]$ and $points[i+1]$ [S1, S4, unofficial]:
   $$
   V = (1 - w) \cdot points[i].value + w \cdot points[i+1].value
   $$
   - For color curves, this is applied independently to each RGB channel and the alpha channel [S4, unofficial].
   - Alternative formula form: $V = \frac{(B - P) \cdot V_i + (P - A) \cdot V_{i+1}}{B - A}$, where $V_i$ and $V_{i+1}$ are the values at indices $i$ and $i+1$ [S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def piecewise_linear_interpolate(control_points, query_value):
    """
    Interpolate a value from a sorted list of control points.
    
    Args:
        control_points: List of tuples (position, value), sorted by position.
        query_value: The scalar value to interpolate at.
        
    Returns:
        The interpolated value (scalar or vec4).
    """
    if not control_points:
        return 0.0  # [G] Default return for empty list; behavior may vary by implementation.

    n = len(control_points)
    
    # Step 1: Boundary Clamping
    if query_value <= control_points[0][0]:
        return control_points[0][1]
    if query_value >= control_points[n - 1][0]:
        return control_points[n - 1][1]
    
    # Step 2: Interval Search (Binary Search)
    # Find largest index i such that control_points[i].position <= query_value
    left = 0
    right = n - 1
    i = 0
    while left <= right:
        mid = (left + right) // 2
        if control_points[mid][0] <= query_value:
            i = mid
            left = mid + 1
        else:
            right = mid - 1
            
    # Step 3: Weight Calculation
    pos_left = control_points[i][0]
    pos_right = control_points[i + 1][0]
    
    # Avoid division by zero if duplicate positions exist (though sorted list should prevent this)
    if pos_right == pos_left:
        return control_points[i][1]
        
    w = (query_value - pos_left) / (pos_right - pos_left)
    
    # Step 4: Value Interpolation
    val_left = control_points[i][1]
    val_right = control_points[i + 1][1]
    
    # Handle both scalar and vector values
    if isinstance(val_left, (int, float)):
        return (1.0 - w) * val_left + w * val_right
    else:
        # Assume val_left is a sequence (e.g., list or tuple of length 4)
        return tuple((1.0 - w) * v_l + w * v_r for v_l, v_r in zip(val_left, val_right))
```

**Parameters:**
- `control_points`: The set of scalar-to-value mappings. Typical number of points: 2–10 for simple transfer functions [S3, unofficial].
- `query_value`: The scalar value from the volume data being sampled [S4, unofficial].

**Edge Cases and Pitfalls:**
- **Duplicate Positions:** If control points have identical positions, the denominator in weight calculation becomes zero. Sources do not specify a standard resolution; implementations should ensure unique positions or handle equality explicitly [S4, unofficial].
- **Out-of-Bounds:** Binary search may fail if input is not sorted or if $P$ is outside the range without clamping [S4, unofficial].
- **Exact Control Points:** If $P$ equals a control point position, the weight $w$ becomes 0 or 1, correctly returning the control point's value [S4, unofficial].

## Subtopics

### [Transfer Function Control Point Management](2718-transfer-function-control-point-management.md)
Manages transfer function control points with automatic sorting, duplicate handling, and LUT initialization based on scalar range [2718-transfer-function-control-point-management.md]. It ensures the control point list is sorted and handles edge cases like duplicate positions before interpolation is performed [2718-transfer-function-control-point-management.md]. Choose this when dynamic user interaction requires frequent addition, removal, or reordering of control points [2718-transfer-function-control-point-management.md].

### Blurring a Step Function
An algorithm to convolve a step function with a rectangular pulse to yield a piecewise linear spline [S5]. It operates on input arrays $x$ and $y$ and uses a half-width parameter $r$ [S5]. It is useful for anti-aliasing transfer functions by smoothing sharp discontinuities [S5]. Choose this when a smooth, continuous derivative is desired without using cubic splines, particularly in neural radiance field contexts [S5].

## Operating characteristics
**Time Complexity:** $O(\log N)$ for binary search interval lookup, where $N$ is the number of control points [S4, unofficial]. Interpolation is $O(1)$ [S4, unofficial].
**Memory Complexity:** $O(N)$ to store control points [S4, unofficial].
**Measured Performance:** Binary search is significantly faster than linear scan for $N > 10$ [S4, unofficial]. The method is extremely lightweight compared to cubic splines, requiring only basic arithmetic operations [S4, unofficial].
**Strengths:** Simple to implement, fast evaluation, easy to understand and edit by artists [S1, S4, unofficial].
**Limitations:** Discontinuous derivatives at control points can cause visual artifacts in gradient-based shading [S4, unofficial]. Not suitable for applications requiring smooth transitions without additional smoothing steps [S4, unofficial].
**When to use:** Real-time volume rendering, simple transfer functions, interactive editing [S1, S4, unofficial].
**When not to use:** When $C^1$ continuity is required, or when the transfer function needs to be differentiable for optimization [S4, unofficial].

## Background
Originated from standard computer graphics techniques for transfer function definition in volume rendering [S1, S4, unofficial]. The method is widely used in visualization toolkits and research prototypes [S1, S3, S4, unofficial]. The "Blurring a Step Function" variant was introduced in Zip-NeRF for anti-aliased neural radiance fields [S5].

## Sources
[S1] GitThirteen/vis-magnus-2018, https://github.com/GitThirteen/vis-magnus-2018, fetched 2026-10-26 (unofficial)
[S2] pcarnah/flashdrr, https://github.com/pcarnah/flashdrr, fetched 2026-10-26 (unofficial)
[S3] wilsonCernWq/TransferFunctionModule, https://github.com/wilsonCernWq/TransferFunctionModule, fetched 2026-10-27 (unofficial)
[S4] devangag558/parallel-3d-volume-rendering-mpi, https://github.com/devangag558/parallel-3d-volume-rendering-mpi, fetched 2026-10-27 (unofficial)
[S5] Zip-NeRF: Anti-Aliased Grid-Based Neural Radiance Fields, https://doi.org/10.1109/iccv51070.2023.01804, fetched 2026-10-26
