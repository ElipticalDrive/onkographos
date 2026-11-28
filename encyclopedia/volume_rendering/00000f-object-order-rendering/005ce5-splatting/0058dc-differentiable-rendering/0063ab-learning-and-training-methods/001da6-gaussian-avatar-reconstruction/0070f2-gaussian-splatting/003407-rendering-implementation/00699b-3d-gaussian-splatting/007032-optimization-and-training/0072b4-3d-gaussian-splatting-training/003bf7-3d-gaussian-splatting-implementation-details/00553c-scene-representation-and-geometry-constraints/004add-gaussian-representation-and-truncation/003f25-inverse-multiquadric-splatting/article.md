# Inverse Multiquadric Splatting

Also called: 3DIQS, Inverse Multiquadric

## Overview
Inverse Multiquadric Splatting is a rendering technique that replaces the standard Gaussian density function with an inverse multiquadric radial basis function to define the splat's alpha contribution. It solves the problem of over-saturation and artifacts in high-opacity regions by providing a different decay profile for the splat's weight. It differs from the standard Gaussian splatting [3D Gaussian Splatting](3d-gaussian-splatting.md) by using a rational function rather than an exponential decay, and from [Opacity-Independent Gaussian Truncation](10266-opacity-independent-gaussian-truncation.md) by modifying the core density function rather than just the truncation threshold.

## How it works
**Inputs and Outputs:**
*   **Input:** A set of 3D Gaussians $\{G_i\}$, where each Gaussian has mean $\boldsymbol{\mu}_i$, 3D covariance $\boldsymbol{\Sigma}_i$, opacity $\alpha_i$, and view-dependent color $\mathbf{c}_i$. For a specific pixel/fragment, the 2D projected mean $\boldsymbol{\mu}'$ and 2D conic matrix (inverse covariance) $\mathbf{C}_{2D} = (\Sigma'_{2 \times 2})^{-1}$ are computed.
*   **Output:** The final alpha value $\alpha$ for the splat, and gradients for the mean and conic matrix during backpropagation.

**Data Structures:**
The method operates on the standard 3D Gaussian Splatting representation. The 2D conic matrix is typically stored as a symmetric 2x2 matrix or its upper-triangular components. In the implementation, the conic matrix entries are accessed as `con_o.x` (top-left), `con_o.y` (top-right/bottom-left), and `con_o.z` (bottom-right), with `con_o.w` representing the opacity.

**Parameters:**
*   `correctionFactor`: A scalar multiplier applied to the 3D covariance matrix $\boldsymbol{\Sigma}$ and the 2D covariance matrix $\Sigma'_{2 \times 2}$ to adjust the spread of the splat. Value: $1.38$ [S1].

**Steps:**

1.  **Covariance Correction:** Multiply the 3D covariance matrix $\boldsymbol{\Sigma}$ by `correctionFactor` ($1.38$) before projection to 2D [S1].
2.  **Compute Ellipse Term:** Calculate the squared Mahalanobis distance (ellipse term) using the 2D conic matrix and the offset vector $\mathbf{d} = \mathbf{x} - \boldsymbol{\mu}'$:
    $$ \text{ellipse} = \mathbf{d}^\top \mathbf{C}_{2D} \mathbf{d} = C_{xx} d_x^2 + C_{zz} d_y^2 + 2 C_{xy} d_x d_y $$
    where $C_{xx}, C_{xy}, C_{zz}$ are the components of the 2D conic matrix [S1].
3.  **Boundary Checks:**
    *   If $\text{ellipse} \le 0$, skip the splat (continue) [S1].
    *   If $\text{ellipse} \ge 9$, skip the splat (continue) [S1]. This limits the effective support of the splat.
4.  **Compute Weight ($w$):** Calculate the inverse multiquadric weight:
    $$ w = \frac{1}{\text{ellipse} + 1} $$
    This corresponds to the 2D splat function derived from the 3D inverse multiquadric function $\frac{1}{\frac{1}{\xi} (d_M)^2 + 1}$ with $\xi=1$ [S1].
5.  **Compute Alpha:** Combine the weight with the base opacity:
    $$ \alpha = \min(0.99, \alpha_{base} \cdot w) $$
    where $\alpha_{base}$ is the stored opacity of the Gaussian (accessed as `con_o.w`) [S1].

**Pseudocode (Forward Pass):**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def inverse_multiquadric_splat_forward(conic_matrix, opacity, d_x, d_y):
    """
    conic_matrix: tuple (C_xx, C_xy, C_zz) representing the 2D conic matrix
    opacity: float, the base opacity alpha_base
    d_x, d_y: float, offset from mean to pixel center
    """
    C_xx, C_xy, C_zz = conic_matrix
    
    # Step 2: Compute ellipse term
    ellipse = C_xx * d_x * d_x + C_zz * d_y * d_y + 2.0 * C_xy * d_x * d_y
    
    # Step 3: Boundary checks
    if ellipse <= 0.0:
        return None, None
    if ellipse >= 9.0:
        return None, None
        
    # Step 4: Compute weight
    w = 1.0 / (ellipse + 1.0)
    
    # Step 5: Compute alpha
    alpha = min(0.99, opacity * w)
    
    return alpha, w
```

**Pseudocode (Backward Pass):**

The gradients are computed with respect to the loss $L$. Let $dL\_d\alpha$ be the gradient of the loss w.r.t. alpha.

1.  Compute gradient w.r.t. the weight term $G$ (where $G = w$):
    $$ dL\_dG = \alpha_{base} \cdot dL\_d\alpha $$
2.  Compute gradient w.r.t. the ellipse term:
    $$ \text{ellipsepow} = \frac{-2 \cdot dL\_dG}{(\text{ellipse} + 1)^2} $$
3.  Compute gradients w.r.t. the offset components $d_x, d_y$:
    $$ \frac{\partial G}{\partial d_x} = \text{ellipsepow} \cdot (C_{xx} d_x + C_{xy} d_y) $$
    $$ \frac{\partial G}{\partial d_y} = \text{ellipsepow} \cdot (C_{xy} d_x + C_{zz} d_y) $$
4.  Update gradients for the 2D mean (assuming $\mathbf{d} = \mathbf{x} - \boldsymbol{\mu}'$, so $\frac{\partial \mathbf{d}}{\partial \boldsymbol{\mu}'} = -1$):
    $$ \frac{\partial L}{\partial \mu'_x} = -\frac{\partial G}{\partial d_x} $$
    $$ \frac{\partial L}{\partial \mu'_y} = -\frac{\partial G}{\partial d_y} $$
5.  Compute gradients w.r.t. the conic matrix components. The gradient of $G$ w.r.t. the conic matrix $\mathbf{C}_{2D}$ is:
    $$ \frac{\partial G}{\partial C_{xx}} = \text{ellipsepow} \cdot \frac{1}{2} d_x^2 $$
    $$ \frac{\partial G}{\partial C_{xy}} = \text{ellipsepow} \cdot d_x d_y $$
    $$ \frac{\partial G}{\partial C_{zz}} = \text{ellipsepow} \cdot \frac{1}{2} d_y^2 $$
    Note: The factor of $1/2$ arises because $C_{xy}$ appears twice in the quadratic form, or equivalently, the gradient w.r.t. the symmetric matrix entry requires care. The source code uses `halfellipsepow` for diagonal terms and full `ellipsepow` for off-diagonal accumulation in specific ways. Specifically, the source updates `dL_dconic2D` using `halfellipsepow * d.x * d.x` for the x-component, `halfellipsepow * d.y * d.y` for the z-component, and `ellipsepow * d.x * d.y` for the y-component (which represents the off-diagonal term in the symmetric matrix storage) [S1].

**Pseudocode (Backward Pass):**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def inverse_multiquadric_splat_backward(conic_matrix, opacity, d_x, d_y, dL_dalpha):
    """
    Returns gradients: dL_dmean_x, dL_dmean_y, dL_dconic_xx, dL_dconic_xy, dL_dconic_zz
    """
    C_xx, C_xy, C_zz = conic_matrix
    
    ellipse = C_xx * d_x * d_x + C_zz * d_y * d_y + 2.0 * C_xy * d_x * d_y
    
    # Gradient w.r.t. G (weight)
    dL_dG = opacity * dL_dalpha
    
    # Gradient w.r.t. ellipse
    ellipsepow = -2.0 * dL_dG / ((ellipse + 1.0) ** 2)
    halfellipsepow = 0.5 * ellipsepow
    
    # Gradients w.r.t. offset d_x, d_y
    dG_ddx = ellipsepow * (C_xx * d_x + C_xy * d_y)
    dG_ddy = ellipsepow * (C_xy * d_x + C_zz * d_y)
    
    # Gradients w.r.t. mean (negative of gradient w.r.t. offset)
    dL_dmean_x = -dG_ddx
    dL_dmean_y = -dG_ddy
    
    # Gradients w.r.t. conic matrix
    # dL/dC_xx = dL/dG * dG/dC_xx = dL_dG * (halfellipsepow * d_x^2 / dL_dG) ? 
    # Actually, dG/dC_xx = -2 * w^2 * d_x^2 / 2 ? 
    # From source: dL_dconic2D.x += halfellipsepow * d.x * d.x
    # Note: halfellipsepow already contains dL_dG factor? 
    # In source: ellipsepow = - dL_dG * 2 / (ellipse+1)^2. 
    # dG/dC_xx = -2 * (1/(ellipse+1)^2) * d_x^2. 
    # So dL/dC_xx = dL_dG * (-2 * d_x^2 / (ellipse+1)^2) = ellipsepow * d_x^2.
    # But source uses halfellipsepow for diagonal. 
    # Let's follow source logic exactly:
    
    dL_dconic_xx = halfellipsepow * d_x * d_x
    dL_dconic_xy = ellipsepow * d_x * d_y
    dL_dconic_zz = halfellipsepow * d_y * d_y
    
    return dL_dmean_x, dL_dmean_y, dL_dconic_xx, dL_dconic_xy, dL_dconic_zz
```

## Operating characteristics
*   **Time Complexity:** Same as standard splatting, $O(N)$ per pixel for $N$ splats, with a constant factor increase due to the rational function evaluation.
*   **Memory Complexity:** $O(1)$ additional memory per splat.
*   **Strengths:** Provides a different decay profile that can reduce over-saturation in high-opacity regions compared to the Gaussian kernel.
*   **Limitations:** The rational function may behave differently in the tails, potentially affecting the smoothness of the rendered image. The correction factor $1.38$ is specific to this method and may require tuning for different scenes.
*   **When to use:** When standard Gaussian splatting produces artifacts due to high opacity or when a different radial basis function is desired for specific visual effects.

## Background
Inverse Multiquadric Splatting was introduced in the paper "DARB-Splatting: Generalizing Splatting with Decaying Anisotropic Radial Basis Functions" [S1]. It is part of a family of methods that generalize the standard Gaussian splatting by using different radial basis functions. The method modifies the forward and backward propagation of the CUDA rasterizer to support the inverse multiquadric function.

## Sources
[S1] DARB-Splatting: Generalizing Splatting with Decaying Anisotropic Radial Basis Functions, https://doi.org/10.1109/WACV61042.2026.00438, fetched 2026-10-30
