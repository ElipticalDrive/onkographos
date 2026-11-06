# YCoCg Color Space Rectification

Also called: Color Space Selection for History Rectification, YCrCb Rectification

## Overview
YCoCg Color Space Rectification is a history rectification technique used in Temporal Anti-Aliasing (TAA) for volumetric rendering that transforms colors into a luminance-chrominance color space (specifically YCoCg or YCrCb) before applying clipping or clamping operations [S1, unofficial]. It solves the problem of color-related ghosting artifacts and temporal color bleeding that occur when history rectification is performed directly in RGB space, where channels are highly correlated and bounding boxes are poorly aligned with the actual distribution of color samples [S1, unofficial]. What sets it apart from its siblings and nearest alternatives, such as [AABB Clamping](2019-aabb-clamping.md) and [Variance Clipping](1970-variance-clipping.md), is the explicit decoupling of luminance and chrominance to orient the rectification bounds along the luma direction, which typically has high local contrast while chroma does not [S1, unofficial].

## How it works
**Inputs and outputs:**
- Input: Current frame color sample $C_{curr} = [R, G, B]^T$ and historical color sample $C_{hist} = [R, G, B]^T$ (or their transformed equivalents) [S1, unofficial].
- Output: Rectified color sample $C_{out} = [R, G, B]^T$ in RGB space [S1, unofficial].

**Data structures:**
- Color vectors are 3-element arrays or tuples [G].
- Transform matrices are $3 \times 3$ matrices [G].
- Rectification bounds (min/max) are computed per-channel in the target color space [S1, unofficial].

**Steps:**
1. **Transform to Color Space:** Convert the current frame color $C_{curr}$ and the reprojected historical color $C_{hist}$ from RGB to the chosen luminance-chrominance space (YCoCg or YCrCb) [S1, unofficial].
2. **Compute Bounds:** Determine the minimum and maximum values for each component (Y, Co/Cr, Cg/Cb) within the local neighborhood or based on the current frame's extent [S1, unofficial].
3. **Rectify History:** Clamp or clip the historical color components $C_{hist}'$ to the computed bounds $[min_i, max_i]$ for each component $i \in \{Y, Co/Cr, Cg/Cb\}$ [S1, unofficial].
4. **Transform Back:** Convert the rectified color $C_{rect}'$ from the luminance-chrominance space back to RGB space [S1, unofficial].
5. **Blend:** Use the rectified RGB color in the temporal blending process (e.g., with the current frame) [S1, unofficial].

**Formulas:**

*RGB to YCoCg Transform:*
$$
\begin{bmatrix} Y \\ Co \\ Cg \end{bmatrix} = \begin{bmatrix} \frac{1}{4} & \frac{1}{2} & \frac{1}{4} \\ \frac{1}{2} & 0 & -\frac{1}{2} \\ -\frac{1}{4} & \frac{1}{2} & -\frac{1}{4} \end{bmatrix} \begin{bmatrix} R \\ G \\ B \end{bmatrix}
$$

*Inverse YCoCg to RGB Transform:*
$$
\begin{bmatrix} R \\ G \\ B \end{bmatrix} = \begin{bmatrix} 1 & 1 & -1 \\ 1 & 0 & 1 \\ 1 & -1 & -1 \end{bmatrix} \begin{bmatrix} Y \\ Co \\ Cg \end{bmatrix}
$$

*RGB to YCrCb Transform:*
$$
\begin{bmatrix} Y \\ Cr \\ Cb \end{bmatrix} = \begin{bmatrix} 0.299 & 0.587 & 0.114 \\ -0.5 & -0.4187 & 0.0813 \\ -0.1687 & -0.3313 & -0.5 \end{bmatrix} \begin{bmatrix} R \\ G \\ B \end{bmatrix}
$$

*Inverse YCrCb to RGB Transform:*
$$
\begin{bmatrix} R \\ G \\ B \end{bmatrix} = \begin{bmatrix} 1 & 1.402 & 0 \\ 1 & -0.714 & -0.344 \\ 1 & 0 & 1.772 \end{bmatrix} \begin{bmatrix} Y \\ Cr \\ Cb \end{bmatrix}
$$

**Parameters:**
- Color space selection: YCoCg or YCrCb. YCoCg is preferred for better decorrelation of luminance and chrominance [S1, unofficial].

**Edge cases and pitfalls:**
- **Luminance Artifacts:** While YCoCg and YCrCb reduce chrominance color bleeding, they can increase luminance artifacts. For example, when color changes from green to blue, stale luminance information in the green channel may bleed into the next frame, visible as excess luminance around the light source [S1, unofficial].
- **Precision:** Standard floating-point precision is sufficient for the matrix multiplications [G].
- **Bound Computation:** The bounds must be computed in the transformed space, not RGB space, to ensure the AABB is aligned with the luma direction [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def rgb_to_ycocg(r, g, b):
    y = 0.25 * r + 0.5 * g + 0.25 * b
    co = 0.5 * r - 0.5 * b
    cg = -0.25 * r + 0.5 * g - 0.25 * b
    return y, co, cg

def ycocg_to_rgb(y, co, cg):
    r = y + co - cg
    g = y + cg
    b = y - co - cg
    return r, g, b

def rgb_to_ycrcb(r, g, b):
    y = 0.299 * r + 0.587 * g + 0.114 * b
    cr = -0.5 * r - 0.4187 * g + 0.0813 * b
    cb = -0.1687 * r - 0.3313 * g - 0.5 * b
    return y, cr, cb

def ycrcb_to_rgb(y, cr, cb):
    r = y + 1.402 * cr
    g = y - 0.714 * cr - 0.344 * cb
    b = y + 1.772 * cb
    return r, g, b

def rectify_color_in_ycocg(current_color, historical_color, bounds_min, bounds_max):
    """
    Rectifies historical color using YCoCg space.
    current_color: tuple (R, G, B)
    historical_color: tuple (R, G, B)
    bounds_min: tuple (min_y, min_co, min_cg)
    bounds_max: tuple (max_y, max_co, max_cg)
    """
    # Transform to YCoCg
    y_curr, co_curr, cg_curr = rgb_to_ycocg(*current_color)
    y_hist, co_hist, cg_hist = rgb_to_ycocg(*historical_color)
    
    # Clamp historical color components to bounds
    y_hist_rect = max(bounds_min[0], min(bounds_max[0], y_hist))
    co_hist_rect = max(bounds_min[1], min(bounds_max[1], co_hist))
    cg_hist_rect = max(bounds_min[2], min(bounds_max[2], cg_hist))
    
    # Transform back to RGB
    r_out, g_out, b_out = ycocg_to_rgb(y_hist_rect, co_hist_rect, cg_hist_rect)
    
    return (r_out, g_out, b_out)

def rectify_color_in_ycrcb(current_color, historical_color, bounds_min, bounds_max):
    """
    Rectifies historical color using YCrCb space.
    current_color: tuple (R, G, B)
    historical_color: tuple (R, G, B)
    bounds_min: tuple (min_y, min_cr, min_cb)
    bounds_max: tuple (max_y, max_cr, max_cb)
    """
    # Transform to YCrCb
    y_curr, cr_curr, cb_curr = rgb_to_ycrcb(*current_color)
    y_hist, cr_hist, cb_hist = rgb_to_ycrcb(*historical_color)
    
    # Clamp historical color components to bounds
    y_hist_rect = max(bounds_min[0], min(bounds_max[0], y_hist))
    cr_hist_rect = max(bounds_min[1], min(bounds_max[1], cr_hist))
    cb_hist_rect = max(bounds_min[2], min(bounds_max[2], cb_hist))
    
    # Transform back to RGB
    r_out, g_out, b_out = ycrcb_to_rgb(y_hist_rect, cr_hist_rect, cb_hist_rect)
    
    return (r_out, g_out, b_out)
```

## Subtopics

### [AABB Clamping](2019-aabb-clamping.md)
AABB Clamping is a history rectification technique used in Temporal Anti-Aliasing (TAA) to prevent ghosting artifacts by restricting reprojected historical color values to a valid range defined by the current frame. It solves the problem of temporal lag and ghosting during abrupt scene changes where a baseline Exponential Moving Average (EMA) fails to react quickly enough. What sets it apart from its siblings and nearest alternatives is its use of a simple Axis-Aligned Bounding Box (AABB) approximation constructed from per-channel minimum and maximum values, whereas History Rectification encompasses more sophisticated variants like Variance Clipping that handle color distributions more accurately. It builds on [YCoCg Color Space Rectification](1971-ycocg-color-space-rectification.md), which transforms colors into a luminance-chrominance space before applying this rectification to reduce color-related ghosting.

### Color Space Conversion for TAA
Transformation of volumetric samples into a specific color space before blending, then transforming back to RGB for storage. Evaluated spaces include RGB, YCoCg, and YCrCb [S1, unofficial].

## Operating characteristics
- **Time Complexity:** The transformation and inverse transformation involve a constant number of arithmetic operations (matrix-vector multiplications), so the time complexity per sample is $O(1)$ [G].
- **Memory Complexity:** No additional memory is required beyond the storage for the color values and bounds [G].
- **Measured Performance:** YCoCg and YCrCb reduce chrominance color bleeding artifacts compared to RGB space [S1, unofficial]. However, this comes at the cost of increased luminance artifacts in certain scenarios, such as when color changes from green to blue [S1, unofficial].
- **Strengths:** Reduces color-related ghosting and temporal color bleeding by decoupling luminance and chrominance. YCoCg provides better decorrelation than YCrCb [S1, unofficial].
- **Limitations:** Can introduce luminance artifacts when chrominance changes rapidly. Requires additional matrix multiplications [S1, unofficial].
- **When to use:** Not given in the sources.
- **When not to use:** Not given in the sources.

## Background
YCoCg Color Space Rectification originates from research into temporal anti-aliasing for real-time froxel-based volumetric rendering [S1, unofficial]. It was explored as a method to improve the quality of TAA by addressing the limitations of RGB-space rectification [S1, unofficial]. The technique leverages the properties of luminance-chrominance color spaces to better align the rectification bounds with the natural distribution of color data in images [S1, unofficial].

## Sources
[S1] Temporal Anti-Aliasing for Real-Time Froxel-Based Volumetric Rendering - Advances in volumetric temporal anti-aliasing for real-time rendering of participating media, https://openalex.org/W7218067689, fetched 2026-11-01 (unofficial)
