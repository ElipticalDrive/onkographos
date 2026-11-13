# Weighted Blended Order-Independent Transparency

Also called: OIT based implementation, WBOIT (Weighted Blended Order-Independent Transparency), WBOT Transparency Mode, Weighted Blended OIT

## Overview
Weighted Blended Order-Independent Transparency (WBOIT) is a transparency rendering technique that bypasses explicit depth sorting to achieve order-independent blending, primarily intended for resource-constrained CPUs and weaker GPUs (e.g., integrated graphics) [S1, unofficial]. It addresses the non-commutative nature of translucent compositing by modifying fragment weights to implicitly encode depth information, allowing standard alpha blending to approximate sorted results [S2]. Unlike siblings such as [Improved Order Independent Transparency for Opaque Kernels](4562-improved-order-independent-transparency-for-opaque-kernels.md) or [Hardware-Z Cull Depth Peeling](13947-hardware-z-cull-depth-peeling.md), WBOIT performs sorting in a single pass without depth-based sorting, trading visual accuracy for computational efficiency [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A stream of splats or fragments, each with position, color, opacity ($\alpha$), and potentially depth.
*   **Output:** A final color buffer where fragments are blended in a manner that approximates correct depth ordering without explicit sorting.

**Data Structures:**
The method relies on standard framebuffer attachments (color, depth, stencil) and does not require complex auxiliary structures like linked lists or multiple render targets for accumulation, distinguishing it from Per-Pixel Linked Lists [S2]. Configuration is managed via specific properties:
*   `wboit_weight_scale_property_`: Controls the weight scale factor [S1, unofficial].
*   `wboit_weight_exponent_property_`: Controls the weight exponent [S1, unofficial].
*   `wboit_alpha_discard_property_`: Threshold for discarding fragments based on alpha [S1, unofficial].

**Parameters:**
*   **Weight Scale ($w_{scale}$):** A multiplier applied to the depth-based weight [S1, unofficial].
*   **Weight Exponent ($w_{exp}$):** An exponent applied to the depth term to sharpen the falloff [S1, unofficial].
*   **Alpha Discard Threshold ($\alpha_{thresh}$):** Fragments with opacity below this value are discarded to reduce overdraw [S1, unofficial].

**Algorithm Steps:**
1.  **Initialization:** If the transparency mode is set to WBOIT, enable the compositor viewport. This involves calling `ensureDefined()`, `enable(vp)`, setting `compositor_viewport_ = vp`, and setting `wboit_compositor_active_ = true` [S1, unofficial].
2.  **Weight Calculation:** For each fragment, calculate a weight $w$ based on its depth $z$ relative to the camera and the scene's transparency distribution. The weight is designed to estimate occlusion as a function of the transparency of the rest of the objects considered [S2]. The specific formula is not provided in the sources, but it typically takes the form $w = \exp(-w_{scale} \cdot (z - z_{ref})^{w_{exp}})$ or similar, where $z_{ref}$ is a reference depth [G].
3.  **Blending:** Perform standard alpha blending using the calculated weight $w$ and the fragment's opacity $\alpha$. The effective opacity used for blending is often modified by the weight to simulate depth sorting [S2].
4.  **Termination:** If WBOIT is disabled and was previously active, call `disable(compositor_viewport_)` and set `wboit_compositor_active_ = false` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Configuration parameters
wboit_weight_scale = get_property("wboit_weight_scale_property_")
wboit_weight_exponent = get_property("wboit_weight_exponent_property_")
wboit_alpha_discard = get_property("wboit_alpha_discard_property_")

def enable_wboit(vp):
    ensure_defined()
    enable(vp)
    compositor_viewport = vp
    wboit_compositor_active = True

def disable_wboit(vp):
    disable(vp)
    wboit_compositor_active = False

def fragment_shader(position, depth, color, alpha):
    # Discard low-alpha fragments
    if alpha < wboit_alpha_discard:
        discard()
    
    # Calculate weight based on depth
    # Note: Exact formula not provided in sources; [G] indicates assumed standard WBOIT weight calculation
    # w = exp(-wboit_weight_scale * (depth - reference_depth) ** wboit_weight_exponent)
    # For implementation, reference_depth is often the near plane or a scene-specific value
    weight = calculate_wboit_weight(depth, wboit_weight_scale, wboit_weight_exponent)
    
    # Blend using weight and alpha
    # Standard alpha blend: result = src * src_alpha + dst * (1 - src_alpha)
    # Here, src_alpha is effectively modulated by weight
    effective_alpha = alpha * weight
    blended_color = color * effective_alpha + current_pixel_color * (1 - effective_alpha)
    
    return blended_color

def calculate_wboit_weight(depth, scale, exponent):
    # Placeholder for the specific weight function
    # The sources state weights are used to estimate occlusion [S2]
    # A common implementation uses exponential falloff from the camera
    return exp(-scale * (depth ** exponent))
```

**Edge Cases and Pitfalls:**
*   **Cluster Quality:** Rendering quality decreases significantly in cases with close clusters of several transparent elements of different colors [S2].
*   **Expert Tuning:** Weight assignment requires expert adjustment for each scene according to the distribution of elements to achieve quality sorting [S2].
*   **Medical Data:** Due to the need for accuracy in depth ranges for medical segmented 3D models, this method is often deemed unsuitable for such purposes [S2].

## Subtopics

### Per-Pixel Linked Lists for Order Independent Transparency
An Order Independent Transparency (OIT) technique that operates at the fragment level to analyze and sort all object depths linked to each pixel, avoiding layer-based grouping and ensuring precise blending suitable for volumetric data fusion [S2]. It requires two rendering passes per frame, regardless of scene complexity, to achieve real-time performance while maintaining accurate depth sorting [S2]. Unlike Depth Peeling, which requires separate passes for each depth layer, and Dual Depth Peeling, which may store elements at different depths within the same layer, Per-Pixel Linked Lists provide precise per-fragment depth sorting without layer-based grouping [S2]. Unlike WBOIT, which performs sorting in a single pass but suffers from quality degradation in clusters and requires expert weight tuning, Per-Pixel Linked Lists provide accurate sorting without scene-specific weight tuning [S2].

## Operating characteristics
*   **Time Complexity:** Single pass rendering, making it computationally efficient compared to multi-pass peeling methods [S2].
*   **Memory Complexity:** Low memory overhead as it does not require storing linked lists or multiple depth layers [S2].
*   **Performance:** Suitable for resource-constrained CPUs and weaker GPUs (e.g., integrated graphics) [S1, unofficial].
*   **Strengths:** Simple implementation, low computational cost, no need for depth-based sorting [S2].
*   **Limitations:** Visual artifacts in complex scenes with clustered transparent objects; requires manual tuning of weights per scene [S2].
*   **When to Use:** Real-time applications on low-end hardware where approximate transparency is acceptable [S1, unofficial].
*   **When Not to Use:** Applications requiring high precision in depth sorting, such as medical image visualization [S2].

## Background
WBOIT was introduced as a weight-based sorting OIT technique to reduce computational cost by performing sorting in a single pass without depth-based sorting [S2]. It is used in various visualization tools, such as the RViz plugin for ROS, where it can be activated via the 'Advanced' options [S1, unofficial]. The method is particularly relevant in scenarios where hardware limitations prevent the use of more accurate but expensive OIT techniques like Per-Pixel Linked Lists [S2].

## Sources
[S1] splat-ros/RVizSplat, https://github.com/splat-ros/RVizSplat, fetched 2026-10-27 (unofficial)
[S2] Hybrid visualisation of medical image: Surface and volume rendering ..., https://www.sciencedirect.com/science/article/pii/S0097849325002948, fetched 2026-10-28
