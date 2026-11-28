# Partial Pre-Integration


## Overview
Partial Pre-Integration is a volume ray integration technique that improves ray integration approximation by pre-integrating part of the volume rendering integral into a 2D lookup table ($\Psi$ table) that is independent of the transfer function [S1, S2]. Unlike previous pre-integration approaches that rely on specific transfer function attributes (requiring table recomputation upon changes), this method allows interactive transfer function updates without recomputing the table, while avoiding the heavy computational overhead of direct integral calculation [S1, S2]. It provides superior accuracy compared to Average Color/Luminance approximations (eliminating artifacts like blue bleeding through yellow) and is visually indistinguishable from Linear Color/Luminance methods but an order of magnitude faster [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** Scalar field samples $s(\bar{\rho})$ along a ray segment, transfer function $g(s)$ (color $c$) and $\tau(s)$ (extinction coefficient), ray segment thickness $l$ (or optical thickness parameters).
*   **Output:** Final fragment color $C_{frag}$.

**Data Structures:**
*   **2D $\Psi$ Table:** A pre-computed 2D texture or array indexed by $\gamma_b$ and $\gamma_f$.
    *   **Indices:** $\gamma_b, \gamma_f \in [0, 1)$.
    *   **Values:** $\Psi_{\gamma_b, \gamma_f}$, representing the pre-integrated transmittance contribution.
    *   **Size:** Not given in the sources.

**Parameters:**
*   **$\gamma$ (Gamma):** A normalized optical thickness parameter defined as $\gamma \equiv \frac{\tau D}{\tau D + 1}$, where $\tau D$ is the optical thickness. The valid range is $[0, 1)$ [S2].
*   **Optical Thickness ($\tau D$):** Recovered from $\gamma$ via $\tau D = \gamma / (1 - \gamma)$ [S2].

**Steps:**
1.  **Retrieve Colors:** For the current ray segment endpoints (or scalar values $s_f, s_b$), retrieve the RGBA colors $c_f, c_b$ and extinction coefficients $\tau_f, \tau_b$ from the transfer function [S1, S2].
2.  **Compute $\gamma$ Indices:** Calculate the $\gamma$ values for the front and back of the segment:
    $$ \gamma_f = \frac{\tau_f l}{\tau_f l + 1} $$
    $$ \gamma_b = \frac{\tau_b l}{\tau_b l + 1} $$
    *(Note: The sources define $\gamma$ generally for optical thickness $\tau D$. In the context of a segment of length $l$, $\tau D$ becomes $\tau l$.)* [S2].
3.  **Lookup $\Psi$:** Use $\gamma_b$ and $\gamma_f$ as indices to retrieve the value $\Psi_{\gamma_b, \gamma_f}$ from the pre-compiled 2D $\Psi$ table [S1, S2].
4.  **Compute Final Color:** Use the retrieved $\Psi$ value, along with the colors $c_f, c_b$ and potentially other shading attributes, to compute the final fragment color. The exact combination formula is not explicitly detailed in the provided text, but it replaces the direct integration step [S1].

**Formula:**
The core function $\Psi_{\gamma_b, \gamma_f}$ is computed by integrating the transmittance over the interval $[0, 1]$, where the extinction coefficient varies linearly between boundary values $\gamma_b$ and $\gamma_f$:

$$ \Psi_{\gamma_b, \gamma_f} = \int_0^1 e^{-\int_s^1 \left( \frac{\gamma_b}{1-\gamma_b}(1-t) + \frac{\gamma_f}{1-\gamma_f}t \right) dt} ds $$

This integral represents the pre-integration of the transmittance term, assuming a linear variation of the optical thickness parameter $\gamma$ across the segment [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def partial_pre_integration(ray_segment, transfer_function, psi_table):
    """
    Computes the fragment color for a ray segment using Partial Pre-Integration.
    
    Args:
        ray_segment: Object containing scalar values at endpoints s_f, s_b and length l.
        transfer_function: Function mapping scalar s to (color, extinction).
        psi_table: 2D lookup table indexed by (gamma_b, gamma_f).
        
    Returns:
        Final fragment color.
    """
    # 1. Retrieve colors and extinction coefficients from transfer function
    c_f, tau_f = transfer_function(ray_segment.s_f)
    c_b, tau_b = transfer_function(ray_segment.s_b)
    
    # 2. Compute gamma indices
    # gamma = (tau * l) / (tau * l + 1)
    gamma_f = (tau_f * ray_segment.l) / (tau_f * ray_segment.l + 1)
    gamma_b = (tau_b * ray_segment.l) / (tau_b * ray_segment.l + 1)
    
    # 3. Lookup Psi value
    # Note: In implementation, gamma indices must be scaled to table resolution
    psi_val = psi_table.lookup(gamma_f, gamma_b)
    
    # 4. Compute final color
    # The exact combination of c_f, c_b, and psi_val is not specified in the sources.
    # Typically, this involves blending the colors weighted by the pre-integrated transmittance.
    # [G] Placeholder for the specific color blending formula derived from the volume rendering equation.
    final_color = compute_color_from_psi(c_f, c_b, psi_val)
    
    return final_color
```

**Edge Cases and Pitfalls:**
*   **Table Construction:** The construction of the $\Psi$ table is very compute intensive and can take hours to perform [S2]. However, it is computed once and reused [S2].
*   **Precision:** The sources do not specify precision issues, but the mapping from $\tau D$ to $\gamma$ handles the infinite domain of optical thickness by mapping it to $[0, 1)$ [S2].
*   **Boundary Conditions:** The range of $\gamma$ is $[0, 1)$. If $\tau D \to \infty$, $\gamma \to 1$. The table must handle the limit as $\gamma \to 1$ appropriately [S2].

## Subtopics

### [Hybrid Volume and Isosurface Rendering with Partial Pre-Integration](1311-hybrid-volume-and-isosurface-rendering-with-partial-pre-inte.md)
A GPU-accelerated cell projection method that combines direct volume rendering with multiple isosurfaces. It uses tetrahedral discretization and eliminates bus transfer overhead.

### [Semi-Analytical Fragment Shader Preintegration](1379-semi-analytical-fragment-shader-preintegration.md)
Improves accuracy by adaptively subdividing the ray integration interval based on local extrema of scalar and shading polynomials, using front- and back-weighted tables.

## Operating characteristics
*   **Time Complexity:** The table construction is $O(N^2)$ where $N$ is the table resolution, but this is a one-time cost [S2]. Per-fragment cost involves two transfer function lookups, two gamma calculations, one 2D table lookup, and a color blend. It is slower than the simple average scalar method but faster than direct integration [S1, S2].
*   **Memory Complexity:** Requires storage for the 2D $\Psi$ table. Size depends on resolution, not given in sources.
*   **Performance:** Rendering speeds vary with volume, classification, projection, and image size [S2]. It is an order of magnitude faster than Linear Color/Luminance methods [S2].
*   **Strengths:** Interactive transfer function changes without table recomputation; superior accuracy to Average Color/Luminance; applicable to any classification system (gradient shading, opacity, etc.) [S1, S2].
*   **Limitations:** Slower than simple average scalar methods; table construction is computationally expensive (though one-time) [S1, S2].

## Background
Partial Pre-Integration was introduced by Kenneth Moreland in 2004 [S2]. It builds upon the original Pre-Integration (PT) method by Shirley and Tuchman, which required the pre-integration table to be computed based on the transfer function [S1, S2]. Moreland's method decouples the table from the transfer function by using the $\gamma$ parameterization, allowing a single universal table to be used for any classification algorithm [S2]. The method is used in fragment processors for volume ray integration and supports various shading techniques including gradient shading, gradient opacity, boundary highlights, silhouettes, and multidimensional transfer functions [S2].

## Sources
[S1] GPU-Based Cell Projection for Interactive Volume Rendering, https://graphics.tudelft.nl/~marroquim/publications/pdfs/marroquim-sibgrapi2006.pdf, fetched 2026-10-27
[S2] A Fast High Accuracy Volume Renderer for Unstructured ..., https://www.kennethmoreland.com/partial-pre-integration/volvis04.pdf, fetched 2026-10-28
