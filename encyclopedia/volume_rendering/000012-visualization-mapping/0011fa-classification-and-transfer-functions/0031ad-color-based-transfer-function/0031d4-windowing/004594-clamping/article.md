# Clamping

Also called: Range Clamping, Scalar Clamping

## Overview
Clamping is a boundary handling technique used within transfer functions to define the behavior of scalar values that fall outside the defined data range. While [Windowing](6611-windowing.md) maps out-of-range values to black or white, clamping maps them to the boundary colors of the transfer function itself. This prevents the loss of information for extreme values by preserving their association with the nearest defined color rather than discarding them into a neutral background.

## How it works
**Inputs and Outputs:**
*   **Input:** A scalar value $s$ from the volume data.
*   **Input:** The transfer function range $[x_1, x_2]$.
*   **Input:** A boolean flag `clamping_enabled`.
*   **Input:** The color at the lower bound $C_{min}$ (first node color).
*   **Input:** The color at the upper bound $C_{max}$ (last node color).
*   **Output:** The resulting color $C(s)$.

**Data Structures:**
The transfer function is typically represented as a set of nodes with values regularly spaced between $x_1$ and $x_2$ [S1]. The implementation requires access to the first and last node colors.

**Steps:**
1.  Check the `clamping_enabled` flag.
2.  If `clamping_enabled` is true:
    *   If $s < x_1$, return $C_{min}$ [S1].
    *   If $s > x_2$, return $C_{max}$ [S1].
    *   Otherwise, interpolate normally within $[x_1, x_2]$.
3.  If `clamping_enabled` is false:
    *   If $s < x_1$ or $s > x_2$, return black (or the background color) [S1].
    *   Otherwise, interpolate normally.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def clamp_transfer(s, x1, x2, c_min, c_max, clamping_enabled):
    """
    Maps scalar s to color using clamping logic.
    """
    if clamping_enabled:
        if s < x1:
            return c_min
        elif s > x2:
            return c_max
        else:
            # Standard interpolation between x1 and x2
            # Not given in the sources, assumed to be linear or cubic spline
            # based on the parent Windowing method.
            t = (s - x1) / (x2 - x1)
            return lerp(c_min, c_max, t)
    else:
        if s < x1 or s > x2:
            return (0.0, 0.0, 0.0, 0.0) # Black
        else:
            t = (s - x1) / (x2 - x1)
            return lerp(c_min, c_max, t)
```

**Parameters:**
*   `clamping_enabled`: Boolean. Default behavior in VTK is often off, mapping to black [S1].
*   $x_1, x_2$: The lower and upper bounds of the transfer function range [S1].

**Edge Cases:**
*   If $x_1 = x_2$, the range is degenerate. The sources do not specify behavior for this case [G].
*   Precision errors near $x_1$ or $x_2$ should be handled by strict inequality checks ($<$ and $>$) to ensure boundary colors are hit exactly at the limits.

## Subtopics

### Sigmoid Clamping
A specific mathematical function used to smooth and clip objective function values (such as $f_C, f_Q, f_D, f_T$) to the range $[0, 1]$ [S2, unofficial]. It is distinct from the boundary clamping of transfer functions as it applies a non-linear sigmoidal curve to constrain values rather than hard-clipping to boundary colors. It is chosen when smooth gradients are required for optimization objectives in view planning [S2, unofficial].

**Formula:**
$$ \xi(x) = \max\left(0, \min\left(1, \frac{1}{3} \left[1 + e^{(-\alpha_3 e x) + (\alpha_3 0.5 e)}\right]^{-1}\right)\right) $$

Where:
*   $x$ is the input value.
*   $\alpha_3$ is a parameter with a default value of 3 [S2, unofficial].
*   $e$ is the base of the natural logarithm.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per scalar value.
*   **Memory Complexity:** $O(1)$ additional memory (requires storing two boundary colors).
*   **Strengths:** Preserves visibility of extreme data values; prevents "holes" in visualization where data exceeds the window.
*   **Limitations:** Can obscure the distinction between "out of range" and "at the boundary" if the user is not aware clamping is active.
*   **When to use:** When the volume contains values outside the window of interest that still need to be rendered, albeit with reduced contrast or as a solid cap.
*   **When not to use:** When out-of-range values should be treated as empty space or background.

## Background
Clamping is a standard feature in visualization libraries such as VTK (Visualization Toolkit), specifically implemented in `vtkColorTransferFunction` [S1]. It provides an alternative to the default behavior of mapping out-of-range values to black, allowing for more robust handling of data with heavy tails or outliers.

## Sources
[S1] vtkColorTransferFunctions, http://www.vtk.org/doc/nightly/html/classvtkColorTransferFunction.html, fetched 2026-10-29
[S2] SO-NeRF: Active View Planning for NeRF using Surrogate Objectives, https://doi.org/10.48550/arXiv.2312.03266, fetched 2026-10-27 (unofficial)
