# SnugBox

Also called: FastGS Optimizations, Rasterization Optimizations, Speedy-Splat Optimizations

## Overview
SnugBox is a precise algorithm for computing Gaussian-to-tile bounding box intersections in 3D Gaussian Splatting, replacing the overly conservative tile assignment of standard 3D-GS to reduce the number of pixels processed per Gaussian [S1][S2, unofficial]. It computes a tight axis-aligned bounding box (AABB) around each Gaussian's elliptical extent in image space, assigning the Gaussian only to tiles that strictly intersect this box [S1][S2, unofficial]. Unlike the standard method which inscribes a circle and uses a square bounding box, SnugBox derives the box directly from the conic equation of the projected Gaussian, providing numerical stability under edge-on tilts [S3, unofficial]. It serves as a lossless optimization that reduces the number of Gaussian-tile pairs, contributing to an average inference speed-up of 1.82× [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** 3D Gaussian parameters (mean $\mu_{i3D}$, scale $s_i$, rotation $r_i$), camera parameters (view matrix $W$, projection matrix), tile size $T$, image dimensions.
*   **Output:** A set of tile indices $(t_x, t_y)$ that the Gaussian intersects.

**Data Structures:**
*   $\Sigma_{i3D}$: $3 \times 3$ 3D covariance matrix.
*   $\Sigma_{i2D}$: $2 \times 2$ 2D projected covariance matrix.
*   $\Sigma_{i2D}^{-1}$: Inverse of the 2D covariance matrix, represented as $\begin{pmatrix} a & b \\ b & c \end{pmatrix}$.
*   $\mu_{i2D}$: 2D mean position $(\mu_x, \mu_y)$.
*   $t$: Threshold value for the ellipse equation.

**Steps:**

1.  **Project 3D Covariance to 2D:**
    Compute the 2D covariance matrix $\Sigma_{i2D}$ from the 3D covariance $\Sigma_{i3D}$ using the Jacobian $J$ of the perspective projection [S1]:
    $$ \hat{\Sigma}_{i3D} = J W \Sigma_{i3D} W^T J^T $$
    Drop the last row and column of $\hat{\Sigma}_{i3D}$ to obtain $\Sigma_{i2D}$ [S1].

2.  **Compute Inverse Covariance and Threshold:**
    Compute the inverse of the 2D covariance matrix $\Sigma_{i2D}^{-1} = \begin{pmatrix} a & b \\ b & c \end{pmatrix}$ [S1].
    Calculate the threshold $t$ based on the Gaussian's opacity $\sigma_i$:
    $$ t = 2 \log(255 \sigma_i) $$
    [S1]

3.  **Compute Extrema in X:**
    Find the values of $x_d$ (where $x_d = p_x - \mu_x$) where the derivative $\partial y_d / \partial x_d = 0$. These are the $x$-coordinates of the vertical tangents of the ellipse [S1]:
    $$ x_{d_{args}} = \pm \sqrt{\frac{-b^2 t}{(b^2 - ac)a}} $$
    [S1]
    *Note: If the term inside the square root is negative or $a=0$, the ellipse is degenerate or axis-aligned in a way that requires handling (e.g., treating as axis-aligned bounds).* [G]

4.  **Compute Extrema in Y:**
    Substitute $x_{d_{args}}$ into the rearranged ellipse equation to find the corresponding $y_d$ values. The equation for $y_d$ is:
    $$ y_d = \frac{-bx_d \pm \sqrt{(b^2 - ac)x_d^2 + tc}}{c} $$
    [S1]
    Calculate $y_{min}$ and $y_{max}$ by evaluating this for both $x_{d_{args}}$ values and taking the minimum and maximum results. Add $\mu_y$ to get the absolute $y$ bounds:
    $$ y_{min} = \min(y_d) + \mu_y, \quad y_{max} = \max(y_d) + \mu_y $$
    [S1]

5.  **Compute Extrema in X (Absolute):**
    Due to the symmetry of the ellipse equation, find $x_{min}$ and $x_{max}$ by swapping $y_d$ and $x_d$ and constants $a$ and $c$ in the equations above, or by solving for $y_d$ extrema and substituting back [S1].
    $$ x_{min} = \min(x_d) + \mu_x, \quad x_{max} = \max(x_d) + \mu_x $$
    [S1]

6.  **Convert to Tile Indices:**
    Convert the bounding box edges $(x_{min}, y_{min}, x_{max}, y_{max})$ to tile indices by dividing by the tile size $T$, rounding to the nearest integer (or floor/ceil appropriately for inclusive bounds), and clipping to the image boundaries [S1]:
    $$ t_x^{start} = \max(0, \lfloor x_{min} / T \rfloor), \quad t_x^{end} = \min(W_{img}/T, \lceil x_{max} / T \rceil) $$
    $$ t_y^{start} = \max(0, \lfloor y_{min} / T \rfloor), \quad t_y^{end} = \min(H_{img}/T, \lceil y_{max} / T \rceil) $$
    [S1]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_snugbox_tile_indices(gaussian_mean_2d, cov_inv_2d, opacity, tile_size, image_width, image_height):
    """
    Computes the tile indices intersected by a Gaussian using SnugBox.
    
    Args:
        gaussian_mean_2d: Tuple (mu_x, mu_y)
        cov_inv_2d: Tuple (a, b, c) representing [[a, b], [b, c]]
        opacity: Float sigma_i
        tile_size: Float T
        image_width: Int W_img
        image_height: Int H_img
        
    Returns:
        Tuple (t_x_start, t_y_start, t_x_end, t_y_end)
    """
    mu_x, mu_y = gaussian_mean_2d
    a, b, c = cov_inv_2d
    
    # Step 2: Compute threshold t
    # t = 2 * log(255 * sigma_i)
    # Guard against non-positive opacity
    if opacity <= 0:
        return (0, 0, 0, 0)
        
    t = 2.0 * math.log(255.0 * opacity)
    
    # Step 3: Compute x_d extrema (x_d_args)
    # x_d_args = +/- sqrt( (-b^2 * t) / ((b^2 - a*c) * a) )
    # Check for degenerate cases where denominator is zero or negative
    denom_x = (b**2 - a*c) * a
    if denom_x <= 0:
        # If the ellipse is degenerate or axis-aligned such that vertical tangents are at infinity,
        # we fall back to a simpler bound or handle as axis-aligned.
        # For a standard valid 2D Gaussian, this should not happen if covariance is positive definite.
        # However, if a=0, the ellipse is vertical/horizontal aligned differently.
        # We use a safe fallback: assume infinite extent in x if derivative is undefined?
        # In practice, if a=0, the ellipse equation is 2bx_dy_d + cy_d^2 = t.
        # This implies y_d is bounded but x_d is not? No, if a=0, it's a parabola-like shape in x?
        # Actually, if a=0, the inverse covariance has 0 in top-left, meaning variance in x is infinite?
        # This indicates a degenerate Gaussian. We clamp to a small box or skip.
        # For robustness, we return empty or full image. Let's return empty for safety.
        return (0, 0, 0, 0)
        
    sqrt_term_x = math.sqrt((-b**2 * t) / denom_x)
    x_d_args = [sqrt_term_x, -sqrt_term_x]
    
    # Step 4: Compute y_d extrema
    # y_d = (-b*x_d +/- sqrt((b^2 - a*c)*x_d^2 + t*c)) / c
    y_d_values = []
    det = b**2 - a*c
    
    for x_d in x_d_args:
        # Calculate discriminant for y_d
        disc = det * (x_d**2) + t * c
        if disc < 0:
            # Numerical noise might cause slight negative values
            disc = 0.0
        sqrt_disc = math.sqrt(disc)
        
        # Guard against c=0
        if c == 0:
            continue
            
        y_d1 = (-b * x_d + sqrt_disc) / c
        y_d2 = (-b * x_d - sqrt_disc) / c
        y_d_values.append(y_d1)
        y_d_values.append(y_d2)
        
    if not y_d_values:
        return (0, 0, 0, 0)
        
    y_min_d = min(y_d_values)
    y_max_d = max(y_d_values)
    
    y_min = y_min_d + mu_y
    y_max = y_max_d + mu_y
    
    # Step 5: Compute x_d extrema (symmetric to y)
    # Swap a and c, and x_d and y_d logic.
    # We need to find y_d_args (extrema in y) to get x_d bounds.
    # The formula for x_d given y_d is:
    # x_d = (-b*y_d +/- sqrt((b^2 - a*c)*y_d^2 + t*a)) / a
    
    # Find y_d_args: where dx_d/dy_d = 0
    # x_d_args formula was: +/- sqrt( (-b^2 * t) / ((b^2 - a*c) * a) )
    # For y_d_args, we swap a and c:
    # y_d_args = +/- sqrt( (-b^2 * t) / ((b^2 - a*c) * c) )
    
    denom_y = (b**2 - a*c) * c
    if denom_y <= 0:
        return (0, 0, 0, 0)
        
    sqrt_term_y = math.sqrt((-b**2 * t) / denom_y)
    y_d_args = [sqrt_term_y, -sqrt_term_y]
    
    x_d_values = []
    for y_d in y_d_args:
        disc = det * (y_d**2) + t * a
        if disc < 0:
            disc = 0.0
        sqrt_disc = math.sqrt(disc)
        
        if a == 0:
            continue
            
        x_d1 = (-b * y_d + sqrt_disc) / a
        x_d2 = (-b * y_d - sqrt_disc) / a
        x_d_values.append(x_d1)
        x_d_values.append(x_d2)
        
    if not x_d_values:
        return (0, 0, 0, 0)
        
    x_min_d = min(x_d_values)
    x_max_d = max(x_d_values)
    
    x_min = x_min_d + mu_x
    x_max = x_max_d + mu_x
    
    # Step 6: Convert to tile indices
    t_x_start = max(0, math.floor(x_min / tile_size))
    t_x_end = min(image_width // tile_size, math.ceil(x_max / tile_size))
    t_y_start = max(0, math.floor(y_min / tile_size))
    t_y_end = min(image_height // tile_size, math.ceil(y_max / tile_size))
    
    return (t_x_start, t_y_start, t_x_end, t_y_end)
```

**Parameters:**
*   $\sigma_i$: Gaussian opacity. Typical values depend on the training data, but the threshold $t$ scales with $\log(\sigma_i)$.
*   Tile Size $T$: Typically 16 or 32 pixels in standard 3D-GS implementations.

**Edge Cases and Pitfalls:**
*   **Degenerate Covariance:** If $a=0$ or $c=0$, the inverse covariance is singular or the ellipse is axis-aligned in a way that makes the derivative zero at infinity. The pseudocode handles this by returning empty bounds or skipping, which effectively culls the Gaussian or requires a fallback. [G]
*   **Numerical Stability:** Under edge-on tilts, the projected covariance can become ill-conditioned. SnugBox is noted to provide numerical stability in these cases compared to simpler bounding circles [S3, unofficial].
*   **Negative Discriminant:** Due to floating-point errors, the term inside the square root for $y_d$ or $x_d$ might be slightly negative. It should be clamped to 0. [G]

## Subtopics

### [Compact Box (CB)](17690-compact-box-cb.md)
A preprocessing optimization that further shrinks the 2D Gaussian footprint by introducing a scaling factor $\beta$ into the alpha blending equation, reducing the number of Gaussian-tile pairs beyond what SnugBox achieves.

### [Compact Box (CB)](17690-compact-box-cb.md)
An extension of SnugBox that computes exact Gaussian-tile intersections during the duplicate-keys phase, emitting keys only for tiles the ellipse strictly overlaps, providing an additional speed-up over SnugBox alone.

## Operating characteristics
*   **Time Complexity:** Constant number of operations per Gaussian ($O(1)$) [S1]. It is called twice in the rendering pipeline: once to count Gaussians in the preprocess step and once to populate the Gaussian-to-tile arrays in `duplicateWithKeys` [S1].
*   **Memory Complexity:** No additional significant memory overhead compared to standard 3D-GS, as it only computes bounds [S1].
*   **Performance:** Produces an average overall inference speed-up of 1.82× on Mip-NeRF 360, Tanks & Temples, and Deep Blending datasets compared to baseline 3D-GS [S1]. When combined with AccuTile, it contributes to a 2× FPS boost [S2, unofficial].
*   **Strengths:** Lossless (does not change rendered image), precise tile assignment reduces overestimation, numerically stable [S1][S2, S3, unofficial].
*   **Limitations:** Requires computation of inverse covariance and eigenvalues/square roots, which is more expensive per Gaussian than the simple circle radius check in standard 3D-GS, but the reduction in tile processing offsets this [S1].

## Background
SnugBox was introduced in the "Speedy-Splat" paper by Hanson et al. (2025) [S1]. It addresses the inefficiency of the standard 3D-GS tile assignment, which uses a square bounding box inscribing a circle with radius $3\sqrt{\lambda_{max}}$, leading to overly conservative tile assignments [S1][S2, unofficial]. SnugBox is part of a suite of lossless optimizations including AccuTile and pruning methods, designed to maximize rendering speed while maintaining visual fidelity [S1][S3, unofficial]. It builds on the concept of using the elliptical extent of the Gaussian for more precise culling [S1].

## Sources
[S1] Speedy-Splat: Fast 3D Gaussian Splatting with Sparse Pixels and Sparse Primitives, https://doi.org/10.1109/cvpr52734.2025.02006, fetched 2026-10-26
[S2] Poster Speedy-Splat: Fast 3D Gaussian Splatting with Sparse Pixels and Sparse Primitives, https://speedysplat.github.io/static/files/speedysplat_poster.pdf?v=2026.10.18, fetched 2026-10-29 (unofficial)
[S3] Bake It Till You Make It: Ultrafast Spatial Texture-Atlas Splatting, https://doi.org/10.48550/arXiv.2607.13808, fetched 2026-10-29 (unofficial)
