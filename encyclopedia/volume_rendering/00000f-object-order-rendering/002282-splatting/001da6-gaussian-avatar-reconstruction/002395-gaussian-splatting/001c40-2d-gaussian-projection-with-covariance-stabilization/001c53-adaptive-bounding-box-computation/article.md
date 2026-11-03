# Adaptive Bounding Box Computation

Also called: Tsplat bounding box

## Overview
Adaptive Bounding Box Computation computes a tight, axis-aligned bounding box (AABB) for a projected 2D Gaussian splat based on an opacity threshold, replacing the wasteful circular bounding boxes of the original 3D Gaussian Splatting method [S1]. It solves the problem of excessive tile intersections and duplicated Gaussian processing by filtering out Gaussians whose opacity falls below a threshold and by deriving non-square bounding boxes from the projected 2D covariance matrix [S1]. Unlike its sibling [2D Gaussian Projection with Covariance Stabilization](1198-2d-gaussian-projection-with-covariance-stabilization.md), which ensures numerical stability of the covariance matrix, this method uses the opacity and the projected covariance to derive an adaptive bounding box that significantly reduces false-positive tile intersections [S1].

## How it works
### Inputs and Outputs
**Inputs:**
- Projected 2D Gaussian parameters: mean position $(sx, sy)$ in screen space, diagonal covariance elements $cov2d_{00}$ and $cov2d_{11}$, and splat opacity [S1].
- Framebuffer dimensions: width $w_i$ and height $h_i$ [S1].
- Alpha threshold $\tau$ (minimum opacity for visibility) [S1].
- Maximum squared distance threshold $max\_k2$ [G].

**Outputs:**
- Integer bounding box coordinates $(x_0, y_0, x_1, y_1)$ defining the pixel range to rasterize [S1].
- If the splat is invisible or out of bounds, the output is `None` [S1].

### Data Structures
The method operates on projected 2D Gaussian parameters [S1]. While the parent article computes the full 2D covariance matrix $\Sigma_{2D}$ and its inverse, this method uses the diagonal elements of $\Sigma_{2D}$ for bounding box calculation [S1].

### Steps
1. **Opacity Filter:** If the splat's opacity $\le \tau$, return `None` [S1].
2. **Compute $k^2$:** Calculate the squared distance threshold based on the opacity ratio:
   $$ k^2 = \min\left(2.0 \cdot \ln\left(\frac{\text{opacity}}{\tau}\right), max\_k2\right) $$
   If $k^2 \le 0.0$, return `None` [S1].
3. **Compute Half-Widths:** Calculate the half-extents of the bounding box using the diagonal elements of $\Sigma_{2D}$:
   $$ rx_f = \sqrt{k^2 \cdot cov2d_{00}} $$
   $$ ry_f = \sqrt{k^2 \cdot cov2d_{11}} $$
   If $rx_f$ or $ry_f$ are not finite or $< 1.0$, return `None` [S1].
4. **Compute Integer Bounding Box:**
   $$ x_0 = \lfloor sx - rx_f \rfloor $$
   $$ y_0 = \lfloor sy - ry_f \rfloor $$
   $$ x_1 = \lceil sx + rx_f \rceil $$
   $$ y_1 = \lceil sy + ry_f \rceil $$
   [S1].
5. **Clip to Viewport:**
   $$ x_0 = \max(0, x_0), \quad y_0 = \max(0, y_0) $$
   $$ x_1 = \min(w_i - 1, x_1), \quad y_1 = \min(h_i - 1, y_1) $$
   If $x_0 > x_1$ or $y_0 > y_1$, return `None` [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_adaptive_bounding_box(
    opacity: float,
    cov2d_00: float,
    cov2d_11: float,
    sx: float,
    sy: float,
    w_i: int,
    h_i: int,
    alpha_threshold: float,
    max_k2: float = 100.0  # [G] Typical value to cap box size; not explicitly defined in sources
) -> tuple[int, int, int, int] | None:
    """
    Computes an adaptive AABB for a 2D Gaussian splat.
    
    Args:
        opacity: 3D Gaussian opacity.
        cov2d_00: Diagonal element (0,0) of the 2D screen-space covariance matrix.
        cov2d_11: Diagonal element (1,1) of the 2D screen-space covariance matrix.
        sx: Projected mean x-coordinate.
        sy: Projected mean y-coordinate.
        w_i: Framebuffer width in pixels.
        h_i: Framebuffer height in pixels.
        alpha_threshold: Minimum opacity threshold tau.
        max_k2: Maximum allowed k^2 value to prevent huge boxes.
        
    Returns:
        Tuple (x0, y0, x1, y1) or None if splat is invisible.
    """
    # Step 1: Opacity Filter
    if opacity <= alpha_threshold:
        return None

    # Step 2: Compute k^2
    # Formula: alpha = opacity * exp(-0.5 * d^T Sigma^-1 d)
    # We want alpha >= tau => d^T Sigma^-1 d <= 2 * ln(opacity / tau) = k^2
    log_ratio = opacity / alpha_threshold
    if log_ratio <= 1.0:
        return None  # ln(1) = 0, k^2 <= 0
        
    k2 = 2.0 * math.log(log_ratio)
    k2 = min(k2, max_k2)
    
    if k2 <= 0.0:
        return None

    # Step 3: Compute Half-Widths
    # Using diagonal elements of the covariance matrix for axis-aligned box
    rx_f = math.sqrt(k2 * cov2d_00)
    ry_f = math.sqrt(k2 * cov2d_11)
    
    if not math.isfinite(rx_f) or rx_f < 1.0:
        return None
    if not math.isfinite(ry_f) or ry_f < 1.0:
        return None

    # Step 4: Compute Integer Bounding Box
    x0 = math.floor(sx - rx_f)
    y0 = math.floor(sy - ry_f)
    x1 = math.ceil(sx + rx_f)
    y1 = math.ceil(sy + ry_f)

    # Step 5: Clip to Viewport
    x0 = max(0, x0)
    y0 = max(0, y0)
    x1 = min(w_i - 1, x1)
    y1 = min(h_i - 1, y1)

    if x0 > x1 or y0 > y1:
        return None

    return (x0, y0, x1, y1)
```

### Parameters
- **Alpha Threshold ($\tau$):** The minimum opacity for a splat to be considered visible [S1].
- **Max $k^2$:** Capping factor for the logarithmic term [G].

### Edge Cases and Pitfalls
- **Precision:** If $rx_f$ or $ry_f$ are very small, they are not clamped; instead, the function returns `None` to skip the splat [S1].
- **Tilted Gaussians:** The adaptive bounding box uses the diagonal elements of $\Sigma_{2D}$ [S1]. This assumes the ellipse is axis-aligned [G]. For highly tilted ellipses, the diagonal approximation may be loose [G].
- **Depth Scaling:** Not given in the sources.

## Subtopics

### Original 3DGS Bounding Box
Uses a fixed circular bounding box based on a constant number of standard deviations [S1].

## Operating characteristics
- **Time Complexity:** $O(1)$ per Gaussian for bounding box computation [G].
- **Memory Complexity:** $O(1)$ auxiliary memory [G].
- **Performance:** Not given in the sources.
- **Strengths:** Significantly reduces overdraw and tile intersections, especially for low-opacity or elongated Gaussians [S1].
- **Limitations:** Requires accurate 2D covariance projection. If the covariance is not stabilized (e.g., near singular), the box may be incorrect [G].

## Background
The method was introduced as an improvement over the original 3D Gaussian Splatting rendering pipeline, which used fixed circular bounding boxes [S1]. It is implemented in unofficial libraries like `tsplat` [S1]. It addresses the inefficiency of the original method where large, transparent Gaussians would affect many tiles unnecessarily [S1].

## Sources
[S1] Tsplat – Run Gaussian splatting in your terminal, https://github.com/darshanmakwana412/tsplat, fetched 2026-10-26 (unofficial)
