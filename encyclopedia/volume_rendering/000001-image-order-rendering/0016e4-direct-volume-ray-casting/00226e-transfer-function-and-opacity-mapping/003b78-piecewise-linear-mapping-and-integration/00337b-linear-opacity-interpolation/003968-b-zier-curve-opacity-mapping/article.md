# Bézier Curve Opacity Mapping

Also called: opacity curve editor, pokrytieKriv

## Overview
Bézier Curve Opacity Mapping maps scalar volume data values to opacity using a user-defined Bézier curve, allowing non-linear opacity control across a selected data range [S1, unofficial]. It sets itself apart from its parent [Linear Opacity Interpolation](7016-linear-opacity-interpolation.md) by replacing piecewise linear segments with smooth polynomial curves, and from [Piecewise Function Scalar Opacity Transfer](12435-piecewise-function-scalar-opacity-transfer.md) by restricting the mapping to a single continuous Bézier segment defined by control points within a specific editor window.

## How it works
**Inputs and Outputs**
*   **Input:** A scalar volume data value $i$ (UINT16, range 0–4095).
*   **Input:** A set of control points defining the Bézier curve, stored in an array `kriv.suradnice[]` with size `kriv.pocet`.
*   **Input:** Editor geometry parameters: left coordinate `edt_left`, top coordinate `edt_top`, width `edt_sir`, and height `edt_vys`.
*   **Output:** Opacity $\alpha \in [0, 1]$.

**Data Structures**
*   `kriv`: Structure containing:
    *   `pocet`: Integer count of control points.
    *   `suradnice[]`: Array of points, where each point has `x` and `y` coordinates (integer pixel coordinates relative to the editor).
*   `edt_*`: Integer pixel coordinates defining the bounding box of the curve editor.

**Steps**
1.  **Validation:** Check if the curve has at least 2 points (`kriv.pocet > 1`). If not, return opacity 0.0.
2.  **Normalize Input:** Normalize the input data value $i$ to the range $[0, 1]$ by dividing by the maximum data value (4095.0):
    $$ \text{norm\_i} = \frac{i}{4095.0} $$
3.  **Normalize Curve Boundaries:** Normalize the x-coordinates of the first ($x_0$) and last ($x_{n-1}$) control points to the range $[0, 1]$ relative to the editor width:
    $$ \text{norm\_oa} = \frac{x_0 - \text{edt\_left}}{\text{edt\_sir}} $$
    $$ \text{norm\_ob} = \frac{x_{n-1} - \text{edt\_left}}{\text{edt\_sir}} $$
4.  **Calculate Parameter $t$:** Compute the normalized parameter $t$ along the curve's x-axis domain:
    $$ t = \frac{\text{norm\_i} - \text{norm\_oa}}{\text{norm\_ob} - \text{norm\_oa}} $$
5.  **Boundary Check:** If $\text{norm\_i} < \text{norm\_oa}$ or $\text{norm\_i} > \text{norm\_ob}$, the input value lies outside the defined curve range. Return opacity 0.0.
6.  **Evaluate Bézier Curve:** Evaluate the Bézier curve at parameter $t$ using a function `vratKrivku(t, kriv)`. This returns a point `krivka` with coordinates $(x(t), y(t))$. Note that $y(t)$ is in pixel coordinates relative to the editor top.
7.  **Normalize and Invert Opacity:** Normalize the y-coordinate to the editor height and invert it to derive opacity (assuming y=0 is top and y=height is bottom, or vice versa depending on coordinate system; the source inverts by subtracting from 1.0 after normalization):
    $$ \alpha = 1.0 - \frac{y(t) - \text{edt\_top}}{\text{edt\_vys}} $$

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def bezier_opacity_mapping(i, kriv, edt_left, edt_top, edt_sir, edt_vys):
    """
    Calculate opacity for input value i using Bézier curve mapping.
    
    Args:
        i (int): Input scalar value (0-4095).
        kriv (dict): Curve structure with 'pocet' (int) and 'suradnice' (list of dicts with 'x', 'y').
        edt_left (int): Editor left x-coordinate.
        edt_top (int): Editor top y-coordinate.
        edt_sir (int): Editor width.
        edt_vys (int): Editor height.
        
    Returns:
        float: Opacity alpha in [0.0, 1.0].
    """
    # Step 1: Validation
    if kriv['pocet'] < 2:
        return 0.0

    # Step 2: Normalize input
    norm_i = i / 4095.0

    # Step 3: Normalize curve endpoints
    x0 = kriv['suradnice'][0]['x']
    xn_1 = kriv['suradnice'][kriv['pocet'] - 1]['x']
    
    norm_oa = (x0 - edt_left) / edt_sir
    norm_ob = (xn_1 - edt_left) / edt_sir

    # Step 4: Calculate parameter t
    # Note: Division by zero check implied by norm_ob != norm_oa if points are distinct
    denominator = norm_ob - norm_oa
    if denominator == 0:
        return 0.0 # Fallback for degenerate horizontal line in x
        
    t = (norm_i - norm_oa) / denominator

    # Step 5: Boundary check
    if norm_i < norm_oa or norm_i > norm_ob:
        return 0.0

    # Step 6: Evaluate Bézier curve
    # vratKrivku(t, kriv) computes the y-coordinate of the Bézier curve at parameter t
    # using the control points in kriv['suradnice'].
    krivka = vrat_krivku(t, kriv)
    y_t = krivka['y']

    # Step 7: Normalize and invert
    alpha = 1.0 - (y_t - edt_top) / edt_vys
    
    return alpha

def vrat_krivku(t, kriv):
    """
    Evaluates a cubic Bézier curve at parameter t.
    Assumes the curve is defined by 4 control points. 
    If more points are provided, this implementation assumes a single cubic segment 
    or requires a generalized Bézier evaluation. 
    Based on the source, 'vratKrivku' handles the curve evaluation. 
    For a standard cubic Bézier with control points P0, P1, P2, P3:
    B(t) = (1-t)^3*P0 + 3*(1-t)^2*t*P1 + 3*(1-t)*t^2*P2 + t^3*P3
    """
    # Note: The source does not specify the degree of the Bézier curve.
    # Standard opacity editors often use cubic. 
    # If the number of points is not 4, this function would need to handle 
    # higher-order curves or piecewise segments. 
    # For the purpose of this implementation, we assume a cubic curve 
    # or that the source's 'vratKrivku' handles the specific curve type.
    # Since the source code snippet does not show 'vratKrivku' implementation,
    # we provide a standard cubic evaluation as a placeholder for the method's core.
    
    points = kriv['suradnice']
    n = len(points)
    
    # If the curve is not cubic, this logic needs adjustment.
    # However, typical "Bézier Curve Opacity Mapping" in this context 
    # often implies a single cubic segment or a specific editor behavior.
    # Without explicit degree info, we assume cubic for the formula.
    if n == 4:
        p0 = points[0]['y']
        p1 = points[1]['y']
        p2 = points[2]['y']
        p3 = points[3]['y']
        
        u = 1.0 - t
        u2 = u * u
        u3 = u2 * u
        t2 = t * t
        t3 = t2 * t
        
        y = u3 * p0 + 3 * u2 * t * p1 + 3 * u * t2 * p2 + t3 * p3
        return {'y': y}
    else:
        # Fallback: Linear interpolation if not 4 points, or error.
        # Given the name "Bézier", we expect non-linear.
        # This is a gap filled by [G] assuming standard cubic behavior 
        # for the primary case described in similar editors.
        # If the source implies a different curve type, this would change.
        # For now, we return 0.0 or handle as linear if n=2.
        if n == 2:
            # Linear interpolation
            y = points[0]['y'] + t * (points[1]['y'] - points[0]['y'])
            return {'y': y}
        else:
            # Not given in the sources for n != 2, 4.
            return {'y': 0.0}
```

**Parameters**
*   `i`: Input data value (UINT16, range 0-4095).
*   `edt_sir`/`edt_vys`: Editor width and height in pixels.
*   `edt_left`/`edt_top`: Editor top-left coordinates in pixels.
*   `kriv.suradnice[]`: Array of control points with x, y coordinates.

**Edge Cases and Pitfalls**
*   **Division by Zero:** If `norm_ob == norm_oa` (i.e., the first and last control points have the same x-coordinate), the denominator in the calculation of $t$ becomes zero. The source code implies this is avoided by ensuring distinct x-coordinates for at least 2 points, but explicit handling is not shown [S1, unofficial].
*   **Out-of-Bounds Input:** If the input value $i$ normalized falls outside the range $[\text{norm\_oa}, \text{norm\_ob}]$, the opacity is explicitly set to 0.0 [S1, unofficial].
*   **Coordinate System:** The inversion $1.0 - \dots$ assumes a specific y-axis direction (likely y=0 at top, increasing downwards, common in screen coordinates). If the editor uses a different coordinate system, the inversion logic may need adjustment [G].

## Subtopics

### [Range of Emphasis Opacity](13924-range-of-emphasis-opacity.md)
A technique that emphasizes specific density ranges by scaling opacity differently inside and outside a user-defined interval. It differs from Bézier Curve Opacity Mapping by using interval-based scaling rather than a continuous curve, allowing distinct control over visibility inside vs. outside the range.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per sample, assuming the Bézier curve evaluation is a fixed-degree polynomial (e.g., cubic).
*   **Memory Complexity:** $O(1)$ auxiliary memory per sample, dependent on the number of control points stored in the curve structure.
*   **Strengths:** Provides smooth, non-linear control over opacity, allowing for fine-tuned emphasis of specific data ranges without the sharp transitions of linear interpolation.
*   **Limitations:** Requires user interaction to define control points. Performance is negligible per sample, but the complexity of curve evaluation increases with the degree of the Bézier curve.
*   **When to Use:** When smooth transitions and precise control over opacity gradients are needed, especially for highlighting specific features in volumetric data.
*   **When Not to Use:** When simple, fast, and predictable opacity mapping is sufficient, or when the data range is uniform and does not require non-linear emphasis.

## Background
Bézier Curve Opacity Mapping is a technique used in volume rendering transfer function editors. It allows users to define opacity as a function of scalar data values using Bézier curves. The specific implementation described here is found in the `besnik/parallel-volume-rendering` repository, which is an unofficial source [S1, unofficial]. The method is part of a broader family of opacity mapping techniques that evolved from simple linear mappings to more flexible parametric curves.

## Sources
[S1] besnik/parallel-volume-rendering, https://github.com/besnik/parallel-volume-rendering, fetched 2026-10-27 (unofficial)
