# Half-cosine Squared Splatting

Also called: 3DHCS, Half Cosine Squared

## Overview
Half-cosine Squared Splatting is a radial basis function (RBF) technique for 3D Gaussian Splatting that replaces the standard Gaussian decay with a half-cosine squared function [S1]. It solves the problem of slow convergence and high memory usage in standard splatting by providing higher opacity values per primitive, allowing fewer primitives to achieve the same accumulated opacity [S1]. It sets itself apart from standard Gaussian Splatting by using a bounded, flatter decay profile that accelerates training and reduces memory footprint, while differing from its sibling [Non-Tiled 2D Gaussian Splatting with Transmittance Culling](2067-non-tiled-2d-gaussian-splatting-with-transmittance-culling.md) by fundamentally altering the splat shape rather than just the culling strategy.

## How it works
**Inputs:** 3D Gaussian parameters (mean $\mu$, 3D covariance $\Sigma$, opacity $\alpha$, color $C$); Camera projection matrix; Image resolution.
**Outputs:** Final pixel color and alpha.

**Data Structures:**
*   `cov`: 3x3 symmetric 3D covariance matrix.
*   `cov2D`: 2x2 symmetric 2D covariance matrix (projected).
*   `conic`: 2x2 symmetric conic matrix (inverse of `cov2D`), stored as 3 floats $(a, b, c)$ for the upper triangle.
*   `mu_prime`: 2D mean in screen space.
*   `ellipse`: Scalar value representing the squared Mahalanobis distance scaled by the ellipse factor.

**Parameters:**
*   `correctionFactor`: 1.36 [S1].
*   $\beta$: 2 [S1].
*   $\xi$: 36 [S1].
*   `ellipseLimit`: 9.0 [S1].
*   `alphaClip`: 0.99 [S1].

**Steps:**

1.  **Covariance Correction:** Multiply the 3D covariance matrix $\Sigma$ by `correctionFactor` before projection [S1].
    $$ \Sigma_{corrected} = \Sigma \times 1.36 $$
2.  **Projection:** Project the 3D Gaussian to 2D screen space to obtain the 2D mean $\mu'$ and the 2D covariance $\Sigma'_{2 \times 2}$ [S1].
3.  **Conic Calculation:** Compute the inverse of the 2D covariance to get the conic matrix $J = (\Sigma'_{2 \times 2})^{-1}$ [S1].
4.  **Ellipse Factor Calculation:** For a pixel at screen coordinate $x$, calculate the squared Mahalanobis distance term $d_M^2$ (referred to as `ellipse` in code) [S1]:
    $$ d_M^2 = (x - \mu')^T (\Sigma'_{2 \times 2})^{-1} (x - \mu') $$
    In the implementation, this is computed using the conic matrix components $(a, b, c)$ where $J = \begin{bmatrix} a & b \\ b & c \end{bmatrix}$:
    $$ d_M^2 = a \cdot dx^2 + c \cdot dy^2 + 2 \cdot b \cdot dx \cdot dy $$
    where $dx = x - \mu'_x$ and $dy = x - \mu'_y$ [S1].
5.  **Boundary Check:** If $d_M^2 < 0$ or $d_M^2 > 9$, skip the splat for this pixel [S1]. This limits the 2D ellipse to restrict the half-cosine pulse [S1].
6.  **Weight Calculation:** Calculate the half-cosine squared weight $w$ [S1]:
    $$ ellipseFactor = \frac{2 \pi \cdot d_M^2}{36} = 0.175 \cdot d_M^2 $$
    $$ w = \cos(ellipseFactor) $$
    The condition for validity is $\frac{d_M^2}{\xi} \leq \frac{\pi}{2}$, which corresponds to $d_M^2 \leq 18$. However, the code explicitly clamps the check to 9.0 [S1].
7.  **Alpha Computation:** Compute the final alpha $\alpha$ [S1]:
    $$ \alpha = \min(0.99, \alpha_{primitive} \cdot w) $$
    where $\alpha_{primitive}$ is the primitive's base opacity.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def half_cosine_squared_splat(x_screen, mu_prime, conic_matrix, primitive_opacity):
    """
    x_screen: tuple (x, y)
    mu_prime: tuple (mu_x, mu_y)
    conic_matrix: tuple (a, b, c) representing [[a, b], [b, c]]
    primitive_opacity: float
    """
    dx = x_screen[0] - mu_prime[0]
    dy = x_screen[1] - mu_prime[1]
    
    a, b, c = conic_matrix
    
    # Calculate squared Mahalanobis distance (ellipse)
    # Formula: a*dx^2 + c*dy^2 + 2*b*dx*dy
    ellipse = a * dx * dx + c * dy * dy + 2.0 * b * dx * dy
    
    # Boundary check
    if ellipse < 0.0 or ellipse > 9.0:
        return 0.0, 0.0
        
    # Calculate ellipse factor
    # 0.175 is approximately 2*pi/36
    ellipse_factor = 0.175 * ellipse
    
    # Calculate cosine weight
    cosine_weight = math.cos(ellipse_factor)
    
    # Calculate final alpha
    alpha = min(0.99, primitive_opacity * cosine_weight)
    
    return alpha, cosine_weight

def backward_step_half_cosine(ellipse, dL_dalpha, primitive_opacity, conic_matrix, dx, dy):
    """
    Computes gradients for the half-cosine squared splat.
    dL_dalpha: gradient of loss w.r.t alpha
    """
    # Gradient w.r.t opacity
    # alpha = min(0.99, opacity * cos(ellipse_factor))
    # Assuming we are in the valid range where alpha < 0.99
    cosine_weight = math.cos(0.175 * ellipse)
    
    # dL/d_opacity = dL/d_alpha * d_alpha/d_opacity
    dL_d_opacity = dL_dalpha * cosine_weight
    
    # Gradient w.r.t conic matrix entries (a, b, c)
    # w = cos(0.175 * ellipse)
    # dw/d_ellipse = -0.175 * sin(0.175 * ellipse)
    # d_ellipse/d_conic = partial derivatives of (a*dx^2 + c*dy^2 + 2*b*dx*dy)
    
    sin_term = math.sin(0.175 * ellipse)
    dL_d_ellipse = dL_dalpha * primitive_opacity * (-0.175 * sin_term)
    
    # d_ellipse/da = dx^2
    # d_ellipse/dc = dy^2
    # d_ellipse/db = 2*dx*dy
    
    dL_da = dL_d_ellipse * (dx * dx)
    dL_dc = dL_d_ellipse * (dy * dy)
    dL_db = dL_d_ellipse * (2.0 * dx * dy)
    
    return dL_d_opacity, dL_da, dL_db, dL_dc
```

**Backpropagation Details:**
The derivatives required for training are [S1]:
$$ \frac{dw}{d(x - \mu')} = \frac{-2(\Sigma'_{2 \times 2})^{-1} (x - \mu') \sin\left(\frac{(x - \mu')^T (\Sigma'_{2 \times 2})^{-1} (x - \mu')}{\xi}\right)}{\xi} $$
$$ \frac{dw}{d((\Sigma'_{2 \times 2})^{-1})} = \frac{-(x - \mu') (x - \mu')^T \sin\left(\frac{(x - \mu')^T (\Sigma'_{2 \times 2})^{-1} (x - \mu')}{\xi}\right)}{\xi} $$
The gradient w.r.t the conic matrix entries must account for the `correctionFactor` if the conic is derived from the corrected covariance [S1].

## Operating characteristics
*   **Time Complexity:** Similar to standard Gaussian Splatting per primitive, but the cosine calculation is slightly more expensive than the exponential Gaussian decay [S1]. However, overall training time is reduced because fewer primitives are needed [S1].
*   **Memory Usage:** Reduced compared to standard Gaussian Splatting. Half-cosine primitives provide higher opacity values, requiring fewer primitives to achieve the same accumulated opacity coverage [S1].
*   **Performance:** Demonstrates superior performance in training loss curves and speed compared to Gaussians, despite similar final training loss [S1].
*   **Strengths:** Faster training, lower memory usage, higher opacity per primitive.
*   **Limitations:** The bounded nature of the cosine function requires careful boundary checking (ellipse < 9) to avoid invalid values [S1]. The computation of trigonometric functions may be slower than simple exponentials on some hardware, though this is offset by the reduced primitive count [S1].

## Background
Half-cosine Squared Splatting was introduced in the DARB-Splatting paper as a generalization of splatting using decaying anisotropic radial basis functions [S1]. It was developed to address the limitations of exponentially decaying functions (like Gaussians) which require many primitives for accurate blending [S1]. The method uses $\beta=2$ and $\xi=36$ as key parameters for the half-cosine square function [S1].

## Sources
[S1] DARB-Splatting: Generalizing Splatting with Decaying Anisotropic Radial Basis Functions, https://doi.org/10.1109/WACV61042.2026.00438, fetched 2026-10-30
