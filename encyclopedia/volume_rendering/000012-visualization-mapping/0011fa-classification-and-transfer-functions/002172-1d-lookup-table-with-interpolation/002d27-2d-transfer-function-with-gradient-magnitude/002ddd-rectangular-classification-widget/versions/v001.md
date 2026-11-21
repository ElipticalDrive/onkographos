# Rectangular Classification Widget

Also called: General Transfer Function Widget

## Overview
The Rectangular Classification Widget is an interactive tool for defining general transfer functions for scalar data that lacks clear material boundaries [S1, S2]. It operates in the 2D domain of data value and gradient magnitude, allowing users to define regions of interest via a resizable rectangle [S1, S2]. Unlike triangular widgets which emphasize surfaces, this widget supports flexible opacity falloffs (ellipsoidal, linear ramps, tents, or pyramids) while maintaining constant color within the defined region [S1, S2]. It builds on [2D Transfer Function with Gradient Magnitude](5491-2d-transfer-function-with-gradient-magnitude.md).

## How it works
### Inputs and Outputs
*   **Input:** A 2D transfer function domain defined by scalar data value ($v$) and gradient magnitude ($g$).
*   **Output:** A transfer function mapping $(v, g)$ to color ($C$) and opacity ($\alpha$).

### Data Structures
The widget is defined by a rectangular region in the $(v, g)$ plane.
*   **Rectangle:** Defined by minimum and maximum bounds for value ($v_{min}, v_{max}$) and gradient magnitude ($g_{min}, g_{max}$).
*   **Top Bar:** A control element associated with the rectangle, used for translation.
*   **Resize Handles:** Two "balls" (control points) located at the top-right and bottom-left corners of the rectangle, used for resizing.

### Steps
1.  **Define Region:** The user positions a rectangle in the 2D transfer function domain. The rectangle spans specific ranges of data values and gradient magnitudes [S1, S2].
2.  **Translate:** The entire widget is moved freely in the two visible dimensions (value and gradient magnitude) by dragging the top bar [S1, S2].
3.  **Resize:** The dimensions of the rectangular region are adjusted using the balls located at the top-right and bottom-left corners [S1, S2].
4.  **Assign Color:** Color is assigned as a constant value for all points within the rectangular region [S1, S2].
5.  **Assign Opacity:** Opacity is determined based on the position within the rectangle relative to its center or edges, using one of the following falloff modes [S1, S2]:
    *   **Constant:** Opacity is uniform across the rectangle.
    *   **Quadratic (Ellipsoid):** Opacity falls off quadratically, forming an ellipsoid shape where the axes correspond to the rectangle’s aspect ratio.
    *   **Linear (Ramp, Tent, Pyramid):** Opacity falls off linearly from the center or edges.

### Formulas
Let the rectangle be defined by $v \in [v_{min}, v_{max}]$ and $g \in [g_{min}, g_{max}]$.
Let $v_c = \frac{v_{min} + v_{max}}{2}$ and $g_c = \frac{g_{min} + g_{max}}{2}$ be the center of the rectangle.
Let $\Delta v = v_{max} - v_{min}$ and $\Delta g = g_{max} - g_{min}$ be the dimensions.

**1. Constant Opacity:**
$$ \alpha(v, g) = \alpha_{max} \quad \text{if } v_{min} \le v \le v_{max} \text{ and } g_{min} \le g \le g_{max} $$

**2. Quadratic (Ellipsoid) Falloff:**
The opacity falls off quadratically as an ellipsoid with axes corresponding to the rectangle's aspect ratio [S1, S2].
$$ \alpha(v, g) = \alpha_{max} \left( 1 - \left( \frac{v - v_c}{\Delta v / 2} \right)^2 - \left( \frac{g - g_c}{\Delta g / 2} \right)^2 \right) $$
*Note: The opacity is zero outside the ellipse defined by the term in parentheses being $\ge 1$. The source describes this as "quadratic as an ellipsoid with axes corresponding to the rectangle’s aspect ratio" [S1, S2].*

**3. Linear Falloff (Ramp, Tent, Pyramid):**
The opacity falls off linearly [S1, S2]. Specific shapes include ramp, tent, or pyramid.
*   **Pyramid:** Maximal opacity at the center $(v_c, g_c)$, linearly decreasing to zero at the boundaries [S1, S2].
    $$ \alpha(v, g) = \alpha_{max} \left( 1 - \max \left( \frac{|v - v_c|}{\Delta v / 2}, \frac{|g - g_c|}{\Delta g / 2} \right) \right) $$
*   **Ramp/Tent:** The source mentions these as linear options but does not provide explicit formulas for ramp vs. tent distinctions in the 2D rectangular context, only stating they are linear fall-offs [S1, S2].

### Parameters
*   **$\alpha_{max}$:** Maximum opacity (typically 1.0).
*   **Color:** Constant value assigned to the region.

### Edge Cases and Pitfalls
*   **Third Dimension:** The widget operates in 2D (value and gradient magnitude). The third dimension of the transfer function domain (second derivative of scalar data) requires additional controls not provided by the rectangular widget itself [S1, S2].
*   **Boundary Definition:** The rectangular region defines where opacity and color are applied. Points outside the rectangle have no contribution from this widget [S1, S2].

## Operating characteristics
Not given in the sources.

## Background
The Rectangular Classification Widget was introduced in the context of interactive volume rendering using multi-dimensional transfer functions [S1, S2]. It was developed to address the limitation of triangular widgets, which are effective for visualizing surfaces but less suitable for data without clear boundaries [S1, S2]. The research focused on visualizing boundaries between material regions, often using the second derivative to emphasize regions where the second derivative magnitude is small or zero [S1, S2].

## Sources
[S1] PDF Interactive Volume Rendering Using Multi-Dimensional Transfer Functions ..., https://publications.sci.utah.edu/publications/jmk01/simian-paper.pdf, fetched 2026-10-27
[S2] PDF Interactive Volume Rendering Using Multi-Dimensional Transfer Functions ..., https://collections.lib.utah.edu/dl_files/d4/e4/d4e47bfc09e54eee9f532e4f22b17e9214f4c755.pdf, fetched 2026-10-27
