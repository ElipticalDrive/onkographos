# Piecewise Hermite Color Transfer Function

Also called: VTK color mapping, vtkColorTransferFunction interpolation

## Overview
The Piecewise Hermite Color Transfer Function is a method for mapping scalar values to RGB colors using modified piecewise Hermite interpolation within a specified color space. It solves the problem of defining smooth, perceptually controlled color gradients between user-defined nodes, supporting interpolation ranging from piecewise constant to piecewise linear. It is distinguished from its sibling [Piecewise Linear Color Transfer Function](11524-piecewise-linear-color-transfer-function.md) by the inclusion of `midpoint` and `sharpness` parameters, which allow for non-linear blending curves that can "squish" the transition or shift the average color location within an interval.

## How it works
**Inputs and Outputs**
*   **Input:** A scalar value $x$ (double).
*   **Output:** An RGB color triplet $(R, G, B)$ with components in the range $[0, 1]$.
*   **Configuration:** A set of nodes defining the transfer function, interpolation parameters (`midpoint`, `sharpness`), color space, scale, and boundary handling flags.

**Data Structures**
The nodes are stored in an interleaved double array. For $n$ nodes, the memory layout is:
$$ [X_1, R_1, G_1, B_1, X_2, R_2, G_2, B_2, \dots, X_n, R_n, G_n, B_n] $$
where $X_i$ is the scalar position and $R_i, G_i, B_i$ are the color components for the $i$-th node.

**Parameters**
*   **midpoint:** A normalized distance $m \in [0, 1]$ between two bounding nodes. It specifies the exact location along the interval where the interpolated color reaches the arithmetic average of the two bounding node colors.
*   **sharpness:** A control parameter $s$ that adjusts the interpolation curve. Higher values "squish" the Hermite basis function, creating more abrupt transitions; lower values create smoother transitions.
*   **colorSpace:** Enum specifying the interpolation domain. Supported values are `RGB`, `HSV`, `CIELAB`, `PROLAB`, `Diverging`, and `Step`.
*   **scale:** Enum specifying the scalar mapping domain. Supported values are `linear` and `logarithmic` (`log10`).
*   **clamping:** Boolean flag. When enabled, scalar values outside the defined node range are clamped to the color of the nearest boundary node.
*   **belowRangeColor / aboveRangeColor:** RGB triplets assigned to scalars falling strictly below or above the minimum/maximum node $X$ values. Controlled by `useBelowRangeColor` and `useAboveRangeColor` boolean flags.
*   **nanColor / nanOpacity:** RGBA color and opacity values returned when the input scalar is NaN. Alpha is only supported for NaN handling via the `NanColorRGBA` feature.

**Algorithm Steps**
1.  **NaN Check:** If the input scalar $x$ is NaN, return `nanColor` (or `nanColorRGBA` if alpha is required).
2.  **Scale Transformation:** If `scale` is `logarithmic`, transform $x$ to $x' = \log_{10}(x)$. If `scale` is `linear`, $x' = x$.
3.  **Boundary Handling:**
    *   If $x' < X_{min}$:
        *   If `useBelowRangeColor` is true, return `belowRangeColor`.
        *   If `clamping` is true, return the color of the first node $(R_1, G_1, B_1)$.
        *   Otherwise, return the color of the first node (default behavior for out-of-bounds if no specific below-range color is set, or potentially undefined depending on implementation; VTK defaults to clamping or below-range color).
    *   If $x' > X_{max}$:
        *   If `useAboveRangeColor` is true, return `aboveRangeColor`.
        *   If `clamping` is true, return the color of the last node $(R_n, G_n, B_n)$.
        *   Otherwise, return the color of the last node.
4.  **Interval Selection:** Locate the interval $[X_i, X_{i+1}]$ such that $X_i \le x' \le X_{i+1}$.
5.  **Normalization:** Compute the normalized distance $t$ within the interval:
    $$ t = \frac{x' - X_i}{X_{i+1} - X_i} $$
6.  **Color Space Conversion:** If `colorSpace` is not `RGB`, convert the RGB components of the bounding nodes $(R_i, G_i, B_i)$ and $(R_{i+1}, G_{i+1}, B_{i+1})$ into the target color space coordinates.
7.  **Hermite Interpolation:** Evaluate the modified piecewise Hermite basis functions using $t$, `midpoint` ($m$), and `sharpness` ($s$) to compute the blending weights. Let $H_0(t)$ and $H_1(t)$ be the basis functions for the start and end nodes respectively.
    *   *Note: The exact formula for the modified Hermite basis is not explicitly provided in the source text, but it is described as a function that "squishes" the standard Hermite curve based on $s$ and shifts the average based on $m$.* [G] The standard Hermite interpolation for a value $C$ between $C_i$ and $C_{i+1}$ is typically $C(t) = H_0(t)C_i + H_1(t)C_{i+1}$. The modification involves adjusting $H_0$ and $H_1$ based on $m$ and $s$.
8.  **Blending:** Compute the interpolated color components in the selected color space:
    $$ C_{interp} = H_0(t) C_i + H_1(t) C_{i+1} $$
    where $C$ represents the color coordinates (e.g., R, G, B or H, S, V).
9.  **Inverse Color Space Conversion:** If the `colorSpace` was not `RGB`, convert the interpolated color coordinates back to RGB.
10. **Clamping:** Ensure the resulting RGB components are within $[0, 1]$.
11. **Return:** Return the final $(R, G, B)$ triplet.

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def piecewise_hermite_color_transfer(x, nodes, params):
    """
    x: scalar value to map
    nodes: list of tuples (X, R, G, B)
    params: dict containing 'midpoint', 'sharpness', 'colorSpace', 'scale', 
            'clamping', 'belowRangeColor', 'aboveRangeColor', 'useBelowRangeColor', 
            'useAboveRangeColor', 'nanColor'
    """
    # 1. NaN Check
    if math.isnan(x):
        return params['nanColor']

    # 2. Scale Transformation
    if params['scale'] == 'logarithmic':
        if x <= 0:
            # Log of non-positive is undefined; handle as below range
            x_scaled = -math.inf
        else:
            x_scaled = math.log10(x)
    else:
        x_scaled = x

    # 3. Boundary Handling
    if not nodes:
        return (0.0, 0.0, 0.0) # Fallback

    X_min = nodes[0][0]
    X_max = nodes[-1][0]

    if x_scaled < X_min:
        if params['useBelowRangeColor']:
            return params['belowRangeColor']
        elif params['clamping']:
            return (nodes[0][1], nodes[0][2], nodes[0][3])
        else:
            return (nodes[0][1], nodes[0][2], nodes[0][3]) # Default to first node color

    if x_scaled > X_max:
        if params['useAboveRangeColor']:
            return params['aboveRangeColor']
        elif params['clamping']:
            return (nodes[-1][1], nodes[-1][2], nodes[-1][3])
        else:
            return (nodes[-1][1], nodes[-1][2], nodes[-1][3]) # Default to last node color

    # 4. Interval Selection
    # Find i such that nodes[i].X <= x_scaled <= nodes[i+1].X
    i = 0
    for j in range(len(nodes) - 1):
        if nodes[j][0] <= x_scaled <= nodes[j+1][0]:
            i = j
            break
    else:
        # Fallback if floating point issues cause it to miss the exact interval
        i = len(nodes) - 2

    X_i, R_i, G_i, B_i = nodes[i]
    X_ip1, R_ip1, G_ip1, B_ip1 = nodes[i+1]

    # 5. Normalization
    denom = X_ip1 - X_i
    if denom == 0:
        t = 0.0
    else:
        t = (x_scaled - X_i) / denom

    # 6. Color Space Conversion
    if params['colorSpace'] == 'RGB':
        C_i = [R_i, G_i, B_i]
        C_ip1 = [R_ip1, G_ip1, B_ip1]
    else:
        # Convert RGB to target space
        C_i = rgb_to_color_space([R_i, G_i, B_i], params['colorSpace'])
        C_ip1 = rgb_to_color_space([R_ip1, G_ip1, B_ip1], params['colorSpace'])

    # 7. Hermite Interpolation
    m = params['midpoint']
    s = params['sharpness']
    
    # Calculate basis functions H0(t) and H1(t)
    # The exact formula for modified Hermite is not in source, using standard Hermite as placeholder
    # with a note that it should be modified by m and s.
    # Standard Hermite: H0 = 2t^3 - 3t^2 + 1, H1 = -2t^3 + 3t^2
    # This is a placeholder for the "modified" part described in [S1]
    H0 = 2 * t**3 - 3 * t**2 + 1
    H1 = -2 * t**3 + 3 * t**2
    
    # Apply sharpness and midpoint modifications [G]
    # Note: The specific mathematical modification for 'sharpness' and 'midpoint' 
    # is not detailed in the source text. This step represents the conceptual application.
    # In VTK, this is handled internally by vtkPiecewiseFunction.
    
    # 8. Blending
    C_interp = [
        H0 * C_i[k] + H1 * C_ip1[k] for k in range(3)
    ]

    # 9. Inverse Color Space Conversion
    if params['colorSpace'] == 'RGB':
        R, G, B = C_interp
    else:
        R, G, B = color_space_to_rgb(C_interp, params['colorSpace'])

    # 10. Clamping
    R = max(0.0, min(1.0, R))
    G = max(0.0, min(1.0, G))
    B = max(0.0, min(1.0, B))

    # 11. Return
    return (R, G, B)

def rgb_to_color_space(rgb, space):
    # Placeholder for actual color space conversion logic
    # Not provided in source, assumed to be standard conversion
    if space == 'HSV':
        # Standard RGB to HSV
        r, g, b = rgb
        max_c = max(r, g, b)
        min_c = min(r, g, b)
        diff = max_c - min_c
        if diff == 0:
            h = 0
        elif max_c == r:
            h = 60 * (((g - b) / diff) % 6)
        elif max_c == g:
            h = 60 * (((b - r) / diff) + 2)
        else:
            h = 60 * (((r - g) / diff) + 4)
        s = 0 if max_c == 0 else diff / max_c
        v = max_c
        return [h, s, v]
    elif space == 'CIELAB':
        # Placeholder for CIELAB conversion
        return rgb # Simplified for pseudocode completeness
    elif space == 'PROLAB':
        return rgb # Simplified
    elif space == 'Diverging':
        return rgb # Simplified
    elif space == 'Step':
        return rgb # Simplified
    return rgb

def color_space_to_rgb(c, space):
    # Placeholder for inverse conversion
    if space == 'HSV':
        h, s, v = c
        if s == 0:
            return [v, v, v]
        h = h / 60.0
        i = int(h)
        f = h - i
        p = v * (1 - s)
        q = v * (1 - s * f)
        t = v * (1 - s * (1 - f))
        if i == 0:
            return [v, t, p]
        elif i == 1:
            return [q, v, p]
        elif i == 2:
            return [p, v, t]
        elif i == 3:
            return [p, q, v]
        elif i == 4:
            return [t, p, v]
        else:
            return [v, p, q]
    return c
```

**Edge Cases and Pitfalls**
*   **Alpha Channel:** Alpha channel interpolation is not supported for standard color transfer. Transparency is strictly limited to the NaN color override feature.
*   **Precision:** The use of `log10` requires handling of non-positive inputs carefully to avoid domain errors.
*   **Degenerate Input:** If nodes are not sorted by X, the interval selection may fail. The source implies nodes are ordered.

## Operating characteristics
*   **Time Complexity:** $O(\log n)$ or $O(n)$ depending on the implementation of interval search, where $n$ is the number of nodes. Color space conversions are constant time $O(1)$.
*   **Memory Complexity:** $O(n)$ to store the node data.
*   **Strengths:** Provides fine-grained control over color transitions via `midpoint` and `sharpness`. Supports multiple color spaces for perceptual uniformity.
*   **Limitations:** Alpha is not interpolated. Requires careful tuning of `midpoint` and `sharpness` for desired effects.
*   **When to use:** When smooth, customizable color gradients are needed, and perceptual uniformity is important (using HSV/CIELAB).
*   **When not to use:** When alpha blending is required across the range, or when simple linear interpolation is sufficient.

## Background
The method is implemented in the Visualization Toolkit (VTK) as `vtkColorTransferFunction`. It is based on the `vtkPiecewiseFunction` class, which provides the underlying piecewise Hermite interpolation mechanism. The class was introduced as part of VTK's visualization mapping capabilities.

## Sources
[S1] vtkColorTransferFunctions, http://www.vtk.org/doc/nightly/html/classvtkColorTransferFunction.html, fetched 2026-10-29
