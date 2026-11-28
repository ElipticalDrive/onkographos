# Weighted Blended Order-Independent Transparency

Also called: OIT based implementation, WBOIT (Weighted Blended Order-Independent Transparency), WBOT Transparency Mode, Weighted Blended OIT

## Overview
Weighted Blended Order-Independent Transparency (WBOIT) is a transparency rendering technique that bypasses sorting entirely to achieve order-independent blending in a single pass [S1, unofficial][S2]. It solves the non-commutative nature of standard alpha blending by modifying fragment weights in the translucency equation to estimate occlusion based on the transparency of surrounding objects [S2]. Unlike [Improved Order Independent Transparency for Opaque Kernels](4562-improved-order-independent-transparency-for-opaque-kernels.md) or [Hardware-Z Cull Depth Peeling](13947-hardware-z-cull-depth-peeling.md), which rely on sorting or sequential depth extraction, WBOIT modifies blending weights to approximate depth-sorted accumulation without explicit ordering, making it suitable for resource-constrained environments such as integrated graphics or CPUs [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A stream of splats (or fragments) with position, color, opacity ($\alpha$), and covariance data.
*   **Output:** A final color buffer where transparent objects are blended in an order-independent manner.

**Data Structures:**
In implementations like `gsplat_rviz_plugin`, splats are packed into a `SplatGPU` structure for GPU consumption [S1, unofficial]:
*   `center[3]`: Float3 position.
*   `alpha`: Float opacity.
*   `covA[3]`: Upper-triangle covariance components $\{v_{11}, v_{12}, v_{13}\}$.
*   `covB[3]`: Remaining covariance components $\{v_{22}, v_{23}, v_{33}\}$.
*   `sh[16][4]`: Spherical Harmonics coefficients (RGB + padding).
These are packed into a `Texture Buffer Object` (TBO) using half-precision floats for covariance components to save memory [S1, unofficial].

**Parameters:**
*   `wboit_weight_scale`: Controls the scaling of the weight function [S1, unofficial].
*   `wboit_weight_exponent`: Controls the steepness of the weight falloff [S1, unofficial].
*   `wboit_alpha_discard`: Threshold below which fragments are discarded to reduce noise [S1, unofficial].

**Algorithm Steps:**
1.  **Initialization:** If the `transparency_mode_property` option is 1, enable WBOIT by calling `ensureDefined()`, `enable(vp)`, setting `compositor_viewport_ = vp`, and setting `wboit_compositor_active_ = true` [S1, unofficial].
2.  **Weight Calculation:** For each fragment, calculate a weight $w$ based on its depth and the estimated transparency of the scene [S2]. The algorithm uses weights in the translucency equation running through the scene to estimate the occlusion for each depth as a function of the transparency of the rest of the objects considered [S2].
3.  **Blending:** Perform blending using the calculated weight $w$ instead of, or in addition to, the fragment's opacity $\alpha$ [S2]. The blending equation is modified to be commutative [S2]. Standard blending follows $C_{out} = C_{frag} \alpha_{frag} + C_{prev} (1 - \alpha_{frag})$ [G]. WBOIT modulates the contribution of the fragment to approximate its position in the depth-sorted order [S2].
4.  **Discard:** Fragments with $\alpha < \text{wboit\_alpha\_discard}$ are discarded to improve performance and reduce noise [S1, unofficial].
5.  **Termination:** If WBOIT is disabled and was active, call `disable(compositor_viewport_)` and set `wboit_compositor_active_ = false` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def wboit_process_fragment(fragment, wboit_alpha_discard, wboit_weight_scale, wboit_weight_exponent, framebuffer_color):
    # 1. Discard fragments below alpha threshold [S1, unofficial]
    if fragment.alpha < wboit_alpha_discard:
        return framebuffer_color

    # 2. Calculate weight based on depth and scene transparency [S2]
    # Sources do not provide the exact weight formula [S2]
    # Standard implementations typically use a power-law or exponential decay of depth/accumulated opacity
    w = fragment.alpha * (wboit_weight_scale * fragment.depth) ** wboit_weight_exponent

    # 3. Blend using the calculated weight to approximate depth-sorted order [S2]
    blended_color = fragment.color * w + framebuffer_color * (1.0 - w)
    return blended_color
```

**Edge Cases and Pitfalls:**
*   **Cluster Quality:** Rendering quality degrades in scenes with close clusters of several transparent elements of different colors [S2].
*   **Expert Tuning:** Weight assignment requires expert adjustment for each specific scene to achieve quality sorting [S2].
*   **Precision:** The method is an approximation and may not preserve the exact integrity of underlying data, which is critical for medical visualization [S2].

## Subtopics

### Per-Pixel Linked Lists for Order Independent Transparency
An Order Independent Transparency (OIT) technique that operates at the fragment level to analyze and sort all object depths linked to each pixel, avoiding layer-based grouping and ensuring precise blending suitable for volumetric data fusion [S2]. It requires two rendering passes per frame, regardless of scene complexity, to achieve real-time performance [S2]. Unlike Depth Peeling, it avoids layer-based grouping, and unlike WBOIT, it provides accurate sorting without scene-specific weight tuning [S2].

## Operating characteristics
*   **Time Complexity:** Single pass rendering, making it computationally efficient compared to sorting-based methods ($O(N \log N)$) or depth peeling ($O(N)$ passes) [S1, unofficial][S2].
*   **Memory Complexity:** Low memory overhead as it does not require storing linked lists or multiple depth buffers [S2].
*   **Performance:** Suitable for resource-constrained CPUs and weaker GPUs (e.g., integrated graphics) [S1, unofficial].
*   **Strengths:** Fast, single-pass, low memory usage [S2].
*   **Limitations:** Visual artifacts in complex transparent clusters; requires manual tuning of weights per scene; less accurate than sorting-based methods [S2].
*   **When to Use:** Resource-constrained CPUs and weaker GPUs (e.g., integrated graphics) where approximate transparency is acceptable [S1, unofficial].
*   **When Not to Use:** Medical visualization or scientific data inspection where precise depth sorting and color integrity are essential [S2].

## Background
WBOIT was introduced as a weight-based sorting OIT technique to reduce the computational cost of rendering by performing sorting in a single pass without depth-based sorting [S2]. It is used in applications like `gsplat_rviz_plugin` to enable transparency rendering on constrained hardware [S1, unofficial]. The method is particularly relevant in fields like medical image visualization, where it is often compared against more accurate but slower methods like Per-Pixel Linked Lists [S2].

## Sources
[S1] splat-ros/RVizSplat, https://github.com/splat-ros/RVizSplat, fetched 2026-10-27 (unofficial)
[S2] Hybrid visualisation of medical image: Surface and volume rendering ..., https://www.sciencedirect.com/science/article/pii/S0097849325002948, fetched 2026-10-28
