# Weighted Blended Order-Independent Transparency

Also called: OIT based implementation, WBOIT (Weighted Blended Order-Independent Transparency), WBOT Transparency Mode, Weighted Blended OIT

## Overview
Weighted Blended Order-Independent Transparency (WBOIT) is a weight-based sorting technique that approximates correct alpha compositing by performing blending in a single rendering pass, thereby avoiding the computational cost of explicit depth sorting [S1, unofficial][S2]. It modifies the standard blending equation to incorporate weights that estimate occlusion based on fragment transparency, trading precise depth accuracy for significantly reduced hardware overhead [S2].

## How it works
WBOIT modifies the standard alpha blending equation to incorporate a weight that estimates the occlusion contribution of fragments based on their transparency, allowing the hardware's fixed-function blending unit to approximate a sorted accumulation [S1, unofficial][S2].

**Inputs and Outputs:**
*   **Input:** A stream of translucent fragments with color $C_i$ and opacity $\alpha_i$, plus scene-specific weight parameters [S1, unofficial][S2].
*   **Output:** A final color buffer approximating the correct order-independent blend [S1, unofficial][S2].

**Data Structures:**
*   Standard frame buffer (color and depth) [S1, unofficial][S2].
*   Compositor viewport state tracking `wboit_compositor_active_` and `compositor_viewport_` [S1, unofficial].

**Parameters:**
*   **Weight scale:** Configured via `setWboitWeightScale` from `wboit_weight_scale_property_` [S1, unofficial].
*   **Weight exponent:** Configured via `setWboitWeightExponent` from `wboit_weight_exponent_property_` [S1, unofficial].
*   **Alpha discard threshold:** Configured via `setWboitAlphaDiscard` from `wboit_alpha_discard_property_` [S1, unofficial].
*   Typical values are not provided in the sources [S1, unofficial][S2].

**Algorithm Steps:**
1.  **Enable:** If `transparency_mode_property` is 1, call `ensureDefined()`, `enable(vp)`, set `compositor_viewport_ = vp`, and set `wboit_compositor_active_ = true` [S1, unofficial].
2.  **Weight Calculation:** For each fragment $i$, compute a weight $w_i$ using the weight scale and exponent to estimate occlusion as a function of the transparency of the rest of the objects [S1, unofficial][S2].
3.  **Single-Pass Blending:** Blend fragments into the color buffer in a single pass using standard hardware blending, where the fragment's contribution is modulated by $w_i$ [S1, unofficial][S2].
4.  **Disable:** If WBOIT is disabled and was active, call `disable(compositor_viewport_)` and set `wboit_compositor_active_ = false` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def apply_wboit_single_pass(fragments, weight_scale, weight_exponent, alpha_discard_threshold):
    """
    Simulates the single-pass WBOIT blending loop.
    Note: The exact mathematical formula for the weight w_i is not provided in the sources [S1, unofficial][S2].
    Standard blending accumulation is used here [G].
    """
    # Initialize frame buffer state
    accumulated_color = [0.0, 0.0, 0.0]
    accumulated_alpha = 0.0

    for frag in fragments:
        # Skip fragments below the alpha discard threshold
        if frag.alpha < alpha_discard_threshold:
            continue

        # Calculate weight to estimate occlusion
        # Formula not given in sources; placeholder logic uses scale/exponent [S1, unofficial][S2]
        weight = weight_scale * (frag.alpha ** weight_exponent)

        # Weighted fragment contribution
        w_color = [c * weight for c in frag.color]
        w_alpha = frag.alpha * weight

        # Standard over-operator accumulation [G]
        out_alpha = w_alpha + accumulated_alpha * (1.0 - w_alpha)
        if out_alpha > 0.0:
            out_color = [
                (w_color[i] + accumulated_color[i] * (1.0 - w_alpha)) / out_alpha
                for i in range(3)
            ]
        else:
            out_color = [0.0, 0.0, 0.0]

        accumulated_color = out_color
        accumulated_alpha = out_alpha

    return accumulated_color, accumulated_alpha
```

**Edge Cases and Pitfalls:**
*   **Color Clusters:** Rendering quality decreases significantly in cases with close clusters of several transparent elements of different colors [S2].
*   **Weight Tuning:** The assignment of weights requires expert adjustment for each scene according to the distribution of elements if quality sorting is to be achieved [S2].
*   **Medical Data Integrity:** Due to the potential for inaccurate depth sorting, this method is considered unsuitable for medical image visualization where preserving the integrity of underlying data is essential [S2].

## Subtopics

### Per-Pixel Linked Lists for Order Independent Transparency
An Order Independent Transparency (OIT) technique that operates at the fragment level to analyze and sort all object depths linked to each pixel, avoiding layer-based grouping and ensuring precise blending suitable for volumetric data fusion [S2]. Unlike Weighted Blended Order-Independent Transparency, which performs sorting in a single pass but suffers from quality degradation in clusters of transparent elements and requires expert weight adjustment, Per-Pixel Linked Lists provide accurate sorting without scene-specific weight tuning [S2]. It requires two rendering passes per frame regardless of scene complexity [S2]. Choose it when exact depth sorting is critical for volumetric fusion; link to [Per-Pixel Linked Lists](...).

## Operating characteristics
*   **Time Complexity:** Single pass rendering, making it computationally less intensive than sorting-based methods [S1, unofficial][S2].
*   **Memory Complexity:** Low; it does not require additional memory for linked lists or depth peeling buffers [S1, unofficial][S2].
*   **Performance:** The algorithm is not computationally or memory-intensive [S2].
*   **Strengths:** High performance due to single-pass execution; no need for depth-based sorting [S1, unofficial][S2].
*   **Limitations:** Rendering quality varies greatly depending on the scene type; poor accuracy in clusters of different-colored transparent elements; requires manual weight tuning per scene [S2].
*   **When to use:** Resource-constrained CPUs and weaker GPUs where exact occlusion is less critical than performance [S1, unofficial][S2].
*   **When not to use:** Medical image visualization or any application requiring high accuracy in depth range and color blend integrity [S2].

## Background
The method is identified as a prominent algorithm within the OIT techniques, specifically categorized as weight-based sorting [S2]. It is cited as reference [7] in the medical visualization literature, though the specific original paper title is not provided in the notes [S2]. The method is implemented in the RVizSplat plugin for real-time splat rendering [S1, unofficial]. It was evaluated for medical image visualization but rejected in favor of Per-Pixel Linked Lists due to accuracy requirements [S2].

## Sources
[S1] splat-ros/RVizSplat, https://github.com/splat-ros/RVizSplat, fetched 2026-10-27 (unofficial)
[S2] Hybrid visualisation of medical image: Surface and volume rendering ..., https://www.sciencedirect.com/science/article/pii/S0097849325002948, fetched 2026-10-28
