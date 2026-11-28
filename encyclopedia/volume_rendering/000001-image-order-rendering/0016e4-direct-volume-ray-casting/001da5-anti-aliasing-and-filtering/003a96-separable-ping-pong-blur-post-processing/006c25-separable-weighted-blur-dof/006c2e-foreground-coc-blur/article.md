# Foreground CoC Blur

Also called: fragBlurForFgCoc

## Overview
Foreground CoC Blur is a post-processing blur that calculates the Circle of Confusion (CoC) for foreground elements in depth-of-field rendering [S1, unofficial]. It distinguishes itself from standard alpha-weighted blurs by using the smoothstep difference between neighbor and center alpha values as weights [S1, unofficial]. It builds on [Separable Weighted Blur (DoF)](21017-separable-weighted-blur-dof.md) to isolate foreground blur contributions.

## How it works
**Inputs and Outputs:**
- Input: Alpha values of a center pixel and 11 neighboring pixels.
- Output: A normalized, weighted alpha sum representing the foreground CoC.

**Data Structures:**
- `G_WEIGHTS`: An array of floating-point weights; the exact size and values are not given, but `G_WEIGHTS[1]` is used as a multiplier [S1, unofficial].
- Sample pattern: 11 pixels (labeled A–K) sampled similarly to Alpha-Weighted Blur [S1, unofficial]. The exact spatial offsets are not specified in the sources [G].

**Steps:**
1. Sample the center pixel alpha ($\alpha_c$) and 11 neighbor alphas ($\alpha_i$) [S1, unofficial].
2. Calculate the weight for each neighbor: $w_i = \text{smoothstep}(-0.5, 0.0, \alpha_i - \alpha_c) \times \text{G\_WEIGHTS}[1]$ [S1, unofficial].
3. Accumulate the weighted alphas ($S = \sum \alpha_i w_i$) and the weights ($W = \sum w_i$) [S1, unofficial].
4. Normalize the result: $\text{result} = S / (W + 1\text{e-}4)$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def smoothstep(edge0, edge1, x):
    t = max(0.0, min(1.0, (x - edge0) / (edge1 - edge0)))
    return t * t * (3.0 - 2.0 * t)

def foreground_coc_blur(center_alpha, neighbor_alphas, g_weights):
    """
    center_alpha: float, alpha of the center pixel
    neighbor_alphas: list of floats, alpha values of 11 neighbors
    g_weights: list of floats, G_WEIGHTS array
    """
    sum_alpha = 0.0
    sum_weights = 0.0
    weight_scale = g_weights[1]
    
    for neighbor_alpha in neighbor_alphas:
        diff = neighbor_alpha - center_alpha
        w = smoothstep(-0.5, 0.0, diff) * weight_scale
        
        sum_alpha += neighbor_alpha * w
        sum_weights += w
        
    epsilon = 1e-4
    result = sum_alpha / (sum_weights + epsilon)
    
    return result
```

**Parameters:**
- `G_WEIGHTS[1]`: A constant weight multiplier. Value not given in the sources.
- `smoothstep` bounds: Fixed at `[-0.5, 0.0]` [S1, unofficial].
- Epsilon: `1e-4` [S1, unofficial].

**Edge Cases and Pitfalls:**
- Division by zero is prevented by adding `1e-4` to the denominator [S1, unofficial].
- Boundary checks for sample indices are not specified in the sources [G].

## Subtopics

### Foreground CoC Masking with Blur
This variant integrates foreground CoC calculation with background CoC masking [S1, unofficial]. It samples `_MainTex` for background CoC and `_FgOverlap` for foreground CoC [S1, unofficial]. If the background CoC is negligible ($< 0.01$), it returns the saturated difference between foreground and background CoC: $Result = (bgAndFgCoc < 0.01) \times \text{saturate}(fgOverlapCoc - bgAndFgCoc)$ [S1, unofficial].

### [Capture CoC](21033-capture-coc.md)
Capture CoC calculates the unsigned magnitude of the Circle of Confusion (CoC) for a given pixel depth. It serves as the foundational step in depth-of-field post-processing by determining how much a pixel should blur based on its distance from the focal plane. It differs from Foreground CoC Blur by providing the raw magnitude value (stored in alpha) rather than performing the weighted blur specific to foreground elements. See [Capture CoC](21033-capture-coc.md).

## Operating characteristics
- Time complexity is $O(1)$ per pixel due to the fixed 11-sample pattern [G].
- Memory complexity is $O(1)$ auxiliary space for accumulators [G].
- Strengths: isolates foreground blur contributions using alpha-difference weighting [S1, unofficial].
- Limitations: relies on alpha value distributions; artifacts may occur if alpha is noisy or discontinuous [G].

## Background
The method originates from the `dario-zubovic/metaballs` repository, which implements real-time post-processing effects including this blur [S1, unofficial]. It is integrated into a separable ping-pong blur pipeline for depth-of-field rendering [S1, unofficial].

## Sources
[S1] dario-zubovic/metaballs, https://github.com/dario-zubovic/metaballs, fetched 2026-10-26 (unofficial)
