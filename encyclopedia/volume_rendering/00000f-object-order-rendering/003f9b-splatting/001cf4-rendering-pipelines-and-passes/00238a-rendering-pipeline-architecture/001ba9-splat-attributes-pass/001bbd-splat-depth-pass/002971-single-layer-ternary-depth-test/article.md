# Single-Layer Ternary Depth Test

Also called: Single-layer depth test, Ternary Depth Test, Weyrich et al. depth test

## Overview
The Single-Layer Ternary Depth Test is a rasterization technique for point-sampled surfaces that maintains only a single surface layer per pixel, using a depth band to decide whether to blend, replace, or discard incoming fragments [S1, unofficial]. It builds on [Splat Depth Pass](1078-splat-depth-pass.md) and differs from multi-layer approaches like depth-interval grouping by strictly limiting visibility to one depth value per pixel, which introduces visibility and occlusion artifacts in complex geometry [S1, unofficial].

## How it works
**Inputs and Outputs:**
The method takes a stream of splatted fragments, each with a depth value $d$, color, and weight, as input [S1, unofficial]. It maintains a per-pixel buffer storing a single depth value $d_{stored}$ and the accumulated color and weight for the current surface layer [S1, unofficial]. The output is the final composited color for the pixel after processing all fragments [S1, unofficial].

**Data Structures:**
`d_stored` stores a single floating-point depth value for the current surface layer [S1, unofficial]. `accumulated_color` stores the accumulated color contribution of the current layer [S1, unofficial]. `accumulated_weight` stores the accumulated weight of the current layer [S1, unofficial]. $\epsilon$ defines the depth band threshold [S1, unofficial].

**Parameters:**
$\epsilon$ defines the tolerance for the depth band [S1, unofficial]. Not given in the sources.

**Algorithm Steps:**
1. Initialization: For each pixel, initialize `d_stored` to a value indicating no surface and `accumulated_weight` to 0 [G].
2. Fragment Processing: For each incoming fragment with depth $d$, compare $d$ to the stored depth $d_{stored}$ [S1, unofficial]. Calculate the absolute difference $\Delta d = |d - d_{stored}|$ [S1, unofficial].
3. Ternary Decision: If $\Delta d \le \epsilon$, the fragment falls within the depth band and is blended into the running EWA sum [S1, unofficial]. Update `accumulated_color` and `accumulated_weight` using standard EWA splatting accumulation [S1, unofficial]. If $\Delta d > \epsilon$, the fragment falls outside the depth band [S1, unofficial]. Apply a pass/fail rule to either replace the stored layer or discard the fragment [S1, unofficial]. The specific condition for the pass/fail rule is not given in the sources [S1, unofficial].
4. Normalization: After all fragments are processed, normalize the accumulated color by the accumulated weight [S1, unofficial]. Because no coverage-based opacity is involved, the normalized output is fully opaque wherever any surfel contributes [S1, unofficial].

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
        accumulated_color = (color[0] * weight, color[1] * weight, color[2] * weight)
        accumulated_weight = weight
    else:
        delta_d = abs(d - d_stored)
        
        if delta_d <= epsilon:
            accumulated_color = (accumulated_color[0] + color[0] * weight,
                                 accumulated_color[1] + color[1] * weight,
                                 accumulated_color[2] + color[2] * weight)
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
The method produces hard, opaque fringes along object boundaries because the normalized output is always fully opaque wherever any surfel contributes, even at silhouette pixels with low accumulated weight [S1, unofficial]. It exacerbates self-occlusion artifacts at concavities by incorrectly rejecting deeper surfels that should form a second layer, resulting in holes or translucent patches [S1, unofficial].

## Operating characteristics
Time Complexity: Not given in the sources [S1, unofficial].
Memory Complexity: Not given in the sources [S1, unofficial].
Strengths: Not given in the sources [S1, unofficial].
Limitations: Maintains only a single surface layer per pixel, causing visibility and occlusion artifacts in complex geometry [S1, unofficial].
When to use: Not given in the sources [S1, unofficial].
When not to use: Not given in the sources [S1, unofficial].

## Background
The Single-Layer Ternary Depth Test was introduced by Weyrich et al. in 2007 as a prior approach for rendering point-sampled surfaces [S1, unofficial]. It was later used as a baseline for comparison in ablation studies for 3DSS, where it was contrasted with depth-interval grouping [S1, unofficial].

## Sources
[S1] 3DSS: 3D Surface Splatting for Inverse Rendering, https://doi.org/10.48550/arxiv.2605.05876, fetched 2026-10-26 (unofficial)
