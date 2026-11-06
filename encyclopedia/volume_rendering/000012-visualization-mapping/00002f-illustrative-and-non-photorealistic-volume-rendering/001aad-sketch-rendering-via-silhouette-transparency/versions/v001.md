# Sketch Rendering via Silhouette Transparency

Also called: Volume Sketching, Volumetric Sketch Lines

## Overview
This method creates a sketch-like appearance by rendering volume features as lines or edges, making non-silhouette regions transparent and applying shadows based on the original volume's opacity [S1][S2, unofficial]. It differs from standard illustrative rendering by aggressively modulating opacity based on silhouette strength to abstract continuous volumetric data into discrete, sketch-like outlines, rather than preserving the full volumetric density structure [S1][S2, unofficial].

## How it works
**Inputs:**
*   Volume scalar field $V$.
*   View direction $V_{dir}$.
*   Light direction $L_{dir}$.
*   Sketch color $C_{sketch}$ (e.g., white or black).
*   Original volume opacity $\alpha_{orig}$ (computed from the transfer function and scalar values).
*   Parameters: $k_{sc}$ (silhouette center), $k_{ss}$ (silhouette spread), $k_{se}$ (silhouette exponent) [S1][S2, unofficial].

**Outputs:**
*   Final fragment color $C_{final}$ and opacity $\alpha_{final}$.

**Data Structures:**
*   Fragment shader state holding the normalized gradient $\nabla f_n$ (or surface normal proxy) derived from the volume sample [S2, unofficial].
*   Accumulated shadow buffer or pre-computed shadow term $S$ derived from $\alpha_{orig}$ [S1][S2, unofficial].

**Steps:**
1.  **Compute Silhouette Enhancement Factor ($E_{sil}$):**
    Calculate the alignment between the view direction and the volume gradient. The gradient $\nabla f$ is normalized to $\nabla f_n$. The silhouette factor is determined by the angle between the view vector and the gradient. Features facing the viewer (low angle between view and gradient normal) have low silhouette strength.
    $$ E_{sil} = 1 - \max(0, \nabla f_n \cdot V_{dir}) $$
    *Note: The exact formula for $E_{sil}$ is not explicitly provided in the sources, but the parameters $k_{sc}, k_{ss}, k_{se}$ suggest a Gaussian-like or power-law falloff centered on the silhouette edge. A common implementation for such parameters is:*
    $$ E_{sil} = \exp\left( -k_{ss} \cdot ( \max(0, \nabla f_n \cdot V_{dir}) - k_{sc} )^2 \right)^{k_{se}} $$
    *However, since the exact formula is missing, the implementation must rely on the parameter definitions: $k_{sc}$ is the center, $k_{ss}$ is the spread, and $k_{se}$ is the exponent [S1][S2, unofficial].*

2.  **Determine Sketch Opacity:**
    Map $E_{sil}$ to the final opacity $\alpha_{final}$. Regions with high silhouette enhancement (edges) remain opaque, while non-silhouette regions (interiors) become transparent.
    $$ \alpha_{final} = E_{sil} $$
    *Or a thresholded version if binary sketching is desired.*

3.  **Set Sketch Color:**
    The base color of the fragment is set to $C_{sketch}$ [S2, unofficial].

4.  **Calculate Shadows:**
    Compute the shadow term $S$ using the **original** volume opacity $\alpha_{orig}$, not the sketch opacity. This ensures that the sketch lines are shaded by the underlying volumetric structure [S1][S2, unofficial].
    $$ S = \text{ShadowFunction}(\alpha_{orig}, L_{dir}) $$
    *Typically, this involves ray marching or pre-integration of $\alpha_{orig}$ along the light direction.*

5.  **Composite Final Color:**
    $$ C_{final} = C_{sketch} \cdot S $$
    $$ \alpha_{final} = \alpha_{final} \cdot S $$
    *Note: The sources state shadows are calculated based on original opacity to provide shading to the sketch lines [S1][S2, unofficial]. The exact compositing equation (e.g., whether $S$ modulates color only or also opacity) is not fully specified, but $S$ acts as a shading multiplier.*

**Parameters:**
*   $k_{sc} = 0.9$ (Silhouette center) [S1][S2, unofficial].
*   $k_{ss} = 50$ (Silhouette spread) [S1][S2, unofficial].
*   $k_{se} = 0.25$ (Silhouette exponent) [S1][S2, unofficial].

**Edge Cases:**
*   **Degenerate Gradients:** If $\nabla f_n$ is undefined (constant regions), $E_{sil}$ should default to 0 (transparent) to hide interior noise [S2, unofficial].
*   **Precision:** The high value of $k_{ss}$ (50) requires careful handling of floating-point precision in the exponent to avoid underflow or overflow in the silhouette factor calculation [S1][S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Dominated by the shadow calculation, which requires sampling the original volume along the light direction. This is $O(N)$ per fragment where $N$ is the number of shadow samples.
*   **Memory Complexity:** Requires storage for the original volume opacity field for shadow calculation, in addition to the sketch color buffer [S1][S2, unofficial].
*   **Strengths:** Effectively emphasizes feature orientation and creates clear, stylized outlines of complex internal structures (e.g., liver honeycomb, kidney interior) [S1][S2, unofficial].
*   **Limitations:** Loses volumetric density information in non-silhouette regions, making it unsuitable for tasks requiring quantitative analysis of internal density [S1][S2, unofficial].

## Background
This method was introduced in the context of volume illustration for non-photorealistic rendering of volume models [S1][S2, unofficial]. It was developed to address the challenge of visualizing complex internal structures in medical volumes by abstracting them into sketch-like forms that highlight boundaries and orientation [S1][S2, unofficial].

## Sources
[S1] Volume illustration: nonphotorealistic rendering of volume models, https://www.csee.umbc.edu/~rheingan/pubs/volillus.tvcg01.pdf, fetched 2026-10-27
[S2] Non-Photorealistic Rendering of Volume Models, https://userpages.cs.umbc.edu/ebert/npr/volillus.pdf, fetched 2026-10-27 (unofficial)
