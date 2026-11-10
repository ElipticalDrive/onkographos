# Piecewise Linear Interpolation (Lookup Table)

Also called: 1D Transfer Function Control Point Interpolation, Linear Interpolation between Control Points, Linear Interpolation on 1D LUT, Lookup Table (LUT), Piecewise Linear Transfer Function, Piecewise Linear Transfer Function Evaluation, TFN Linear Interpolation, Tabela de Busca

## Overview

Piecewise linear interpolation maps scalar volume data values to visual parameters (color and opacity) by constructing a continuous curve through a set of user-defined control points [S1, S2, S4, S6, unofficial]. It solves the problem of defining a transfer function (TF) that is intuitive to edit and computationally efficient to evaluate during ray casting [S5, S6, unofficial]. It differs from [Linear Look-up Table Classification](2636-linear-look-up-table-classification.md) by explicitly computing intermediate values via interpolation rather than assigning discrete classes, and from [Smoothstep Interpolation](3287-smoothstep-interpolation.md) by lacking derivative continuity at the knots [S1, S4, unofficial].

## How it works

**Inputs and Outputs**
*   **Input:** A set of $N$ control points $P = \{p_0, p_1, \dots, p_{N-1}\}$, where each point $p_i$ contains a scalar position $pos_i$ and a visual value $val_i$ (scalar or 4-component color vector) [S2, S4, S6, unofficial]. The points must be sorted by $pos_i$ [S6, unofficial].
*   **Input:** A query scalar value $t$ [S2, S5, S6, unofficial].
*   **Output:** An interpolated visual value $v$ [S2, S4, S6, unofficial].

**Data Structures**
*   **Control Point Array:** A contiguous array or list of structures containing position and value fields [S2, S6, unofficial].
*   **Lookup Table (LUT):** An indexed table of discretized visual properties and scalar data values, often pre-computed from the control points for performance [S1, unofficial].

**Numbered Steps**

1.  **Edge Case Handling:** If the control point set is empty, return the default value (e.g., $\{0,0,0,0\}$) [S2, unofficial]. If $t \le p_0.pos$, return $p_0.val$ [S2, S4, unofficial]. If $t \ge p_{N-1}.pos$, return $p_{N-1}.val$ [S2, S4, unofficial]. Clamp $t$ to the valid range if necessary to prevent out-of-bounds access [S5, unofficial].
2.  **Interval Search:** Find the index $i$ (or $nn$) such that $p_i.pos \le t < p_{i+1}.pos$ using binary search [S2, S6, unofficial]. If the search returns an invalid index (e.g., below all points), handle via clamping or error checking [S5, S6, unofficial].
3.  **Weight Calculation:** Let $A = p_i.pos$ and $B = p_{i+1}.pos$ [S6, unofficial]. Compute the interpolation weight $\alpha = \frac{t - A}{B - A}$ [S6, unofficial]. Alternatively, use the symmetric form $\alpha = \frac{(t - p_i.pos)}{(p_{i+1}.pos - p_i.pos)}$ [S2, unofficial].
4.  **Interpolation:** Let $V_1 = p_i.val$ and $V_2 = p_{i+1}.val$ [S6, unofficial]. Compute the result $v = (1 - \alpha) V_1 + \alpha V_2$ [S6, unofficial]. For color values, compute $\alpha$ once and interpolate R, G, B, and A components independently [S6, unofficial]. Alternatively, use the symmetric formula $v = \frac{(t - pl) v_{ir} + (pr - t) v_{il}}{pr - pl}$ where $pl/pr$ are positions and $v_{il}/v_{ir}$ are values [S5, unofficial].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def piecewise_linear_interpolate(control_points, t):
    n = len(control_points)
    if n == 0:
        return (0.0, 0.0, 0.0, 0.0)
    if t <= control_points[0][0]:
        return control_points[0][1]
    if t >= control_points[-1][0]:
        return control_points[-1][1]
    
    left, right = 0, n - 1
    i = 0
    while left <= right:
        mid = (left + right) // 2
        if control_points[mid][0] <= t:
            i = mid
            left = mid + 1
        else:
            right = mid - 1
            
    pos_left = control_points[i][0]
    pos_right = control_points[i + 1][0]
    val_left = control_points[i][1]
    val_right = control_points[i + 1][1]
    
    if pos_right == pos_left:
        return val_left
        
    alpha = (t - pos_left) / (pos_right - pos_left)
    
    if isinstance(val_left, tuple) or isinstance(val_left, list):
        return tuple((1.0 - alpha) * val_left[k] + alpha * val_right[k] for k in range(len(val_left)))
    return (1.0 - alpha) * val_left + alpha * val_right
```

**Parameters**
*   **Control Point Density:** Not given in the sources.
*   **Precision:** Float32 is standard [S1, S4, S5, unofficial].

**Edge Cases and Pitfalls**
*   **Duplicate Positions:** If $p_i.pos == p_{i+1}.pos$, division by zero occurs in weight calculation [S2, unofficial].
*   **Unsorted Input:** Binary search fails if points are not sorted [S6, unofficial].
*   **Out-of-Bounds:** If $t$ is outside $[p_0.pos, p_{N-1}.pos]$, clamping is necessary to prevent index errors or undefined behavior [S4, S5, unofficial].
*   **Binary Search Ambiguity:** If $t$ exactly equals a control point position, standard binary search may return either the left or right interval. Source [S5, unofficial] notes that nearest-neighbor lookup may skip the exact interval if the value equals a control point exactly, requiring careful index handling [S5, S6, unofficial].

## Subtopics

### [Transfer Function Control Point Management](2718-transfer-function-control-point-management.md)
Manages the lifecycle of control points, enforcing sorting and duplicate resolution. Choose this when building a user interface for TF editing to ensure data integrity.

### [Blurring a Step Function](4383-blurring-a-step-function.md)
Convolves step functions with a rectangular pulse to create continuous piecewise linear splines. Choose this for anti-aliased TFs in neural radiance fields (NeRFs) to prevent aliasing during optimization.

### [Step Function Resampling](7924-step-function-resampling.md)
Evaluates integrals of piecewise constant functions. Choose this when you need to preserve the exact mass of a step function rather than interpolating values.

## Operating characteristics

*   **Time Complexity:** $O(\log N)$ per query due to binary search, where $N$ is the number of control points [S5, S6, unofficial].
*   **Memory Complexity:** $O(N)$ for control points [S1, S5, unofficial].
*   **Measured Performance:** Binary search is fast enough for real-time rendering on CPU [S5, unofficial].
*   **Strengths:** Simple to implement, computationally cheap, intuitive for artists [S1, S3, S5, unofficial].
*   **Limitations:** Not smooth (discontinuous derivatives), can appear "blocky" if control points are sparse [S3, S5, unofficial].
*   **When to Use:** Real-time volume rendering, interactive TF editing, when stability is more important than aesthetic smoothness [S3, S5, unofficial].
*   **When Not to Use:** When smooth gradients are required (use [Cubic Spline Transfer Function](1811-cubic-spline-transfer-function.md)) [S1, unofficial].

## Background

Piecewise linear interpolation for transfer functions was used in early volume rendering systems for computational fluid dynamics in the early 1990s [S4, unofficial]. It remains a fundamental technique in modern visualization toolkits and game engines due to its simplicity and efficiency [S1, S4, S5, unofficial]. The method is often implemented as a lookup table (LUT) for performance, where the LUT entries are computed using this interpolation method [S1, unofficial].

## Sources

[S1] A volume exploration approach based on clustering and dimensional reduction to support the definition of multidimensional transfer functions, http://hdl.handle.net/11449/153618, fetched 2026-10-26 (unofficial)
[S2] GitThirteen/vis-magnus-2018, https://github.com/GitThirteen/vis-magnus-2018, fetched 2026-10-26 (unofficial)
[S3] pcarnah/flashdrr, https://github.com/pcarnah/flashdrr, fetched 2026-10-26 (unofficial)
[S4] [PDF] Volume Rendering for Computational Fluid Dynamics: Initial Results, https://www.nas.nasa.gov/assets/nas/pdf/techreports/1991/rnr-91-026.pdf, fetched 2026-10-26 (unofficial)
[S5] wilsonCernWq/TransferFunctionModule, https://github.com/wilsonCernWq/TransferFunctionModule, fetched 2026-10-27 (unofficial)
[S6] devangag558/parallel-3d-volume-rendering-mpi, https://github.com/devangag558/parallel-3d-volume-rendering-mpi, fetched 2026-10-27 (unofficial)
[S7] PDF Parallel Unstructured Volume Rendering in ParaView, https://www.kennethmoreland.com/documents/ParallelVolRen.pdf, fetched 2026-10-27
