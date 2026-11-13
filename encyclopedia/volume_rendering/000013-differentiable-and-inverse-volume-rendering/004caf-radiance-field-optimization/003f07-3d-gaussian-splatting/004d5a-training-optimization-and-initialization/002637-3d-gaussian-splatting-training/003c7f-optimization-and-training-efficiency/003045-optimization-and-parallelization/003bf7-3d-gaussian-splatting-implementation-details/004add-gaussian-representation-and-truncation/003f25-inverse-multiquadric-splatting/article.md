# Inverse Multiquadric Splatting

Also called: 3DIQS, Inverse Multiquadric

## Overview
Inverse Multiquadric Splatting is a rendering technique that replaces the standard Gaussian radial basis function with an inverse multiquadric function to define the splat's falloff profile [S1]. It solves the problem of modeling long-tailed or non-Gaussian decay in radiance fields by introducing a correction factor to the covariance matrix and a specific weighting scheme [S1]. It differs from standard Gaussian Splatting by using a slower decay rate for large distances from the mean, which can better represent certain scene geometries, and from [Opacity-Independent Gaussian Truncation](10266-opacity-independent-gaussian-truncation.md) by maintaining opacity coupling through a cosine-like weighting term derived from the ellipse metric [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** 3D Gaussian parameters (mean $\mu$, covariance $\Sigma$, opacity $\alpha_{base}$, color), camera projection matrix, and image resolution.
*   **Output:** Screen-space color and alpha buffer.
*   **Gradients:** Gradients w.r.t. mean, covariance, and opacity are computed during backpropagation.

**Data Structures:**
The method operates on the standard 3D Gaussian Splatting representation. The 2D projection is represented by the conic matrix $C = (\Sigma'_{2 \times 2})^{-1}$, stored as symmetric elements $(a, b, c)$ corresponding to the inverse covariance matrix entries, and opacity weight $w$ [S1].

**Parameters:**
*   `correctionFactor`: A scalar multiplier applied to the 3D covariance matrix $\Sigma$ and the 2D covariance matrix $\Sigma'_{2 \times 2}$ (or its inverse, the conic matrix) during forward and backward passes. Value: $1.38$ [S1].
*   `ellipse_threshold_max`: The upper bound for the ellipse metric to skip splats. Value: $9.0$ [S1].
*   `alpha_max`: The maximum clamped alpha value. Value: $0.99$ [S1].

**Step-by-Step Method:**

1.  **Covariance Correction:** Multiply the 3D covariance matrix $\Sigma$ by `correctionFactor` ($1.38$) before projection [S1].
2.  **2D Projection:** Project the corrected 3D covariance to 2D using the camera Jacobian $J$ to get $\Sigma'$. Compute the 2D conic matrix $C = (\Sigma'_{2 \times 2})^{-1}$ with elements $(a, b, c)$ [S1].
3.  **Ellipse Metric Calculation:** For a pixel at screen coordinates $(x, y)$ relative to the projected mean $\mu'$, compute the displacement vector $d = (d.x, d.y) = (x - \mu'_x, y - \mu'_y)$. Calculate the ellipse metric $ellipse$ (representing the squared Mahalanobis distance scaled by the inverse covariance):
    $$ ellipse = a \cdot d.x^2 + c \cdot d.y^2 + 2 \cdot b \cdot d.x \cdot d.y $$
    This corresponds to $z^T (\Sigma'_{2 \times 2})^{-1} z$ where $z = x - \mu'$ [S1].
4.  **Culling:**
    *   If $ellipse \le 0$, skip the splat [S1].
    *   If $ellipse \ge 9.0$, skip the splat [S1].
5.  **Weight Calculation:** Compute the inverse multiquadric weight $w$ (denoted as $G$ in code):
    $$ G = \frac{1}{ellipse + 1} $$
    This is derived from the 2D splat function:
    $$ w = \frac{1}{\left[ \frac{1}{\xi} (x - \mu')^T (\Sigma'_{2 \times 2})^{-1} (x - \mu') + 1 \right]} $$
    where the term $\frac{1}{\xi}$ is effectively absorbed or normalized such that the denominator becomes $ellipse + 1$ in the implementation context [S1].
6.  **Alpha Computation:** Compute the final alpha value:
    $$ \alpha = \min(0.99, w \cdot G) $$
    Note: The source code uses `con_o.w` (the opacity weight from the conic matrix) multiplied by $G$. The text describes this as $w \times$ inverse quadric, where $w$ in the text formula (30) is the spatial weight, and `con_o.w` is the scalar opacity parameter. The code snippet `min(0.99f, con_o.w * (1.f / (ellipse + 1.f)))` implies the final alpha is the product of the stored opacity weight and the spatial decay factor $G$ [S1].
7.  **Blending:** Blend the color contribution of the Gaussian into the framebuffer using standard alpha blending with the computed $\alpha$ [S1].

**Backward Pass (Gradients):**
The gradients are computed with respect to the loss $L$. Let $dL\_d\alpha$ be the gradient of the loss w.r.t. alpha.

1.  **Gradient w.r.t. Weight $G$:**
    $$ dL\_dG = con\_o.w \cdot dL\_d\alpha $$
2.  **Gradient w.r.t. Ellipse Metric:**
    $$ dL\_d\delta x = dL\_dG \cdot \frac{-2}{(ellipse + 1)^2} \cdot (d.x \cdot a + d.y \cdot b) $$
    $$ dL\_d\delta y = dL\_dG \cdot \frac{-2}{(ellipse + 1)^2} \cdot (d.x \cdot b + d.y \cdot c) $$
    In the source code, this is optimized using temporary variables:
    $$ ellipsepow = \frac{-2 \cdot dL\_dG}{(ellipse + 1)^2} $$
    $$ dG\_ddelx = ellipsepow \cdot (d.x \cdot a + d.y \cdot b) $$
    $$ dG\_ddely = ellipsepow \cdot (d.x \cdot b + d.y \cdot c) $$
    These gradients are accumulated into $dL\_dmean2D$ [S1].
3.  **Gradient w.r.t. Conic Matrix Elements:**
    The gradients w.r.t. the conic matrix entries $(a, b, c)$ are updated using the chain rule through the ellipse metric. The source code applies `correctionFactor` to the gradient updates for the covariance entries:
    $$ dL\_da = correctionFactor \cdot \dots $$
    $$ dL\_db = correctionFactor \cdot \dots $$
    $$ dL\_dc = correctionFactor \cdot \dots $$
    Specifically, the code updates the conic gradients using `halfellipsepow` (which is $0.5 \cdot ellipsepow$) scaled by the squared displacements $d.x^2, d.x \cdot d.y, d.y^2$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def inverse_multiquadric_splat_forward(conic_a, conic_b, conic_c, opacity_weight, mean_x, mean_y, pixel_x, pixel_y):
    """
    Forward pass for a single pixel contribution.
    Returns alpha and color contribution.
    """
    correction_factor = 1.38
    
    # Displacement vector
    dx = pixel_x - mean_x
    dy = pixel_y - mean_y
    
    # Ellipse metric: z^T * C * z
    ellipse = conic_a * dx * dx + conic_c * dy * dy + 2.0 * conic_b * dx * dy
    
    # Culling
    if ellipse <= 0.0 or ellipse >= 9.0:
        return 0.0, 0.0, 0.0
        
    # Inverse multiquadric weight G
    G = 1.0 / (ellipse + 1.0)
    
    # Alpha calculation
    alpha = min(0.99, opacity_weight * G)
    
    return alpha, G, ellipse

def inverse_multiquadric_splat_backward(alpha, G, ellipse, opacity_weight, dx, dy, dL_dalpha, conic_a, conic_b, conic_c):
    """
    Computes gradients for mean and conic matrix.
    Returns dL_dmean_x, dL_dmean_y, dL_dconic_a, dL_dconic_b, dL_dconic_c.
    """
    correction_factor = 1.38
    
    # Gradient w.r.t G
    dL_dG = opacity_weight * dL_dalpha
    
    # Gradient w.r.t ellipse metric
    # dG/d(ellipse) = -1 / (ellipse + 1)^2
    # dL_d(ellipse) = dL_dG * dG/d(ellipse)
    ellipse_pow_term = -2.0 * dL_dG / ((ellipse + 1.0) ** 2)
    
    # Gradients w.r.t mean (via ellipse metric)
    # d(ellipse)/d(dx) = 2 * (conic_a * dx + conic_b * dy)
    # d(ellipse)/d(dy) = 2 * (conic_b * dx + conic_c * dy)
    # Note: The source code uses a simplified form for dG_ddelx/y which seems to 
    # correspond to the derivative of the weight w.r.t the displacement components 
    # directly in the context of the mean update.
    
    dG_ddelx = ellipse_pow_term * (dx * conic_a + dy * conic_b)
    dG_ddely = ellipse_pow_term * (dx * conic_b + dy * conic_c)
    
    dL_dmean_x = dG_ddelx
    dL_dmean_y = dG_ddely
    
    # Gradients w.r.t conic matrix elements
    # d(ellipse)/da = dx^2
    # d(ellipse)/db = 2 * dx * dy
    # d(ellipse)/dc = dy^2
    # dL_dconic = dL_d(ellipse) * d(ellipse)/d_conic
    # However, the source code applies correction_factor to the covariance gradients.
    # The conic matrix is the inverse covariance. The code updates dL_dconic directly.
    
    half_ellipse_pow = 0.5 * ellipse_pow_term
    
    dL_dconic_a = correction_factor * half_ellipse_pow * (dx * dx)
    dL_dconic_b = correction_factor * half_ellipse_pow * (dx * dy) # Note: source uses 2*b*... in ellipse, so derivative w.r.t b has factor 2*dx*dy. 
    # Wait, source code: atomicAdd(&dL_dconic2D[global_id].y, halfellipsepow * d.x * d.y);
    # The ellipse formula is a*dx^2 + c*dy^2 + 2*b*dx*dy.
    # d(ellipse)/db = 2*dx*dy.
    # The code uses halfellipsepow * dx * dy. 
    # If dL_dconic_b is the gradient w.r.t b, then dL/db = dL/d(ellipse) * 2*dx*dy.
    # ellipse_pow_term is dL/d(ellipse) * 2? No, ellipse_pow_term is -2*dL_dG/(...)^2.
    # Let's stick to the source code logic exactly.
    
    dL_dconic_a = correction_factor * half_ellipse_pow * (dx * dx)
    dL_dconic_b = correction_factor * half_ellipse_pow * (dx * dy)
    dL_dconic_c = correction_factor * half_ellipse_pow * (dy * dy)
    
    return dL_dmean_x, dL_dmean_y, dL_dconic_a, dL_dconic_b, dL_dconic_c
```

## Operating characteristics
*   **Time Complexity:** Same as standard Gaussian Splatting, $O(N \cdot P)$ where $N$ is the number of Gaussians and $P$ is the number of pixels covered, with a constant factor overhead for the additional arithmetic in the weight calculation [S1].
*   **Memory Complexity:** Identical to standard 3D Gaussian Splatting, storing mean, covariance (or conic), opacity, and color for each Gaussian [S1].
*   **Strengths:** Provides a different decay profile that may better fit certain anisotropic or long-tailed structures in the scene compared to the standard Gaussian [S1].
*   **Limitations:** The correction factor $1.38$ is a hyperparameter that requires tuning; the culling threshold of $9.0$ limits the effective radius of the splats [S1].
*   **When to use:** When the standard Gaussian falloff does not adequately represent the visual appearance of the scene, particularly for objects with diffuse or non-Lambertian properties that exhibit slower decay [S1].

## Background
Inverse Multiquadric Splatting was introduced in the paper "DARB-Splatting: Generalizing Splatting with Decaying Anisotropic Radial Basis Functions" [S1]. It is part of the DARB-Splatting framework, which generalizes the splatting process by allowing different radial basis functions. The method modifies the CUDA rasterizer to support the inverse multiquadric function, requiring specific adjustments to the forward and backward propagation steps [S1].

## Sources
[S1] DARB-Splatting: Generalizing Splatting with Decaying Anisotropic Radial Basis Functions, https://doi.org/10.1109/WACV61042.2026.00438, fetched 2026-10-30
