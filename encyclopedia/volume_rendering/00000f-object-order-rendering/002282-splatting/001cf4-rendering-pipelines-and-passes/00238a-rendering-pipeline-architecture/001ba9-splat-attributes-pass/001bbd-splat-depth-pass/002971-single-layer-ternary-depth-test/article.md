# Single-Layer Ternary Depth Test

Also called: Single-layer depth test, Ternary Depth Test, Weyrich et al. depth test

## Overview
The Single-Layer Ternary Depth Test is a rasterization technique for point-sampled surfaces that maintains only a single surface layer per pixel, using a depth band to decide whether to blend, replace, or discard incoming fragments [S1, unofficial]. It differs from multi-layer approaches like depth-interval grouping by strictly limiting visibility to one depth value per pixel, which simplifies the pipeline but introduces specific visibility and occlusion artifacts in complex geometry [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A stream of splatted fragments (surfels), each with a depth value $d$, color, and weight.
*   **State:** A per-pixel buffer storing a single depth value $d_{stored}$ and the accumulated color/weight for the current surface layer.
*   **Output:** The final composited color for the pixel after processing all fragments.

**Data Structures:**
*   `d_stored`: A single floating-point value representing the depth of the current surface layer for the pixel.
*   `accumulated_color`: The accumulated color contribution of the current layer.
*   `accumulated_weight`: The accumulated weight (coverage) of the current layer.
*   $\epsilon$: The depth band threshold parameter.

**Parameters:**
*   $\epsilon$: The tolerance for the depth band. Not given in the sources.

**Algorithm Steps:**
1.  **Initialization:** For each pixel, initialize `d_stored` to infinity (or a value indicating no surface) and `accumulated_weight` to 0.
2.  **Fragment Processing:** For each incoming fragment with depth $d$:
    a.  Compare the fragment's depth $d$ to the stored depth $d_{stored}$.
    b.  Calculate the absolute difference $\Delta d = |d - d_{stored}|$.
    c.  **Ternary Decision:**
        i.  **Blend:** If $\Delta d \le \epsilon$, the fragment is considered part of the same surface layer. Blend the fragment into the running EWA (Elliptical Weighted Average) sum. Update `accumulated_color` and `accumulated_weight` using standard EWA splatting accumulation.
        ii. **Replace/Discard:** If $\Delta d > \epsilon$, the fragment is outside the depth band. The algorithm applies a pass/fail rule: either replace the current stored layer with this new fragment (resetting accumulation) or discard the fragment entirely. The specific pass/fail rule (e.g., depth comparison against $d_{stored}$) is not explicitly detailed in the provided notes, but the outcome is that the fragment does not blend with the existing layer [S1, unofficial].
3.  **Normalization:** After all fragments are processed, normalize the accumulated color by the accumulated weight. Note that because no coverage-based opacity is involved in the blending phase itself, the normalized output is fully opaque wherever any surfel contributes [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Per-pixel state
d_stored = float('inf')
accumulated_color = (0.0, 0.0, 0.0)
accumulated_weight = 0.0
epsilon = ... # Defined by implementation

def process_fragment(fragment):
    global d_stored, accumulated_color, accumulated_weight
    
    d = fragment.depth
    color = fragment.color
    weight = fragment.weight
    
    # Calculate depth difference
    if d_stored == float('inf'):
        # First fragment for this pixel
        d_stored = d
        accumulated_color = color * weight
        accumulated_weight = weight
    else:
        delta_d = abs(d - d_stored)
        
        if delta_d <= epsilon:
            # Blend into existing layer
            accumulated_color += color * weight
            accumulated_weight += weight
        else:
            # Fragment is outside the epsilon band
            # Apply pass/fail rule: Replace or Discard
            # Note: The specific condition for replace vs discard is not given.
            # Assuming a common strategy: replace if closer to camera, else discard.
            if d < d_stored:
                # Replace the layer
                d_stored = d
                accumulated_color = color * weight
                accumulated_weight = weight
            else:
                # Discard the fragment
                pass

def get_final_color():
    if accumulated_weight == 0:
        return (0.0, 0.0, 0.0, 0.0) # Transparent
    # Normalize color
    r, g, b = accumulated_color
    norm_r = r / accumulated_weight
    norm_g = g / accumulated_weight
    norm_b = b / accumulated_weight
    # Output is fully opaque where weight > 0
    return (norm_r, norm_g, norm_b, 1.0)
```

**Edge Cases and Pitfalls:**
*   **Hard Opaque Fringes:** Because the output is normalized and fully opaque wherever any surfel contributes, even at silhouette pixels with low accumulated weight, the method produces hard, opaque fringes along object boundaries [S1, unofficial]. This creates a "crusty" shell effect beyond the true silhouette.
*   **Self-Occlusion Artifacts:** At concavities, deeper surfels that should form a second surface layer are incorrectly rejected by the ternary test. This results in holes or translucent patches where the background is visible through what should be an opaque object [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of fragments, as each fragment is processed once with a constant-time depth comparison and potential accumulation.
*   **Memory Complexity:** Constant per pixel, storing only one depth value and one accumulated color/weight pair.
*   **Strengths:** Simple implementation, low memory footprint per pixel, single-pass rendering.
*   **Limitations:** Cannot handle multiple surface layers per pixel, leading to visibility errors in complex geometry (e.g., concavities, overlapping surfaces). Produces aliased silhouettes and self-occlusion artifacts [S1, unofficial].
*   **When to use:** Simple, convex objects where self-occlusion is minimal and silhouette quality is not critical.
*   **When not to use:** Complex geometry with concavities, overlapping surfaces, or where accurate silhouette anti-aliasing is required.

## Background
The Single-Layer Ternary Depth Test was introduced by Weyrich et al. in 2007 as a prior approach for rendering point-sampled surfaces [S1, unofficial]. It was later used as a baseline for comparison in ablation studies for 3DSS (3D Surface Splatting for Inverse Rendering), where it was contrasted with depth-interval grouping [S1, unofficial].

## Sources
[S1] 3DSS: 3D Surface Splatting for Inverse Rendering, https://doi.org/10.48550/arxiv.2605.05876, fetched 2026-10-26 (unofficial)
