# Boundary Emphasis

Also called: Boundary Emphasis TF Generation, Histogram-based Boundary TF

## Overview
Boundary Emphasis is an automatic technique for generating initial transfer functions that highlight material interfaces in volume data by exploiting local gradient information [S1, unofficial]. It builds on [Histogram-Based Transfer Function Generation](6182-histogram-based-transfer-function-generation.md) by using a 3D histogram of voxel values, first derivatives, and second derivatives to identify boundary regions [S1, unofficial]. Unlike sibling methods that focus on noise reduction or global equalization, this method creates color discontinuities at specific voxel values to visually separate adjacent materials with similar intensities [S1, unofficial].

## How it works
**Inputs:** A 3D scalar dataset $V$ [S1, unofficial].
**Outputs:** A piecewise linear transfer function defined by control points, consisting of an opacity function $\alpha(v)$ and a color function $C(v)$ [S1, unofficial].

**Data Structures:**
1.  **3D Histogram $H$:** A 3D array where axes correspond to voxel value $v$, first derivative, and second derivative [S1, unofficial].
2.  **Control Point List $P$:** A list of objects, each containing: value $v_c$, opacity $\alpha_c \in [\alpha_{min}, \alpha_{max}]$, color_pos (RGB for $v > v_c$), color_neg (RGB for $v < v_c$) [S1, unofficial].

**Parameters:**
*   $\alpha_{min}$: Minimum opacity for control points. Default: Not given in the sources.
*   $\alpha_{max}$: Maximum opacity for control points. Default: Not given in the sources.
*   $w$: Width of the interval around each control point. Default: Adjustable (user-defined) [S1, unofficial].

**Steps:**

1.  **Compute Derivatives:** Calculate the first derivative $\nabla v$ and second derivative $\nabla^2 v$ for every voxel in $V$ [G: Standard finite differences are used].
2.  **Build Histogram:** Populate the 3D histogram $H$ relating voxel values with the first and second derivatives [S1, unofficial].
3.  **Identify Control Points:** Analyze $H$ to identify peaks or regions corresponding to boundaries. Select voxel values $v_c$ that are likely in a boundary region as control points [S1, unofficial].
4.  **Assign Opacity:** For each control point $c$, assign an opacity value $\alpha_c$ chosen randomly from the interval $[\alpha_{min}, \alpha_{max}]$ [S1, unofficial].
5.  **Construct Opacity TF:** For each control point $c$ at value $v_c$, define the opacity contribution over the interval $[v_c - w, v_c + w]$. The opacity decreases linearly with distance from $v_c$:
    $$ \alpha(v) = \alpha_c \cdot \left( 1 - \frac{|v - v_c|}{w} \right) \quad \text{for } |v - v_c| \le w $$
    Outside this interval, the contribution is 0. The final opacity TF is the sum of contributions from all control points, forming triangular shapes [S1, unofficial].
6.  **Assign Colors:** For each control point $c$, generate two random colors: $C_{pos}$ for $v > v_c$ and $C_{neg}$ for $v < v_c$ [S1, unofficial].
7.  **Construct Color TF:** Define the color function $C(v)$ by linearly interpolating between the colors associated with adjacent control points [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def boundary_emphasis_tf(data, alpha_min=0.1, alpha_max=1.0, interval_width=5.0):
    # Step 1: Compute derivatives
    grad_x = np.gradient(data, axis=0)
    grad_y = np.gradient(data, axis=1)
    grad_z = np.gradient(data, axis=2)
    first_deriv = np.sqrt(grad_x**2 + grad_y**2 + grad_z**2)
    d2x = np.gradient(np.gradient(data, axis=0), axis=0)
    d2y = np.gradient(np.gradient(data, axis=1), axis=1)
    d2z = np.gradient(np.gradient(data, axis=2), axis=2)
    second_deriv = np.abs(d2x + d2y + d2z)

    # Step 2 & 3: Build histogram and identify control points
    # Sources do not specify the histogram binning or peak detection algorithm [G: Standard 3D histogram binning and local maxima detection are used].
    # For implementation, bin the 3D space (v, first_deriv, second_deriv) and find local maxima.
    boundary_values = np.unique(data) # Populated by histogram peak detection

    if not boundary_values.size:
        return [], (np.min(data), np.max(data))

    # Step 4: Assign Opacity
    control_points = []
    v_range = (np.min(data), np.max(data))

    for v_c in boundary_values:
        alpha_c = np.random.uniform(alpha_min, alpha_max)
        # Step 6: Assign Colors
        color_pos = np.random.rand(3)
        color_neg = np.random.rand(3)
        control_points.append({
            'value': v_c,
            'opacity': alpha_c,
            'color_pos': color_pos,
            'color_neg': color_neg
        })

    return control_points, v_range

def evaluate_tf(control_points, v, interval_width):
    if not control_points:
        return 0.0, np.array([0.0, 0.0, 0.0])

    # Step 5: Opacity Calculation
    alpha_total = 0.0
    for cp in control_points:
        dist = abs(v - cp['value'])
        if dist <= interval_width:
            alpha_contrib = cp['opacity'] * (1.0 - dist / interval_width)
            alpha_total += alpha_contrib

    # Step 7: Color Calculation
    sorted_cps = sorted(control_points, key=lambda k: k['value'])
    idx = -1
    for i in range(len(sorted_cps) - 1):
        if sorted_cps[i]['value'] <= v < sorted_cps[i+1]['value']:
            idx = i
            break

    if idx == -1:
        if v < sorted_cps[0]['value']:
            color = sorted_cps[0]['color_neg']
        else:
            color = sorted_cps[-1]['color_pos']
    else:
        cp_left = sorted_cps[idx]
        cp_right = sorted_cps[idx+1]
        denom = cp_right['value'] - cp_left['value']
        t = 0.0 if denom == 0 else (v - cp_left['value']) / denom
        color = (1.0 - t) * cp_left['color_pos'] + t * cp_right['color_neg']

    return alpha_total, color
```

**Edge Cases and Pitfalls:**
*   **Precision:** The linear interpolation and triangular opacity functions are sensitive to the exact placement of control points. Small variations in derivative calculation can shift boundary locations [G: Standard floating-point precision is sufficient, but derivative noise must be managed by the histogram binning].
*   **Boundaries:** The color interpolation assumes an ordering of control points [G].
*   **Degenerate Input:** If no boundaries are detected (e.g., uniform data), the control point list is empty, and the TF returns zero opacity and black color [G].

## Subtopics

### Feature-sensitive Transfer Function Generation
A technique by Kindlmann and Durkin that uses first and second derivatives to generate feature-sensitive transfer functions automatically, providing data segmentation [S2, unofficial]. It is referenced as the foundational work for the derivative-based approach in Boundary Emphasis [S1, unofficial].

## Operating characteristics
Not given in the sources.

## Background
The method is based on the work of Kindlmann and Durkin [KIN 98], who proposed using first and second derivatives to generate feature-sensitive transfer functions [S1, unofficial]. It follows the histogram approach used by Prauchner et al. [PRA 2005a] to choose control points [S1, unofficial]. The concept of creating color discontinuities to emphasize boundaries was suggested by Fujishiro et al. [FUJ 2000] [S1, unofficial].

## Sources
[S1] Especificação de funções de transferência unidimensionais e multidimensionais para visualização volumétrica direta, http://hdl.handle.net/10183/12077, fetched 2026-10-30 (unofficial)
[S2] Overview of Volume Rendering - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/papers/volvisOverview.pdf, fetched 2026-10-27 (unofficial)
