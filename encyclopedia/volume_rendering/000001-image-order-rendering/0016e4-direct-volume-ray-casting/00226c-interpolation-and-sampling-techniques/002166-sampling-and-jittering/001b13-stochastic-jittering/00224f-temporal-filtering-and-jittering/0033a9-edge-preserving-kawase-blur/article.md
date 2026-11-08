# Edge-Preserving Kawase Blur

Also called: Edge-Aware Kawase Blur, Kawase Blur with Blue Noise Jittering

## Overview
Edge-Preserving Kawase Blur is a post-processing technique designed to remove dither patterns introduced by blue noise jittering in volume rendering while preserving cloud edge details [S1, unofficial]. It distinguishes itself from standard box blurs by using edge-aware weighting to prevent the smoothing of sharp volumetric boundaries, and from temporal accumulation methods by operating on a single frame without requiring reprojection of previous frames [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   Input: A rendered color buffer containing dithered noise and volumetric data (depth/luminosity).
*   Output: A smoothed color buffer with reduced dithering artifacts but preserved edges.

**Data Structures:**
*   Texture/Sampler: Access to the input color buffer and auxiliary buffers (depth or luminance) for edge detection.
*   Blue Noise Texture: A pre-computed texture containing blue noise samples used to jitter sample positions.

**Parameters:**
*   Blur Radius/Passes: The number of diagonal passes and the distance of samples from the center. Not given in the sources.
*   Edge Threshold: A sensitivity parameter for luminosity/depth difference. Not given in the sources.
*   Extra Blur Factor: A multiplier for blurring distant surfaces. Not given in the sources.

**Steps:**
1.  **Initialize Blue Noise Jitter:** For each pixel, retrieve a blue noise value to jitter the sample positions. This prevents the introduction of new noticeable dither patterns that a standard grid-based Kawase blur would produce [S1, unofficial].
2.  **Sample Diagonals:** For each Kawase blur pass, pick 4 diagonal samples per pixel relative to the center [S1, unofficial].
3.  **Edge-Aware Weighting:** Calculate the weight for each sample based on the difference in luminosity or depth between the sample and the center point [S1, unofficial].
    *   Let $C_{center}$ be the center pixel's luminosity/depth.
    *   Let $C_{sample}$ be the sample's luminosity/depth.
    *   Let $\Delta = |C_{center} - C_{sample}|$.
    *   Samples with very different values ($\Delta$ is large) are weighed less to preserve edges [S1, unofficial].
    *   The exact weighting function is not specified in the sources.
4.  **Depth-Based Extra Blur:** Apply extra blurring to surfaces that are further away. This is done to smooth distant noise but is calibrated to not create a depth of field effect [S1, unofficial].
5.  **Accumulate and Normalize:** Sum the weighted samples and divide by the sum of weights to produce the final pixel color.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def edge_preserving_kawase_blur(input_color, input_depth, blue_noise_tex, width, height, passes):
    # Get current pixel coordinates and values
    center_color = input_color[uv]
    center_depth = input_depth[uv]
    
    # Get blue noise jitter value for this pixel
    # The exact mapping of blue noise to sample offset is not specified
    jitter = blue_noise_tex[uv] 
    
    total_weighted_color = vec4(0.0)
    total_weight = 0.0
    
    for pass_idx in range(passes):
        # Determine sample offsets for 4 diagonal samples
        # Standard Kawase offsets depend on pass_idx
        # Offsets are jittered by blue noise
        offsets = get_kawase_offsets(pass_idx, jitter)
        
        for offset in offsets:
            sample_uv = uv + offset
            sample_color = input_color[sample_uv]
            sample_depth = input_depth[sample_uv]
            
            # Edge detection based on depth difference
            depth_diff = abs(sample_depth - center_depth)
            
            # Weight calculation: samples with large depth difference get less weight
            # Exact formula not provided in sources [G: Assume a decay function like exp(-depth_diff * sensitivity)]
            weight = calculate_edge_weight(depth_diff)
            
            # Apply extra blur for distant surfaces
            # Not given in the sources for the exact implementation of "extra blurring"
            # [G: Assume weight is further modulated by distance if required]
            
            total_weighted_color += sample_color * weight
            total_weight += weight
            
    # Normalize
    if total_weight > 0.0:
        return total_weighted_color / total_weight
    else:
        return center_color
```

## Subtopics

### Median Filter Denoising
A post-processing technique used to reduce 'Salt and Pepper' noise caused by unpredictable precision errors in raycasting that result in wrong or missing pixel colors [S2, unofficial]. It differs from Edge-Preserving Kawase Blur by targeting discrete precision errors rather than dithering patterns, using a sorting-based median approach instead of weighted averaging [S2, unofficial]. Choose this when noise manifests as isolated incorrect pixel colors due to precision issues [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** The sources do not provide a specific complexity analysis for Edge-Preserving Kawase Blur. However, the related Median Filter Denoising is noted as $O(1)$ because it iterates through a fixed array of length 9 [S2, unofficial]. Kawase blur is generally considered cheap, involving a fixed number of diagonal samples per pass [S1, unofficial].
*   **Memory Complexity:** Requires storage for the input buffer, output buffer, and a blue noise texture. Not given in the sources.
*   **Strengths:** Cheap and effective at removing dither patterns without the need for temporal reprojection [S1, unofficial]. Preserves cloud edge details better than standard box blurs [S1, unofficial].
*   **Limitations:** May introduce banding patterns if not jittered [S1, unofficial]. Requires careful tuning of edge thresholds and extra blur factors to avoid depth of field effects [S1, unofficial].
*   **When to use:** When using blue noise jittering in volume rendering and needing to smooth the result without temporal accumulation [S1, unofficial].
*   **When not to use:** When temporal accumulation is viable and the scene is static, as TAA can be more effective [S1, unofficial].

## Background
Edge-Preserving Kawase Blur is a technique discussed in the context of efficient volumetric cloud rendering [S1, unofficial]. It builds on the standard Kawase blur, which is a type of box-blur that produces banding patterns [S1, unofficial]. The method was adapted to include blue noise jittering and edge-aware weighting to suit the fuzzy nature of cloud surfaces, where temporal accumulation techniques may not work as well as they do for hard surfaces [S1, unofficial].

## Sources
[S1] Efficient volumetric clouds - Resources - three.js forum, https://discourse.threejs.org/t/efficient-volumetric-clouds/66067, fetched 2026-10-29 (unofficial)
[S2] Raycasting-basiertes Voxelrendering, https://serwiss.bib.hs-hannover.de/frontdoor/index/index/docId/2250, fetched 2026-10-25 (unofficial)
