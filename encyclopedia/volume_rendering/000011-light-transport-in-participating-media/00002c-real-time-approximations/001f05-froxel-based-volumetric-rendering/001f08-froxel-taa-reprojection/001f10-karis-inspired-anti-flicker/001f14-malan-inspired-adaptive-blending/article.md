# Malan-Inspired Adaptive Blending

Also called: Linear Blend Factor Mapping, Malan's Neighborhood Color Range Adaptation

## Overview
Malan-Inspired Adaptive Blending is a temporal anti-aliasing technique for froxel-based volumetric rendering that computes the blend factor $\alpha$ based on the spatial color extent of the local neighborhood [S1, unofficial]. It solves the problem of balancing ghosting and temporal stability by reducing the contribution of the previous frame in flat areas to accelerate convergence, and increasing temporal blurring in high-contrast or noisy regions [S1, unofficial]. Unlike its parent [Karis-Inspired Anti-Flicker](1924-karis-inspired-anti-flicker.md), which biases the signal towards the current frame to clip impulses, this method leaves the history unmodified and adjusts only the blending weight [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** The current frame's froxel color $C_{curr}$, the historical froxel color $C_{hist}$, the local neighborhood colors $\{C_{neighbor}\}$, and user-defined parameters $\alpha_{min}, \alpha_{max}, r_{min}, r_{max}$ [S1, unofficial].
*   **Output:** The blended froxel color $C_{out}$ and the computed blend factor $\alpha_{i,t}$ [S1, unofficial].

**Data Structures:**
*   **Local Neighborhood:** A set of RGB color vectors sampled from the current frame surrounding the target froxel [S1, unofficial]. The specific spatial radius or kernel shape is not defined in the sources [G: typically a small $3\times3$ or $5\times5$ pixel window].
*   **Color Space:** Samples may be transformed into CoCg or YCbCr spaces before bounding box computation, with results transformed back to RGB before storage [S1, unofficial].

**Steps:**
1.  **Sample Neighborhood:** Retrieve the color values for the pixels in the local neighborhood around the target froxel $i$ at frame $t$ [S1, unofficial].
2.  **Compute Extent ($r_{i,t}$):** Determine the minimum ($c_{min}$) and maximum ($c_{max}$) color vectors within the neighborhood [S1, unofficial]. Calculate the Euclidean length of the difference vector:
    $$r_{i,t} = |c_{max} - c_{min}|$$
    where $| \cdot |$ denotes the Euclidean norm (L2 distance) [S1, unofficial].
3.  **Compute Normalized Factor ($k$):** Map the extent $r_{i,t}$ to a factor $k$ using linear scaling between $r_{min}$ and $r_{max}$:
    $$k = 1 - \text{clamp}\left(\frac{r_{i,t} - r_{\min}}{r_{\max} - r_{\min}}, 0, 1\right)$$
    Here, $\text{clamp}(v, 0, 1)$ restricts $v$ to the range $[0, 1]$ [S1, unofficial].
4.  **Compute Blend Factor ($\alpha_{i,t}$):** Interpolate between the minimum and maximum blend factors using $k$:
    $$\alpha_{i,t} = (1 - k)\alpha_{\min} + k\alpha_{\max}$$
    This linear mapping is used because the original author did not specify a non-linear increase for smaller extents [S1, unofficial].
5.  **Blend:** Compute the final color using standard temporal accumulation:
    $$C_{out} = \alpha_{i,t} C_{curr} + (1 - \alpha_{i,t}) C_{hist}$$
    [G: standard temporal blending formula]. Transform the result back to the output color space if necessary [S1, unofficial].

**Parameters:**
*   $\alpha_{min} = 0.05$: Minimum blend factor (maximum history contribution) [S1, unofficial].
*   $\alpha_{max} = 0.50$: Maximum blend factor (minimum history contribution) [S1, unofficial].
*   $r_{min} = 0.01$: Lower bound for neighborhood extent scaling [S1, unofficial].
*   $r_{max} = 0.40$: Upper bound for neighborhood extent scaling [S1, unofficial].
*   These values were found to produce stable results in tested scenarios [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Division by Zero:** If $r_{min} = r_{max}$, the denominator in the clamp calculation becomes zero [S1, unofficial]. The sources define distinct values ($0.01$ and $0.40$), avoiding this [S1, unofficial].
*   **Flat Areas:** If the neighborhood is perfectly flat ($r_{i,t} \approx 0$), $k$ approaches $1$, resulting in $\alpha_{i,t} \approx \alpha_{max}$ [S1, unofficial]. This forces the history to refresh quickly, reducing ghosting in uniform regions [S1, unofficial].
*   **High Contrast/Noise:** If $r_{i,t} \ge r_{max}$, $k$ approaches $0$, resulting in $\alpha_{i,t} \approx \alpha_{min}$ [S1, unofficial]. This retains more history, introducing temporal blurring to stabilize sharp edges or noise [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def malan_adaptive_blending(c_curr, c_hist, neighborhood_colors, 
                            alpha_min=0.05, alpha_max=0.50, 
                            r_min=0.01, r_max=0.40):
    # Step 1 & 2: Compute extent r_i,t from neighborhood bounding box
    c_min = [float('inf'), float('inf'), float('inf')]
    c_max = [float('-inf'), float('-inf'), float('-inf')]
    for c in neighborhood_colors:
        for i in range(3):
            if c[i] < c_min[i]: c_min[i] = c[i]
            if c[i] > c_max[i]: c_max[i] = c[i]
            
    diff = [c_max[i] - c_min[i] for i in range(3)]
    r_i_t = math.sqrt(sum(d**2 for d in diff))
    
    # Step 3: Compute normalized factor k
    if r_max == r_min:
        k = 0.0
    else:
        normalized = (r_i_t - r_min) / (r_max - r_min)
        clamped = max(0.0, min(1.0, normalized))
        k = 1.0 - clamped
        
    # Step 4: Compute blend factor alpha_i,t
    alpha_i_t = (1.0 - k) * alpha_min + k * alpha_max
    
    # Step 5: Blend current and historical colors
    c_out = [alpha_i_t * c_curr[i] + (1.0 - alpha_i_t) * c_hist[i] for i in range(3)]
    return c_out
```

## Subtopics

### [Adaptive Blending](1983-adaptive-blending.md)
A broader history modification technique that caps the maximum amount of historical data replaced per frame. Unlike Malan-Inspired Adaptive Blending, which adjusts the blend factor based on color extent, Adaptive Blending modifies the history value itself to prevent ghosting from abrupt changes. Choose Malan-Inspired Adaptive Blending when you want to preserve the original history and only adjust the temporal weight based on spatial color variance.

## Operating characteristics
*   **Strengths:** Effectively reduces ghosting in flat areas by forcing faster history refresh [S1, unofficial]. Stabilizes noisy or high-contrast regions by retaining more history [S1, unofficial]. Provides an artist-controllable parameter for the trade-off between ghosting and flicker via $\alpha_{min}$ and $\alpha_{max}$ [S1, unofficial].
*   **Limitations:** Does not correct stale history values directly, unlike history rectification techniques [S1, unofficial].
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Not given in the sources.
*   **When to use:** Not given in the sources.
*   **When not to use:** Not given in the sources.

## Background
The method is inspired by the work of Malan [15], who proposed using the range of colors in a pixel's neighborhood to adaptively adjust the temporal blend factor [S1, unofficial]. This approach was implemented and evaluated in the context of temporal anti-aliasing for real-time froxel-based volumetric rendering [S1, unofficial]. The specific linear mapping formula and parameter values were detailed in a study on advances in volumetric temporal anti-aliasing [S1, unofficial]. The technique is part of a family of history rectification and blending adjustments designed to improve the visual quality of participating media rendering over time [S1, unofficial].

## Sources
[S1] Temporal Anti-Aliasing for Real-Time Froxel-Based Volumetric Rendering - Advances in volumetric temporal anti-aliasing for real-time rendering of participating media, https://openalex.org/W7218067689, fetched 2026-11-01 (unofficial)
