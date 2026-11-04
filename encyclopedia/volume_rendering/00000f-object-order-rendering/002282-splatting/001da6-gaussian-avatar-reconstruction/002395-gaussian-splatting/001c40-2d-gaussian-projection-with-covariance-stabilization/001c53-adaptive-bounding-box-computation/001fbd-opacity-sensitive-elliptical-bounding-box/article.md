# Opacity-Sensitive Elliptical Bounding Box

Also called: Elliptical Bounding Box, Non-square Bounding Box

## Overview
A method to compute 2D Gaussian projections as elliptical disks rather than axis-aligned bounding boxes (AABBs) to reduce false-positive tile intersections [S1]. It sets itself apart by combining an elliptical shape approximation with non-square bounding boxes, significantly reducing the number of duplicated Gaussians processed per tile compared to the original square AABB approach [S1].

## How it works
**Inputs and Outputs:**
- Input: 2D Gaussian parameters (mean $\mu$, covariance $\Sigma$), opacity $\alpha$, opacity threshold $\epsilon$, and tile grid dimensions [S1].
- Output: A non-square axis-aligned bounding box that tightly encloses the elliptical projection of the Gaussian [S1].

**Data Structures:**
- The bounding box is defined by minimum and maximum coordinates $(x_{min}, y_{min})$ and $(x_{max}, y_{max})$ [S1].
- Unlike the original method which computes a square box based on the maximum extent, this method computes distinct $x$ and $y$ extents based on the elliptical shape [S1].

**Steps:**
1. **Opacity Thresholding:** If the Gaussian's opacity $\tau$ falls below a threshold $\epsilon$, the bounding box is considered empty and the Gaussian is filtered out immediately [S1].
2. **Elliptical Projection:** Convert the 2D Gaussian covariance $\Sigma$ into an elliptical disk representation [S1].
3. **Non-Square Box Calculation:** Calculate the width and height of the bounding box independently based on the elliptical extents, allowing for non-square boxes [S1].
4. **Tile Intersection:** Check if the non-square bounding box intersects with the current tile [S1].

**Formulas:**
- The bounding box is derived from the ellipse that encloses the Gaussian at a specific confidence level [G].
- For a 2D covariance matrix $\Sigma$, the eigenvalues $\lambda_1, \lambda_2$ yield the squared semi-axis lengths [G].
- The axis-aligned bounding box of the rotated ellipse is computed from these eigenvalues and the eigenvector rotation angle [G].
- The opacity threshold condition is: if $\tau < \epsilon$, the box is empty [S1].

**Parameters:**
- $\epsilon$: Opacity threshold for filtering [S1].

**Edge Cases and Pitfalls:**
- Large, mostly transparent 3D Gaussians can affect a significant number of tiles in the original approach, leading to higher render times; this method effectively filters them out by considering low opacity [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_opacity_sensitive_elliptical_bbox(mean, covariance, opacity, epsilon):
    # Step 1: Opacity thresholding
    if opacity < epsilon:
        return None  # Box is empty, filter out [S1]

    # Step 2: Compute elliptical extents [G]
    a11, a12 = covariance[0][0], covariance[0][1]
    a21, a22 = covariance[1][0], covariance[1][1]
    trace = a11 + a22
    det = a11 * a22 - a12 * a21
    disc = trace * trace - 4 * det
    sqrt_disc = math.sqrt(max(0.0, disc))
    lambda1 = (trace + sqrt_disc) / 2
    lambda2 = (trace - sqrt_disc) / 2

    # Semi-axis lengths scaled by confidence factor [G]
    scale = 2.447  # [G] Standard confidence scaling factor
    a = math.sqrt(lambda1) * scale
    b = math.sqrt(lambda2) * scale

    # Eigenvector angle for rotation [G]
    if abs(a12) > 1e-8:
        angle = math.atan2(lambda1 - a11, a12)
    else:
        angle = 0.0

    # Step 3: Non-square bounding box calculation [G]
    cos_a = math.cos(angle)
    sin_a = math.sin(angle)
    width = 2.0 * math.sqrt((a * cos_a)**2 + (b * sin_a)**2)
    height = 2.0 * math.sqrt((a * sin_a)**2 + (b * cos_a)**2)

    x_min = mean[0] - width / 2
    y_min = mean[1] - height / 2
    x_max = mean[0] + width / 2
    y_max = mean[1] + height / 2

    return (x_min, y_min, x_max, y_max)
```

## Subtopics

### Staged-fragment Reduction
A reduction technique for per-touched-tile duplicated splats using tighter AABBs and an approximate tile-ellipse intersection test [S2]. Developed concurrently with Speedy-Splat [S2].

## Operating characteristics
- On the Bicycle scene at native resolution with 16x16 tiles, original AABBs resulted in 87,067,709 duplicated Gaussians [S1].
- Using opacity-sensitive ellipses with square boxes reduced this to 55,648,264 [S1].
- Allowing non-square bounding boxes further reduced duplicated Gaussians to 18,466,556 [S1].
- Exact pixel-perfect intersections would result in only 10,976,269 duplicated Gaussians, but are computationally too expensive; the elliptical/non-square approach provides a trade-off [S1].
- In total, on the Bicycle scene at native resolution, the method decreases the render time and GPU memory usage by factors of 3.77 and 2.11, respectively [S1].
- The method is more effective at not rendering large, mostly transparent 3D Gaussians, leading to more consistent render times [S1].

## Background
The method was introduced in "Fast 3D Gaussian Splatting Rendering via Easily Integrable Improvements" [S1]. It addresses the inefficiency of the original 3DGS method's square bounding boxes, which often include many empty pixels, especially for large, transparent Gaussians [S1].

## Sources
[S1] Fast 3D Gaussian Splatting Rendering via Easily Integrable Improvements, https://doi.org/10.1109/lsp.2024.3521379, fetched 2026-10-26
[S2] Splatshop: Efficiently Editing Large Gaussian Splat Models, https://doi.org/10.1111/cgf.70214, fetched 2026-10-31
