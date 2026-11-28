# Textural Complexity Objective


## Overview
The Textural Complexity Objective ($T$) is a metric used in active view planning for radiance-field optimization to identify camera poses that capture regions with high photometric variation [S1, unofficial]. Unlike geometric metrics that analyze 3D mesh structure, $T$ operates on 2D image projections, leveraging the fact that textural characteristics are readily discernible in 2D even if 3D geometry is ambiguous [S1, unofficial]. It serves as a component of Color-Corrected Error Metrics, helping to mitigate photometric variations such as autoexposure shifts or white balance changes in "in the wild" datasets by prioritizing views with rich texture [S1, unofficial].

## How it works
The method computes a complexity score for an image captured at a specific camera pose [S1, unofficial]. The core idea is that images with higher textural complexity (more unique local patterns) provide more information for reconstruction [S1, unofficial]. The final score is derived from the distribution of Local Binary Patterns (LBP) in the Hue and Saturation channels of the image [S1, unofficial].

**Inputs:**
- $m_k$: The rendered or captured image at camera pose $c_k$ [S1, unofficial].
- $r_T$: Filter radius for LBP (default: 3) [S1, unofficial].
- $p_T$: Number of circularly symmetric local neighborhood points for LBP (default: 3) [S1, unofficial].
- $d_T$: Discount factor to prevent overfitting to previously seen textures [S1, unofficial].

**Outputs:**
- $f_T$: The normalized textural complexity score for the image [S1, unofficial].

**Data Structures:**
- Histogram array: A 1D array representing the frequency of LBP codes [S1, unofficial]. The size depends on the number of possible LBP codes for the given $p_T$ [G].

**Steps:**
1. **Channel Selection:** Extract the Hue ($H$) and Saturation ($S$) channels from the image $m_k$ [S1, unofficial].
2. **LBP Computation:** For each channel separately, compute the Local Binary Pattern descriptor [S1, unofficial].
   - For each pixel, compare its intensity to its $p_T$ neighbors arranged in a circle of radius $r_T$ [G].
   - Generate a binary code based on whether neighbors are greater than or equal to the center pixel [G].
   - Aggregate these codes into a histogram $Hist(\cdot)$ [S1, unofficial].
3. **Histogram Normalization:** Let $f'_T$ be the histogram of LBP codes [S1, unofficial]. The source defines $f'_T = Hist(LBP(m_k; r_T, p_T))$ [S1, unofficial].
4. **Complexity Calculation:** Compute the complexity score for the channel using the last bin of the histogram [S1, unofficial].
   - Let $b$ be the index of the last bin in the histogram [S1, unofficial].
   - Calculate $f_T = 1 - f'_T[b]$ [S1, unofficial].
   - Maximizing this equation selects images with the highest textural complexity [S1, unofficial].
5. **Channel Averaging:** Compute the final score by averaging the $f_T$ values obtained from the Hue and Saturation channels [S1, unofficial].
6. **Discounting:** Apply a discount factor $d_T$ to the $T$ values of views that overlap with previously planned trajectories to prevent the planner from overfitting to specific textures already seen [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def rgb_to_hsv(r, g, b):
    """Converts a single RGB pixel to HSV."""
    max_c = max(r, g, b)
    min_c = min(r, g, b)
    diff = max_c - min_c
    if max_c == 0:
        h, s = 0.0, 0.0
    else:
        s = diff / max_c
        if max_c == r:
            h = 60.0 * (((g - b) / diff) % 6)
        elif max_c == g:
            h = 60.0 * (((b - r) / diff) + 2)
        else:
            h = 60.0 * (((r - g) / diff) + 4)
    return h, s, max_c

def interpolate(image, x, y):
    """Bilinear interpolation for floating point coordinates."""
    x0 = int(math.floor(x))
    y0 = int(math.floor(y))
    x1 = x0 + 1
    y1 = y0 + 1
    height = len(image)
    width = len(image[0])
    # Clamp coordinates to image bounds
    x0 = max(0, min(x0, width - 1))
    y0 = max(0, min(y0, height - 1))
    x1 = max(0, min(x1, width - 1))
    y1 = max(0, min(y1, height - 1))
    val00 = image[y0][x0]
    val01 = image[y0][x1]
    val10 = image[y1][x0]
    val11 = image[y1][x1]
    dx = x - x0
    dy = y - y0
    return (1 - dx) * (1 - dy) * val00 + dx * (1 - dy) * val01 + (1 - dx) * dy * val10 + dx * dy * val11

def compute_textural_complexity(image, r_T=3, p_T=3, d_T=0.0):
    """
    Computes the Textural Complexity Objective for an image.
    Args:
        image: 2D list of RGB tuples or 3D array-like (H, W, 3).
        r_T: LBP filter radius.
        p_T: Number of LBP neighborhood points.
        d_T: Discount factor for previously seen views.
    Returns:
        f_T: Normalized textural complexity score.
    """
    # Step 1: Extract Hue and Saturation channels
    height = len(image)
    width = len(image[0])
    hue_channel = [[0.0] * width for _ in range(height)]
    sat_channel = [[0.0] * width for _ in range(height)]
    for y in range(height):
        for x in range(width):
            r, g, b = image[y][x]
            h, s, _ = rgb_to_hsv(r, g, b)
            hue_channel[y][x] = h
            sat_channel[y][x] = s

    # Step 2 & 3: Compute LBP Histogram for each channel
    num_bins = 2 ** p_T
    hist_hue = [0.0] * num_bins
    hist_sat = [0.0] * num_bins

    for channel, hist in [(hue_channel, hist_hue), (sat_channel, hist_sat)]:
        for y in range(1, height - 1):
            for x in range(1, width - 1):
                center_val = channel[y][x]
                code = 0
                for i in range(p_T):
                    angle = 2 * math.pi * i / p_T
                    nx = x + r_T * math.cos(angle)
                    ny = y - r_T * math.sin(angle)
                    neighbor_val = interpolate(channel, nx, ny)
                    if neighbor_val >= center_val:
                        code |= (1 << i)
                hist[code] += 1

    # Normalize histograms to sum to 1 (standard practice for probability distributions) [G]
    sum_hue = sum(hist_hue)
    sum_sat = sum(hist_sat)
    if sum_hue > 0: hist_hue = [h / sum_hue for h in hist_hue]
    if sum_sat > 0: hist_sat = [h / sum_sat for h in hist_sat]

    # Step 4: Calculate complexity for each channel
    b = num_bins - 1
    f_T_hue = 1.0 - hist_hue[b]
    f_T_sat = 1.0 - hist_sat[b]

    # Step 5: Average the scores
    f_T_raw = (f_T_hue + f_T_sat) / 2.0

    # Step 6: Apply discount factor if needed
    # The source mentions applying d_T towards T values seen during planning.
    # This is typically done at the trajectory planning level, not per-image.
    # Here we return the raw score.
    return f_T_raw
```

**Parameters:**
- $r_T = 3$: Filter radius for LBP [S1, unofficial].
- $p_T = 3$: Number of circularly symmetric local neighborhood points [S1, unofficial].

**Pitfalls:**
- **Overfitting:** The planner may overfit to specific textures if it repeatedly selects views of the same complex region [S1, unofficial]. This is mitigated by the discount factor $d_T$ [S1, unofficial].

## Subtopics

### Geometric Complexity Objective
A surrogate objective corresponding to the variability of mesh faces' configurations, quantified via mesh normals to identify regions requiring varied views [S1, unofficial]. It is computed using mesh normals $\hat{n}_j$ [S1, unofficial]. The objective function $Q$ quantifies the variability of mesh faces' configurations to identify geometrically complex regions (e.g., pillars, domes) that require varied views [S1, unofficial].

**Formula:**
The per-face geometric complexity is defined as:
$$ f'_Q = |LoG(\hat{n}_j; \sigma_Q)|^{\beta_Q} V_{jk} + 1 $$
where $LoG(\cdot)$ is the Laplacian of Gaussian filter, $\sigma_Q$ is the standard deviation of the Gaussian filter, $\beta_Q$ is a scaling factor, $f'_Q$ is the per-face geometric complexity, and $V_{jk}$ is an indicator function corresponding to the visibility of face $j$ from the $k$-th camera-pose [S1, unofficial]. The absolute value is taken because the directionality of normal variations does not matter [S1, unofficial].

The normalized total complexity is:
$$ f_Q = \frac{1}{J} \sum_{j=1}^{J} f'_Q $$
where $J$ is the total number of faces [S1, unofficial]. Maximization of $f_Q$ encourages the selection of camera-poses with higher geometric complexity [S1, unofficial].

**Pitfalls:**
- **Local Maxima:** Over-emphasis on local regions with unusually high geometric complexity can trap the planner in a local maxima [S1, unofficial]. This is handled by applying a scaling discount factor $d_Q$ towards the $Q$ values of overlapping faces seen during trajectory planning [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Strengths:** Effective at identifying regions with rich texture that are crucial for high-fidelity reconstruction [S1, unofficial]. Robust to some lighting variations by using Hue/Saturation [S1, unofficial].
- **Limitations:** Computationally more expensive than simple photometric error metrics [G]. Relies on accurate camera poses and rendered images [S1, unofficial].
- **When to use:** When optimizing for "in the wild" datasets with varying lighting and texture, such as Tanks and Temples [S1, unofficial].
- **When not to use:** Not given in the sources.

## Background
The Textural Complexity Objective was introduced in the context of active view planning for NeRF (SO-NeRF) [S1, unofficial]. It is part of a family of surrogate objectives (including Geometric Complexity and Ray Diversity) designed to guide camera trajectory planning for better 3D reconstruction [S1, unofficial]. The method builds on the observation that textural characteristics are discernible in 2D projections, unlike some geometric features [S1, unofficial].

## Sources
[S1] SO-NeRF: Active View Planning for NeRF using Surrogate Objectives, https://doi.org/10.48550/arXiv.2312.03266, fetched 2026-10-27 (unofficial)
