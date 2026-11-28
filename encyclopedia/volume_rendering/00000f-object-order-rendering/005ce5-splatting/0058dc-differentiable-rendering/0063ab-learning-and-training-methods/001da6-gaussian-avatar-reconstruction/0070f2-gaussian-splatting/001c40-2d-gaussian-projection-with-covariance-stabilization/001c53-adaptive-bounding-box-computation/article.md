# Adaptive Bounding Box Computation

Also called: Tsplat bounding box

## Overview
Computes a tight, axis-aligned bounding box (AABB) for a 2D Gaussian splat based on a specific opacity threshold, replacing the loose circular bounding box used in original 3D Gaussian Splatting (3DGS) [S1, unofficial][S2]. This method significantly reduces false-positive tile intersections and the resulting number of duplicated splat fragments during rasterization [S1, unofficial][S2].

## How it works
**Inputs:**
- `sx`, `sy`: Screen-space center coordinates of the 2D Gaussian splat (float) [S1, unofficial].
- `cov2d_00`, `cov2d_11`: Diagonal elements of the projected 2D covariance matrix $\Sigma$ (float) [S1, unofficial].
- `opacity`: The scalar opacity of the splat (float) [S1, unofficial].
- `alpha_threshold`: The minimum opacity threshold $\tau$ for a pixel to be considered visible (float) [S1, unofficial].
- `max_k2`: The maximum allowed squared Mahalanobis distance to prevent overflow or excessive box sizes (float) [S1, unofficial].
- `w_i`, `h_i`: Framebuffer width and height in pixels (uint) [S1, unofficial].

**Outputs:**
- A bounding box defined by integer coordinates $(x_0, y_0, x_1, y_1)$, or `None` if the splat is effectively invisible or has negligible size [S1, unofficial].

**Steps:**
1. **Opacity Check:** If `opacity` $\le$ `alpha_threshold`, return `None` [S1, unofficial].
2. **Compute Squared Mahalanobis Distance Limit ($k^2$):**
   Calculate the squared distance in the normalized space corresponding to the opacity threshold:
   $$ k^2 = \min\left(2.0 \cdot \ln\left(\frac{\text{opacity}}{\text{alpha\_threshold}}\right), \text{max\_k2}\right) $$
   If $k^2 \le 0.0$, return `None` [S1, unofficial].
3. **Compute Half-Widths:**
   Calculate the half-extents of the bounding box along the x and y axes:
   $$ rx_f = \sqrt{k^2 \cdot \text{cov2d}_{00}} $$
   $$ ry_f = \sqrt{k^2 \cdot \text{cov2d}_{11}} $$
   If $rx_f$ or $ry_f$ is not finite or is less than $1.0$, return `None` [S1, unofficial].
4. **Compute Integer Bounding Box:**
   Determine the pixel coordinates of the bounding box corners:
   $$ x_0 = \lfloor sx - rx_f \rfloor $$
   $$ y_0 = \lfloor sy - ry_f \rfloor $$
   $$ x_1 = \lceil sx + rx_f \rceil $$
   $$ y_1 = \lceil sy + ry_f \rceil $$
5. **Clip to Framebuffer:**
   Clamp the coordinates to the valid framebuffer range:
   $$ x_0 = \max(0, x_0) $$
   $$ y_0 = \max(0, y_0) $$
   $$ x_1 = \min(w_i - 1, x_1) $$
   $$ y_1 = \min(h_i - 1, y_1) $$
   If $x_0 > x_1$ or $y_0 > y_1$, return `None` [S1, unofficial].
6. **Return:**
   Return the tuple $(x_0, y_0, x_1, y_1)$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_adaptive_bounding_box(sx, sy, cov2d_00, cov2d_11, opacity, alpha_threshold, max_k2, w_i, h_i):
    # Step 1: Opacity Check
    if opacity <= alpha_threshold:
        return None

    # Step 2: Compute k^2
    k2 = 2.0 * math.log(opacity / alpha_threshold)
    if k2 > max_k2:
        k2 = max_k2
    if k2 <= 0.0:
        return None

    # Step 3: Compute half-widths
    rx_f = math.sqrt(k2 * cov2d_00)
    ry_f = math.sqrt(k2 * cov2d_11)

    if not math.isfinite(rx_f) or rx_f < 1.0:
        return None
    if not math.isfinite(ry_f) or ry_f < 1.0:
        return None

    # Step 4: Compute integer bounding box
    x0 = math.floor(sx - rx_f)
    y0 = math.floor(sy - ry_f)
    x1 = math.ceil(sx + rx_f)
    y1 = math.ceil(sy + ry_f)

    # Step 5: Clip to framebuffer
    x0 = max(0, x0)
    y0 = max(0, y0)
    x1 = min(w_i - 1, x1)
    y1 = min(h_i - 1, y1)

    # Step 6: Validity Check
    if x0 > x1 or y0 > y1:
        return None

    # Step 7: Return
    return (x0, y0, x1, y1)
```

**Parameters:**
- `max_k2`: Not given in the sources [S1, unofficial].
- `alpha_threshold`: Not given in the sources [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Precision:** The calculation of $k^2$ involves logarithms and square roots. The check $k^2 \le 0.0$ handles cases where numerical noise might otherwise produce negative values under the square root [S1, unofficial].
- **Boundaries:** The clipping step ensures the bounding box stays within the framebuffer. If the splat is entirely outside the framebuffer, the clipping will result in an invalid box ($x_0 > x_1$), which is correctly handled [S1, unofficial].
- **Degenerate Input:** If `cov2d_00` or `cov2d_11` is zero or negative, the half-widths will be zero or NaN. The check for $< 1.0$ and `isfinite` handles this [S1, unofficial].

## Subtopics

### [Opacity-Sensitive Elliptical Bounding Box](2100-opacity-sensitive-elliptical-bounding-box.md)
Determines which image tiles a 2D Gaussian splat intersects by computing a tight, non-square AABB around the ellipse defined by the splat's covariance at a specific opacity threshold. It reduces false-positive tile intersections compared to the original 3DGS circular bounding box. Choose it when precise tile culling is required to minimize duplicated splat fragments.

### [Tight Ellipse AABB Derivation](2826-tight-ellipse-aabb-derivation.md)
Computes the tight AABB of a projected 2D Gaussian ellipse in constant time to determine the rectangular tile extent for rasterization. It replaces the loose circular bounding box of standard 3DGS, significantly reducing false-positive tile intersections. Choose it for initial culling and spatial indexing where iterative exact intersection methods are too slow.

### [Opacity-aware Non-Square Bounding Box](6079-opacity-aware-non-square-bounding-box.md)
Computes a tight, axis-aligned rectangular bounding box for a 2D Gaussian splat based on its projected opacity and covariance, replacing the original 3DGS square bounding box derived from a fixed confidence radius. It reduces false positives in per-tile splat lists by accounting for the Gaussian's actual opacity and non-square shape. Choose it when splat aspect ratios vary significantly across views.

### Non-Square Bounding Boxes
Allows the axis-aligned bounding box of a projected 2D Gaussian to have different width and height dimensions, rather than forcing a square shape [S2]. On the Bicycle scene at native resolution with 16x16 tiles, non-square boxes reduced duplicated Gaussians from 55,648,264 (square) to 18,466,556 [S2].

## Operating characteristics
**Time Complexity:** $O(1)$ per splat. The computation involves a constant number of arithmetic operations, logarithms, and square roots [S1, unofficial].
**Memory Complexity:** $O(1)$ per splat. No additional memory is allocated for the bounding box computation itself [S1, unofficial].
**Measured Performance:** On the Bicycle scene at native resolution, the combination of opacity-aware bounding boxes and other improvements decreased render time by a factor of 3.77 and GPU memory usage by a factor of 2.11 [S2]. The reduction in duplicated Gaussians from 55,648,264 (square) to 18,466,556 (non-square) on the same scene highlights the efficiency gain in rasterization [S2].
**Strengths:** Significantly reduces false-positive tile intersections, leading to fewer splat duplications and lower memory bandwidth usage during rasterization [S1, unofficial][S2]. It is computationally inexpensive.
**Limitations:** Requires the covariance matrix to be diagonal or pre-rotated to the screen space axes. If the covariance is not diagonal, the computed box may be loose or incorrect.
**When to use:** Use when efficient culling of Gaussian splats is required, especially in scenes with high splat density or varying opacity. It is particularly effective when combined with opacity thresholds [S1, unofficial][S2].
**When not to use:** Do not use if the covariance matrix is not available or if the splat's shape is not well-approximated by an ellipse in screen space.

## Background
The method addresses the inefficiencies of the original 3D Gaussian Splatting paper's circular bounding box approach [S1, unofficial][S2]. It is implemented in the Tsplat project and discussed in the context of fast 3D Gaussian Splatting rendering improvements [S1, unofficial][S2]. The original 3D Gaussian Splatting paper by Kerbl et al. is acknowledged as the foundation [S1, unofficial].

## Sources
[S1] Tsplat – Run Gaussian splatting in your terminal, https://github.com/darshanmakwana412/tsplat, fetched 2026-10-26 (unofficial)
[S2] Fast 3D Gaussian Splatting Rendering via Easily Integrable Improvements, https://doi.org/10.1109/lsp.2024.3521379, fetched 2026-10-26
