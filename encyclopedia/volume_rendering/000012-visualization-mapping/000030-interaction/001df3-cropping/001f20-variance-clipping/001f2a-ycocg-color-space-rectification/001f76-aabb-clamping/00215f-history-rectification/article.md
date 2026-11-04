# History Rectification

Also called: AABB Clipping

## Overview
History rectification is a class of Temporal Anti-Aliasing (TAA) techniques that correct historical color values before blending by restricting the reprojected history to a region defined by current-frame neighborhood samples [S1, unofficial]. It solves the problem of temporal lag and ghosting during abrupt scene changes (such as light introduction or sudden movement) where a baseline Exponential Moving Average (EMA) fails to react quickly enough [S1, unofficial]. What sets it apart from its siblings and nearest alternatives is the use of a spatial neighborhood of current-frame samples to define a "color extent" or bound; if the historical value falls outside this extent, it is considered stale and is corrected (clamped or clipped) prior to blending [S1, unofficial]. This approach builds on [AABB Clamping](2019-aabb-clamping.md), which uses a simpler Axis-Aligned Bounding Box approximation, whereas History Rectification encompasses more sophisticated variants like Variance Clipping and AABB Clipping that better handle color information and statistical distributions [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** The reprojected history buffer value $H$ for the current pixel, and a neighborhood of current-frame color samples $C_{neighbor}$ (typically a 3x3 grid of pixels surrounding the current pixel).
*   **Output:** A rectified history value $H_{rect}$ ready for blending with the current frame.

**Data Structures:**
*   **Neighborhood Samples:** A set of color vectors $\{c_1, c_2, ..., c_n\}$ representing the RGB(A) values of the neighboring pixels in the current frame.
*   **Bounds:** A minimum vector $min$ and a maximum vector $max$ defining the rectification region. For AABB-based methods, these are component-wise min/max. For Variance Clipping, these are derived from statistical moments.

**Steps:**
1.  **Sample Neighborhood:** Gather the color values of the neighboring pixels in the current frame. The standard evaluation uses a 3x3 neighborhood to avoid averaging across depth slices, which would act as a spatial stabilizer and hide flicker artifacts [S1, unofficial]. Alternatives like 3x3x3 or 6-tap neighborhoods smooth across depth but are not used in the primary evaluation context [S1, unofficial].
2.  **Define Bounds:** Calculate the bounding region based on the chosen variant (see Subtopics).
    *   For AABB variants: $min = \min(c_i)$, $max = \max(c_i)$ for each color channel independently.
    *   For Variance Clipping: Bounds are expanded beyond min/max using the standard deviation of the neighborhood values to account for statistical distribution [S1, unofficial].
3.  **Rectify History:** Compare the reprojected history value $H$ against the defined bounds.
    *   If $H$ is within the bounds, $H_{rect} = H$.
    *   If $H$ is outside the bounds, $H_{rect}$ is adjusted to the nearest boundary value (clamping) or projected onto the boundary (clipping) [S1, unofficial].
4.  **Blend:** Blend $H_{rect}$ with the current frame color $C_{current}$ using the temporal blend factor $\alpha$ (typically $\alpha = 0.05$ for evaluation comparability) [S1, unofficial].

**Formulas:**
For AABB Clamping/Clipping, the rectified value for a single channel is:
$$ H_{rect} = \text{clamp}(H, min, max) $$
Where $\text{clamp}(v, a, b)$ returns $a$ if $v < a$, $b$ if $v > b$, and $v$ otherwise.

For Variance Clipping, the bounds are extended by a factor of the standard deviation $\sigma$:
$$ min_{var} = \mu - k \cdot \sigma $$
$$ max_{var} = \mu + k \cdot \sigma $$
Where $\mu$ is the mean of the neighborhood, $\sigma$ is the standard deviation, and $k$ is a scaling factor (value not explicitly given in sources, but implied to be part of the method definition) [S1, unofficial].

**Parameters:**
*   **Neighborhood Size:** 3x3 is the standard for evaluation to preserve depth separation [S1, unofficial].
*   **Blend Factor ($\alpha$):** 0.05 is used in the cited evaluations to match baseline EMA conditions [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Static Extent:** When the light turns off, the neighborhood extent becomes 0 (a single point). The history is clamped to this point, eliminating temporal lag instantly [S1, unofficial].
*   **Gradual Color Change:** AABB clamping struggles with gradual color transitions (e.g., red to blue fade). It exhibits a large RMSE spike when the color change completes and takes ~50 frames to converge, as clamping is less effective than clipping for introducing new color information [S1, unofficial].
*   **Temporal Instability (TI):** Tighter bounds allow faster adaptation but increase TI, perceived as visual flicker. The distance between highest error peaks is exactly 16 frames, corresponding to the jitter sequence length [S1, unofficial].
*   **Grayscale Scenes:** AABB clipping and clamping produce identical results in grayscale scenes because color rectification is irrelevant [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def history_rectification(history_value, current_frame_color, neighborhood_colors, method='aabb_clamp'):
    """
    Rectifies the history value based on the current frame neighborhood.
    
    Args:
        history_value: float or vec3, the reprojected history color.
        current_frame_color: float or vec3, the current frame's color.
        neighborhood_colors: list of float or vec3, colors of neighboring pixels.
        method: str, one of 'aabb_clamp', 'aabb_clip', 'variance_clip'.
        
    Returns:
        float or vec3, the rectified history value.
    """
    # Step 1: Calculate bounds from neighborhood
    if method == 'aabb_clamp' or method == 'aabb_clip':
        # Component-wise min and max
        min_val = min(neighborhood_colors, key=lambda c: c) # Assuming vector comparison or per-channel
        max_val = max(neighborhood_colors, key=lambda c: c)
        
        # For vector colors, min/max is per-channel
        min_val = vec3(min(c.x for c in neighborhood_colors), 
                       min(c.y for c in neighborhood_colors), 
                       min(c.z for c in neighborhood_colors))
        max_val = vec3(max(c.x for c in neighborhood_colors), 
                       max(c.y for c in neighborhood_colors), 
                       max(c.z for c in neighborhood_colors))
        
        if method == 'aabb_clamp':
            # Clamp history to AABB
            rectified = clamp(history_value, min_val, max_val)
        else: # aabb_clip
            # Project history onto AABB (mathematically similar to clamp for axis-aligned boxes)
            # Note: Sources state AABB Clamp and Clip produce identical results in grayscale,
            # and generally imply similar bounding behavior for rectification.
            rectified = clamp(history_value, min_val, max_val)
            
    elif method == 'variance_clip':
        # Calculate mean and standard deviation
        n = len(neighborhood_colors)
        mean_val = sum(neighborhood_colors) / n
        
        # Calculate variance
        variance = sum((c - mean_val)**2 for c in neighborhood_colors) / n
        std_dev = sqrt(variance)
        
        # Define bounds using variance (k is a hyperparameter, not specified in sources)
        # [G] Assuming a standard deviation multiplier k=1.0 for implementation completeness
        k = 1.0 
        min_val = mean_val - k * std_dev
        max_val = mean_val + k * std_dev
        
        rectified = clamp(history_value, min_val, max_val)
        
    else:
        rectified = history_value # Fallback to no rectification
        
    return rectified

def clamp(value, min_val, max_val):
    """Clamps value between min_val and max_val."""
    if value < min_val:
        return min_val
    elif value > max_val:
        return max_val
    else:
        return value
```

## Subtopics

### Variance Clipping History Rectification
A history rectification method that uses the statistical distribution of values in the current frame neighborhood to create the bounds of the AABB used for clipping [S1, unofficial]. It calculates the mean and standard deviation of the neighborhood colors to define a range that is typically tighter than the raw min/max but accounts for variance. It consistently introduces less bias than AABB-based methods during movement, reducing ghosting artifacts more effectively, but exhibits higher Temporal Instability (flicker) [S1, unofficial]. In step response scenarios, it achieves the lowest RMSE (0.002977) but the highest TI P (0.004217, +190.23% increase over baseline) [S1, unofficial].

### AABB Clipping History Rectification
A history rectification method that projects the history onto the AABB defined by the minimum and maximum values in each color channel of the current frame neighborhood [S1, unofficial]. In grayscale scenes, it produces identical results to AABB Clamping because color rectification is not relevant [S1, unofficial]. It performs better than EMA in abrupt change scenarios and has lower TI than Variance Clipping (TI P of 0.002373, +63.32% over baseline) [S1, unofficial]. However, it struggles with gradual color changes, showing a large RMSE spike when a light fades from red to blue [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
*   **Time:** $O(N)$ where $N$ is the number of neighbors in the sampling kernel (e.g., 9 for 3x3). The operations are simple min/max or mean/std calculations per pixel.
*   **Memory:** $O(1)$ auxiliary memory per pixel for storing neighborhood samples during the rectification pass.

**Measured Performance:**
*   **Hardware/Resolution:** Not given in the sources.
*   **Scene:** Evaluated in step response (abrupt light on/off) and gradual change (moving/rotating light) scenarios [S1, unofficial].
*   **Metrics:**
    *   **Baseline EMA:** RMSE 0.009724, TI P 0.001453 [S1, unofficial].
    *   **AABB Clamp/Clip:** RMSE 0.003136, TI P 0.002373 (+63.32% TI) [S1, unofficial].
    *   **Variance Clip:** RMSE 0.002977, TI P 0.004217 (+190.23% TI) [S1, unofficial].

**Scaling:**
*   Performance scales with neighborhood size. Larger neighborhoods (3x3x3, 6-tap) smooth the signal across depth, reducing visible flicker but potentially blurring details [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Superior to EMA in abrupt change scenarios; immediately drops error to zero when intensity drops, eliminating temporal lag [S1, unofficial]. Variance clipping produces the lowest error spike when reconstructing the signal during quick illumination introduction [S1, unofficial].
*   **Limitations:** Higher Temporal Instability (flicker) compared to EMA, especially with Variance Clipping [S1, unofficial]. AABB Clamping struggles with gradual color changes, taking ~50 frames to converge after a color fade [S1, unofficial]. Jittering causes periodic error patterns (16-frame cycle) perceived as flicker [S1, unofficial].

**When to use/not use:**
*   **Use:** When rapid reaction to scene changes (light on/off, fast motion) is prioritized over absolute temporal stability, and ghosting must be minimized.
*   **Not Use:** In scenes with gradual color transitions where AABB clamping is used, or when visual flicker is unacceptable and higher TI is a dealbreaker.

## Background
History rectification techniques were developed as part of the broader effort to improve Temporal Anti-Aliasing (TAA) for real-time rendering, specifically in the context of froxel-based volumetric rendering. The specific evaluation of these methods (Variance Clipping, AABB Clamping, AABB Clipping) is detailed in the work on "Advances in volumetric temporal anti-aliasing for real-time rendering of participating media" [S1, unofficial]. The techniques build upon earlier pixel-space TAA counterparts, adapting concepts like convex hull clipping (which is too expensive) and variance-based bounds to the volumetric domain [S1, unofficial].

## Sources
[S1] Temporal Anti-Aliasing for Real-Time Froxel-Based Volumetric Rendering - Advances in volumetric temporal anti-aliasing for real-time rendering of participating media, https://openalex.org/W7218067689, fetched 2026-11-01 (unofficial)
