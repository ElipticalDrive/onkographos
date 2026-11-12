# Multi-pass Blending

Also called: Hardware Polygon Blending with Quadratic Approximation, Multi-pass Quadratic Blending, Quadratic Alpha Blending, Quadratic Fit to Exponential

## Overview
Multi-pass blending is a hardware-accelerated technique for direct volume ray casting that approximates the correct intensity and opacity of semi-transparent cells using standard polygon rasterization [S1, unofficial]. It solves the problem where linear interpolation (Gouraud shading) produces incorrect estimates of intensity and opacity across projected polygons, even when vertex values are accurate [S1, unofficial]. By decomposing the rendering of a single cell into three passes, it uses quadratic functions to approximate the exponential decay of background opacity and the nonlinear contribution of cell color, allowing for correct compositing without software-based ray casting [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A visibility-ordered cell (e.g., from a curvilinear or tetrahedral grid) with vertex attributes: color $C$ and opacity $\alpha$. The cell is projected to screen space.
*   **Output:** The composited color and alpha in the framebuffer, correctly accounting for the exponential transmission of light through the cell's depth.

**Data Structures:**
The method relies on standard hardware polygon attributes. For each vertex $v$ of the projected cell, the following values are computed and passed to the rasterizer:
*   **Color Vector ($\mathbf{c}$):** The interpolated color contribution.
*   **Alpha Scalar ($\alpha$):** The interpolated opacity weight.
*   **Depth ($z$):** The physical depth of the vertex within the cell, ranging from $0$ (nearest) to $A$ (thickest point of the cell projection).

**Formulas:**
The physical model assumes that for a semi-transparent light-emitting medium with planar faces, the depth $z$ varies linearly along any line, but the effective transmission of color varies exponentially [S1, unofficial]. The transmitted light $f(z)$ at depth $z$ is:
$$ f(z) = 1 - e^{-a z} $$
where $a$ is the absorption coefficient and $z$ is the depth [S1, unofficial].

This exponential function is approximated by a quadratic function $g(z)$ over the interval $[0, A]$, where $A$ is the depth of the thickest vertex:
$$ g(z) = (c_0 + c_1 z)(a_0 + a_1 z) $$
The approximation is constrained such that $g(0) = f(0)$ and $g(A) = f(A)$ [S1, unofficial].
1.  $f(0) = 0$, so $g(0) = c_0 a_0 = 0$. The source sets $a_0 = 0$ [S1, unofficial].
2.  To maintain generality and scale, $a_1$ is set to $1/A$. Thus, the alpha term becomes $(z/A)$ [S1, unofficial].
3.  The color term $(c_0 + c_1 z)$ is determined by minimizing the squared error between $g(z)$ and $f(z)$ on $[0, A]$, subject to the constraint $g(A) = f(A)$ [S1, unofficial].

The hardware blend function used is `one minus source alpha` for attenuation passes, and `source color x source alpha` (or equivalent additive blend with alpha weighting) for the final color addition [S1, unofficial].

**Numbered Steps:**
The rendering of a single cell requires three distinct passes over the same polygon geometry:

1.  **Pass 1 (First Opacity Attenuation):**
    *   **Vertex Attributes:**
        *   Color: $\mathbf{c}_1 = \mathbf{C}_{vertex}$ (or a placeholder, as it will be multiplied).
        *   Alpha: $\alpha_1(z) = \frac{z}{A}$ (Linear interpolation of depth normalized by max depth).
    *   **Blend Function:** `one minus source alpha`.
    *   **Effect:** Multiplies the current framebuffer background by $(1 - \frac{z}{A})$. This is the first linear factor of the quadratic approximation.

2.  **Pass 2 (Second Opacity Attenuation):**
    *   **Vertex Attributes:**
        *   Color: $\mathbf{c}_2 = \mathbf{C}_{vertex}$ (Placeholder).
        *   Alpha: $\alpha_2(z) = \frac{z}{A}$ (Same linear interpolation).
    *   **Blend Function:** `one minus source alpha`.
    *   **Effect:** Multiplies the result of Pass 1 by $(1 - \frac{z}{A})$ again. The combined attenuation is $(1 - \frac{z}{A})^2$. *Note: The source describes the product of two linear functions yielding a quadratic approximation to the exponential decay. While $(1-z/A)^2$ is a specific quadratic, the general form derived in the text is $(c_0+c_1z)(a_0+a_1z)$. The source states "The product of the two linear functions is a quadratic". In Pass 1 and 2, the alpha is rigged to give a reasonable quadratic approximation. Specifically, the text implies the attenuation is handled by the product of the alpha terms in the blend operations.*
    *   *Correction/Refinement based on Source:* The source states: "The first two apply opacity of the new cell to the background... The product of the two linear functions yields a quadratic approximation of the exponential decay." It does not explicitly state the alpha is identical in both passes, but implies a sequential application. A common implementation of this "quadratic fit" in hardware is to use the depth $z$ to drive the alpha. If $a_0=0$ and $a_1=1/A$, the alpha term is $z/A$. The color term $(c_0+c_1z)$ is applied in the final pass. The attenuation of the *background* is the tricky part. The source says: "The background needs to be reduced according to the cell’s opacity... use a linear function twice... The product of the two linear functions is a quadratic". This implies the *attenuation factor* is quadratic.
    *   Let's stick strictly to the source's description of the passes:
        *   Pass 1: Apply opacity using linear function + `one minus source alpha`.
        *   Pass 2: Apply second linear function using `one minus source alpha`.
        *   Pass 3: Add cell's own color.

3.  **Pass 3 (Color Addition):**
    *   **Vertex Attributes:**
        *   Color: $\mathbf{c}_3(z) = \mathbf{C}_{vertex} \cdot (c_0 + c_1 z)$. The color is modulated by the linear color term of the quadratic approximation.
        *   Alpha: $\alpha_3(z) = \frac{z}{A}$ (Or the alpha term required to weight the color addition correctly, often matching the opacity contribution).
    *   **Blend Function:** `source color x source alpha` (or `add` with alpha weighting depending on hardware state, but effectively adding the cell's contribution weighted by its opacity).
    *   **Effect:** Adds the cell's color contribution to the attenuated background.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_cell_quadratic(vertex_colors, vertex_depths, A, absorption_coeff):
    """
    Renders a single cell using multi-pass blending.
    vertex_colors: List of RGB vectors for each vertex.
    vertex_depths: List of z-depths for each vertex (0 to A).
    A: Max depth of the cell (thickest vertex).
    absorption_coeff: The 'a' in the exponential model.
    """
    
    # Precompute quadratic coefficients c0, c1 for the color term
    # f(z) = 1 - exp(-a*z)
    # g(z) = (c0 + c1*z) * (z/A)
    # Constraints: g(0)=0, g(A)=f(A)
    # Minimize error on [0, A].
    # From source: a0=0, a1=1/A.
    # c0 and c1 are derived from minimizing squared error.
    # The source says "some of the technical details are outlined in Appendix A"
    # and "minimize ... with a Lagrange multiplier".
    # Since exact closed form for c0, c1 is not fully explicit in the provided text,
    # we define them as computed constants based on A and absorption_coeff.
    c0, c1 = compute_quadratic_color_coeffs(A, absorption_coeff)

    # Pass 1: First attenuation
    # Blend: one minus source alpha
    # Alpha: linear depth z/A
    # Color: arbitrary (will be multiplied by alpha in blend, but here we just attenuate background)
    # Note: Hardware blend 'one minus source alpha' does: dst = dst * (1 - src_alpha)
    # We need src_alpha to be the linear term.
    render_pass(
        vertices=vertex_colors, # Color doesn't matter for attenuation if blend is 1-sa
        alphas=[z / A for z in vertex_depths],
        blend_func='ONE_MINUS_SRC_ALPHA'
    )

    # Pass 2: Second attenuation
    # Blend: one minus source alpha
    # Alpha: linear depth z/A
    # This creates a quadratic attenuation (1 - z/A)^2 approximating the exponential decay
    render_pass(
        vertices=vertex_colors,
        alphas=[z / A for z in vertex_depths],
        blend_func='ONE_MINUS_SRC_ALPHA'
    )

    # Pass 3: Add cell color
    # Blend: Source Color * Source Alpha (or equivalent additive with alpha weight)
    # Color: Modulated by the color term (c0 + c1*z)
    # Alpha: Used to weight the addition
    colors_modulated = []
    for i, z in enumerate(vertex_depths):
        color_term = c0 + c1 * z
        colors_modulated.append(vertex_colors[i] * color_term)
        
    render_pass(
        vertices=colors_modulated,
        alphas=[z / A for z in vertex_depths], # Or potentially the full quadratic alpha term if hardware supports
        blend_func='SRC_ALPHA_SAT' # Or 'SRC_COLOR * SRC_ALPHA' depending on exact hardware state
    )

def compute_quadratic_color_coeffs(A, a):
    """
    Computes c0 and c1 for g(z) = (c0 + c1*z)*(z/A)
    such that g(A) = 1 - exp(-a*A) and error is minimized.
    This is a placeholder for the closed-form solution mentioned in Appendix A of [S1, unofficial].
    """
    # The source does not provide the explicit formula for c0 and c1.
    # It states: "The remaining parameter of the quadratic was chosen to minimize the squared error...
    # solved in closed form".
    # Without Appendix A, we cannot provide the exact numbers.
    # However, g(A) = (c0 + c1*A)*(1) = c0 + c1*A = 1 - exp(-a*A).
    # One degree of freedom remains for error minimization.
    # [G] Typical implementation might set c0=0 or optimize for mean squared error.
    # Since we cannot guess, we return a placeholder or note the missing detail.
    # But the prompt says "Never guess".
    # We will leave this as a comment that the coefficients are precomputed constants
    # derived from the minimization problem described in the source.
    return 0.0, 0.0 # Placeholder, actual values require Appendix A
```

**Edge Cases and Pitfalls:**
*   **Precision:** The method relies on the hardware's ability to accurately interpolate linear attributes and perform the blend operations. Precision errors in the quadratic approximation can lead to visible artifacts, especially with high absorption coefficients where the exponential decay is steep.
*   **Visibility Ordering:** Correctness depends on a valid visibility ordering where no earlier cell occludes a later cell in screen space [S1, unofficial].
*   **Thick Cells:** The approximation is valid on the interval $[0, A]$. If the cell projection is not convex or the depth variation is complex, the linear interpolation of $z$ may not accurately represent the path length, leading to errors.

## Subtopics

### Visibility Ordering
The prerequisite for correct compositing. Ensures that cells are processed in an order such that earlier cells do not occlude later ones in screen space. [Link](5751-visibility-ordering.md)

### Gouraud Shading
The underlying hardware rasterization technique used for linear interpolation of vertex attributes, which multi-pass blending extends via multiple passes. [Link](5752-gouraud-shading.md)

## Operating characteristics
*   **Time Complexity:** Three passes per cell, making it significantly slower than single-pass linear blending. The source notes "The extra time in computation and rendering are substantial" [S1, unofficial].
*   **Memory Complexity:** Low, as it uses standard framebuffer operations and does not require storing intermediate volume data per pixel.
*   **Strengths:** Allows the use of standard hardware polygon renderers for volume rendering, avoiding the need for software ray casting. Provides a much better approximation of opacity and intensity than linear interpolation.
*   **Limitations:** Computationally expensive due to multiple passes. Approximation errors exist, particularly for cells with high absorption or complex geometry. Requires careful visibility ordering.
*   **When to Use:** When hardware acceleration is required and the visual quality of linear blending is insufficient, and software ray casting is too slow.
*   **When Not to Use:** For real-time applications with very high frame rate requirements, or when exact physical accuracy is required.

## Background
Multi-pass blending was introduced in the context of rapid exploration of curvilinear grids using direct volume rendering [S1, unofficial]. It addresses the limitations of earlier methods that used linear interpolation (Gouraud shading) for color and opacity, which produced incorrect intensity and opacity estimates [S1, unofficial]. The method was developed to leverage existing hardware polygon renderers while achieving higher fidelity in volume visualization.

## Sources
[S1] NASA Technical Reports Server (NTRS) 19930015755: Rapid exploration of curvilinear grids using direct volume rendering, https://archive.org/details/NASA_NTRS_Archive_19930015755, fetched 2026-10-26 (unofficial)
