# Separable Weighted Blur (DoF)

Also called: Alpha-Weighted Gaussian Blur, fragBlurAlphaWeighted, fragBlurDark, fragBlurWeighted

## Overview
A 2D post-processing blur that weights neighboring samples by their alpha values to preserve sharp edges in depth-of-field rendering [S1, unofficial]. It differs from its parent, [Separable Ping-Pong Blur Post-Processing](8804-separable-ping-pong-blur-post-processing.md), by using a predefined non-separable kernel (typically 11-tap) where weights are dynamically modulated by pixel alpha, rather than applying independent 1D horizontal and vertical passes [S1, unofficial]. This approach prevents alpha bleeding across object boundaries, a common artifact in standard Gaussian blurs [S1, unofficial].

## How it works
**Inputs:** A source texture containing RGBA color data, UV offset coordinates precomputed for the kernel taps, and Gaussian weight constants.
**Outputs:** A blurred RGBA texture.

**Data Structures:**
The vertex shader or compute kernel populates a `v2fBlur` struct with UV coordinates for the kernel taps. The layout is:
- `uv`: Center pixel UV.
- `uv01`: UVs for taps B and C.
- `uv23`: UVs for taps D and E.
- `uv45`: UVs for taps F and G.
- `uv67`: UVs for taps H and I.
- `uv89`: UVs for taps J and K.

**Parameters:**
- `G_WEIGHTS`: Array of 6 floats defining the spatial decay: `{1.0, 0.8, 0.675, 0.5, 0.2, 0.075}` [S1, unofficial].
- `ALPHA_WEIGHT`: Multiplier for alpha modulation, typically `2.0` [S1, unofficial].
- `EPSILON`: Small constant for numerical stability, typically `1e-4f` [S1, unofficial].

**Steps:**
1. Sample the center pixel $A$ at `uv`.
2. Sample the 10 neighbor pixels ($B$ through $K$) using the corresponding `uv` fields in `v2fBlur`.
3. Initialize `sumColor` (RGB) to zero and `sumWeight` to zero.
4. For each neighbor pixel $P \in \{B, \dots, K\}$ with index $i$ (where $B$ is index 1, $C$ is index 2, etc.):
   a. Retrieve the spatial weight $w_{spatial} = G\_WEIGHTS[i]$.
   b. Calculate the alpha-modulated weight: $w_{alpha} = \text{saturate}(ALPHA\_WEIGHT \cdot P.a)$.
   c. Calculate total weight: $w = w_{alpha} \cdot w_{spatial}$.
   d. Accumulate color: $sumColor += P.rgb \cdot w$.
   e. Accumulate weight: $sumWeight += w$.
5. Normalize the accumulated color: $result.rgb = sumColor / (sumWeight + EPSILON)$.
6. Set the output alpha to the center pixel's alpha: $result.a = A.a$.
7. **Edge Case:** If the center pixel alpha is negligible ($A.a < 1e-2f$), restore the original RGB color from $A$ to prevent darkening transparent areas [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def separable_weighted_blur_doF(center_uv, neighbor_uvs, g_weights, alpha_weight, epsilon):
    # Sample center
    sample_a = texture_sample(center_uv)
    
    sum_color = vec3(0.0)
    sum_weight = 0.0
    
    # Process 10 neighbors (indices 1 to 10 map to G_WEIGHTS indices 1 to 10? 
    # Note: G_WEIGHTS has 6 elements. The notes imply mapping neighbors to these weights.
    # Assuming a specific mapping where neighbors B-K use weights 1.0, 0.8, 0.675, 0.5, 0.2, 0.075
    # and potentially repeating or clamping. The notes list 6 weights for 10 taps.
    # We assume the weights are applied cyclically or the first 6 are used for the first 6 taps,
    # and the remaining 4 taps use the last weight or zero. 
    # However, standard implementation usually maps specific taps to specific weights.
    # Given "11-tap blur (center + 10 neighbors)" and "G_WEIGHTS[6]", 
    # we assume the weights are distributed. Let's assume a direct mapping for the first 6, 
    # and the rest might be zero or reuse. 
    # To be safe and implementable: we iterate 10 neighbors.
    
    for i in range(10):
        neighbor_uv = neighbor_uvs[i]
        sample_p = texture_sample(neighbor_uv)
        
        # Map index i to weight. 
        # If i < len(G_WEIGHTS), use G_WEIGHTS[i]. 
        # If i >= len(G_WEIGHTS), use G_WEIGHTS[-1] (0.075) or 0? 
        # The notes don't specify the mapping for taps 7-10. 
        # We will assume the weights are applied to the first 6 taps, 
        # and taps 7-10 have zero weight or are not present in a 6-weight array context.
        # However, a 11-tap blur usually implies a cross or diamond shape.
        # Let's assume the weights array is indexed by tap distance/order.
        
        if i < len(g_weights):
            w_spatial = g_weights[i]
        else:
            w_spatial = 0.0 # Or reuse last weight? Notes are ambiguous. 
                            # "predefined Gaussian-like weight array" of size 6.
                            # We assume taps beyond index 5 have 0 weight or are not sampled.
                            # But it says 11-tap. 
                            # Let's assume the weights are: [1.0, 0.8, 0.675, 0.5, 0.2, 0.075]
                            # and applied to taps B, C, D, E, F, G. 
                            # Taps H, I, J, K might have 0 weight or use the last weight.
                            # Given "11-tap", we sample 10. 
                            # We will assume the weights are applied to the first 6 neighbors, 
                            # and the remaining 4 have 0 weight (effectively 7-tap).
                            # Or, more likely, the weights are reused. 
                            # Without explicit mapping, we assume the first 6 neighbors get the 6 weights.
                            # The remaining 4 neighbors get 0 weight.
        
        w_alpha = clamp(alpha_weight * sample_p.a, 0.0, 1.0)
        w = w_alpha * w_spatial
        
        sum_color += sample_p.rgb * w
        sum_weight += w
        
    # Normalize
    result_rgb = sum_color / (sum_weight + epsilon)
    result_a = sample_a.a
    
    # Edge case for transparent center
    if sample_a.a < 0.01:
        result_rgb = sample_a.rgb
        
    return vec4(result_rgb, result_a)
```

## Subtopics

### 4-Tap Alpha-Weighted Blur
A 4-tap blur used for spawning low-resolution effects, weighting neighbors by their alpha [S1, unofficial]. It samples the center and 4 diagonal neighbors at an offset of $0.75 \times \text{TexelSize}$ [S1, unofficial]. Weights are calculated as $\text{saturate}(10.0 \cdot \text{alpha})$ [S1, unofficial]. If the total weight is significant ($\text{center.a} \cdot \text{sumWeights} \cdot 8.0 > 1e-5f$), it returns the weighted average; otherwise, it returns the original color [S1, unofficial].

### [Insane HQ Bokeh Blur](21018-insane-hq-bokeh-blur.md)
Insane HQ Bokeh Blur is a screen-space post-processing technique that simulates depth-of-field (DoF) or tilt-shift effects using Poisson-disk sampling with a disc kernel. It differs from its parent, [Separable Weighted Blur (DoF)](21017-separable-weighted-blur-dof.md), by employing a spatially varying kernel weight to control blur radius based on screen position and using a fixed set of Poisson-disk samples for uniform coverage, rather than a separable 1D convolution. It handles stereo rendering and alpha-weighted accumulation to preserve sharp edges in the final image.

### [Foreground CoC Blur](21024-foreground-coc-blur.md)
Foreground CoC Blur is a specialized post-processing blur designed to calculate the Circle of Confusion (CoC) for foreground elements in depth-of-field rendering. It distinguishes itself from standard alpha-weighted blurs by using the smoothstep difference between neighbor and center alpha values as weights, effectively isolating foreground blur contributions. It serves as a core component within the Separable Weighted Blur (DoF) pipeline, specifically addressing the need to render sharp foreground objects against blurred backgrounds.

## Operating characteristics
**Time Complexity:** $O(1)$ per pixel, as the kernel size is fixed (11 taps).
**Memory Complexity:** $O(1)$ auxiliary memory per pixel.
**Performance:** Fast due to fixed small kernel and no separability overhead.
**Strengths:** Preserves alpha edges, preventing halo artifacts around transparent objects.
**Limitations:** Not separable, so it cannot be optimized into two 1D passes. The fixed 11-tap structure may not adapt to varying CoC sizes without multiple passes or dynamic UV scaling.
**When to use:** When alpha edges are critical and DoF is applied to transparent objects.
**When not to use:** When performance is paramount and alpha edges are not an issue (use standard Gaussian).

## Background
Originated in real-time rendering pipelines for depth-of-field effects, specifically within Unity-based projects [S1, unofficial]. The method is documented in the `metaballs` repository by dario-zubovic [S1, unofficial].

## Sources
[S1] dario-zubovic/metaballs, https://github.com/dario-zubovic/metaballs, fetched 2026-10-26 (unofficial)
