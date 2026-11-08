# Single-Layer Ternary Depth Test

Also called: Single-layer depth test, Ternary Depth Test, Weyrich et al. depth test

## Overview
The Single-Layer Ternary Depth Test is a rasterization technique for point-sampled surfaces that maintains only a single surface layer per pixel, using a depth band to decide whether to blend, replace, or discard incoming fragments [S1, unofficial]. It builds on [Splat Depth Pass](1078-splat-depth-pass.md) and differs from multi-layer approaches like depth-interval grouping by strictly limiting visibility to one depth value per pixel, which introduces visibility and occlusion artifacts in complex geometry [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A stream of splatted fragments (surfels), each with a depth value $d$, color, and weight [S1, unofficial].
*   **State:** A per-pixel buffer storing a single depth value $d_{stored}$ and the accumulated color/weight for the current surface layer [S1, unofficial].
*   **Output:** The final composited color for the pixel after processing all fragments [S1, unofficial].

**Data Structures:**
*   `d_stored`: A single floating-point value representing the depth of the current surface layer for the pixel [S1, unofficial].
*   `accumulated_color`: The accumulated color contribution of the current layer [S1, unofficial].
*   `accumulated_weight`: The accumulated weight (coverage) of the current layer [S1, unofficial].
*   $\epsilon$: The depth band threshold parameter [S1, unofficial].

**Parameters:**
*   $\epsilon$: The tolerance for the depth band. Not given in the sources.

**Algorithm Steps:**
1.  **Initialization:** For each pixel, initialize `d_stored` to a value indicating no surface and `accumulated_weight` to 0 [G].
2.  **Fragment Processing:** For each incoming fragment with depth $d$:
    a.  Compare the fragment's depth $d$ to the stored depth $d_{stored}$ [S1, unofficial].
    b.  Calculate the absolute difference $\Delta d = |d - d_{stored}|$ [S1, unofficial].
    c.  **Ternary Decision:**
        i.  **Blend:** If $\Delta d \le \epsilon$, the fragment is considered part of the same surface layer [S1, unofficial]. Blend the fragment into the running EWA sum [S1, unofficial]. Update `accumulated_color` and `accumulated_weight` using standard EWA splatting accumulation [S1, unofficial].
        ii. **Replace/Discard:** If $\Delta d > \epsilon$, the fragment is outside the depth band [S1, unofficial]. Apply a pass/fail rule to either replace the stored layer or discard the fragment [S1, unofficial]. The specific condition for the pass/fail rule is not given in the sources.
3.  **Normalization:** After all fragments are processed, normalize the accumulated color by the accumulated weight [S1, unofficial]. Because no coverage-based opacity is involved, the normalized output is fully opaque wherever any surfel contributes [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Per-pixel state
d_stored = float('inf')
accumulated_color = (0.0, 0.0, 0.0)
accumulated_weight = 0.0
epsilon = 0.0  # Not given in the sources.

def process_fragment(fragment):
    global d_stored, accumulated_color, accumulated_weight
    
    d = fragment.depth
    color = fragment.color
    weight = fragment.weight
    
    if d_stored == float('inf'):
        d_stored = d
        accumulated_color = color * weight
        accumulated_weight = weight
    else:
        delta_d = abs(d - d_stored)
        
        if delta_d <= epsilon:
            accumulated_color += color * weight
            accumulated_weight += weight
        else:
            # Pass/fail rule for replace vs discard is not given in the sources.
            pass

def get_final_color():
    if accumulated_weight == 0:
        return (0.0, 0.0, 0.0, 0.0)
    r, g, b = accumulated_color
    return (r / accumulated_weight, g / accumulated_weight, b / accumulated_weight, 1.0)
```

**Edge Cases and Pitfalls:**
*   **Hard Opaque Fringes:** Because the output is normalized and fully opaque wherever any surfel contributes, even at silhouette pixels with low accumulated weight, the method produces hard, opaque fringes along object boundaries [S1, unofficial].
*   **Self-Occlusion Artifacts:** At concavities, deeper surfels that should form a second surface layer are incorrectly rejected by the ternary test, resulting in holes or translucent patches where the background is visible [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Not given in the sources.
*   **Limitations:** Maintains only a single surface layer per pixel, causing visibility and occlusion artifacts in complex geometry [S1, unofficial].
*   **When to use:** Not given in the sources.
*   **When not to use:** Not given in the sources.

## Background
The Single-Layer Ternary Depth Test was introduced by Weyrich et al. in 2007 as a prior approach for rendering point-sampled surfaces [S1, unofficial]. It was later used as a baseline for comparison in ablation studies for 3DSS, where it was contrasted with depth-interval grouping [S1, unofficial].

## Sources
[S1] 3DSS: 3D Surface Splatting for Inverse Rendering, https://doi.org/10.48550/arxiv.2605.05876, fetched 2026-10-26 (unofficial)
