# Inverse Multiquadric Splatting

Also called: 3DIQS, Inverse Multiquadric

## Overview
Inverse Multiquadric Splatting is a differentiable rendering technique that replaces the standard Gaussian radial basis function with an inverse multiquadric function to define the spatial decay of a 3D Gaussian primitive's contribution to a pixel [S1]. It differs from standard Gaussian Splatting by using a heavier-tailed decay profile, which requires a specific correction factor applied to the covariance matrix to maintain visual fidelity and training stability [S1].

## How it works
**Inputs:**
*   3D Gaussian parameters: mean $\mu$, 3D covariance $\Sigma$ (parameterized by scale and rotation), and opacity $\omega$.
*   Camera projection matrix and Jacobian $J$ for screen-space projection.
*   Pixel coordinates $p$ within the splat's bounding box.

**Outputs:**
*   Alpha value $\alpha$ for the pixel.
*   Gradients $\frac{\partial L}{\partial \mu}$, $\frac{\partial L}{\partial \Sigma}$, and $\frac{\partial L}{\partial \omega}$ for backpropagation.

**Data Structures:**
*   `conic` (or `con_o`): A 4-element vector representing the inverse 2D covariance matrix (conic matrix) $C = \Sigma'^{-1}$ and the opacity weight. Layout: `[x, y, z, w]` corresponding to $\begin{bmatrix} C_{11} & C_{12} \\ C_{12} & C_{22} \end{bmatrix}$ and $\omega$.
*   `d`: 2D displacement vector $p - \mu_{2D}$.

**Parameters:**
*   `correctionFactor`: A scalar multiplier applied to the covariance matrices. Value: $1.38$ [S1].
*   `ellipse_threshold`: The upper bound for culling. Value: $9.0$ [S1].
*   `alpha_max`: The maximum clamped alpha value. Value: $0.99$ [S1].

**Steps:**

1.  **Covariance Correction:** Multiply the 3D covariance matrix $\Sigma$ by `correctionFactor` ($1.38$) before projection [S1].
2.  **Projection:** Project the corrected 3D Gaussian to 2D screen space to obtain the 2D mean $\mu_{2D}$ and 2D covariance $\Sigma'$. Compute the conic matrix $C = \Sigma'^{-1}$ [S1].
3.  **Culling:** Calculate the ellipse term (squared Mahalanobis distance):
    $$ \text{ellipse} = d^T C d = C_{11} d_x^2 + C_{22} d_y^2 + 2 C_{12} d_x d_y $$
    If $\text{ellipse} \le 0$ or $\text{ellipse} \ge 9.0$, skip the splat [S1].
4.  **Alpha Computation:** Compute the inverse multiquadric weight $w$ and the final alpha $\alpha$:
    $$ w = \frac{1}{\text{ellipse} + 1} $$
    $$ \alpha = \min(0.99, \omega \cdot w) $$
    [S1]

**Pseudocode (Forward Pass):**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def inverse_multiquadric_splat_forward(conic, opacity, d_x, d_y):
    """
    conic: tuple (C11, C12, C22, w_opacity)
    opacity: float (omega)
    d_x, d_y: float (displacement from mean)
    """
    C11, C12, C22, w_opacity = conic
    
    # Calculate ellipse term (Mahalanobis distance squared)
    ellipse = C11 * d_x * d_x + C22 * d_y * d_y + 2.0 * C12 * d_x * d_y
    
    # Culling
    if ellipse <= 0.0 or ellipse >= 9.0:
        return None, None
        
    # Compute inverse multiquadric weight
    # w = 1 / (ellipse + 1)
    inv_quad = 1.0 / (ellipse + 1.0)
    
    # Compute alpha
    alpha = min(0.99, w_opacity * inv_quad)
    
    return alpha, inv_quad
```

**Pseudocode (Backward Pass):**

The gradient of the loss $L$ with respect to the inverse quadric term $G$ (where $G = \frac{1}{\text{ellipse} + 1}$) is first computed from the gradient w.r.t alpha:
$$ \frac{\partial L}{\partial G} = \omega \cdot \frac{\partial L}{\partial \alpha} $$
[S1]

The gradients with respect to the displacement components $d_x$ and $d_y$ are:
$$ \frac{\partial L}{\partial d_x} = \frac{\partial L}{\partial G} \cdot \frac{-2}{(\text{ellipse} + 1)^2} \cdot (C_{11} d_x + C_{12} d_y) $$
$$ \frac{\partial L}{\partial d_y} = \frac{\partial L}{\partial G} \cdot \frac{-2}{(\text{ellipse} + 1)^2} \cdot (C_{12} d_x + C_{22} d_y) $$
[S1]

The gradients with respect to the conic matrix entries are derived from the chain rule through the ellipse term. Let $E = \text{ellipse} + 1$.
$$ \frac{\partial L}{\partial C_{11}} = \frac{\partial L}{\partial G} \cdot \frac{-2}{E^2} \cdot d_x^2 $$
$$ \frac{\partial L}{\partial C_{22}} = \frac{\partial L}{\partial G} \cdot \frac{-2}{E^2} \cdot d_y^2 $$
$$ \frac{\partial L}{\partial C_{12}} = \frac{\partial L}{\partial G} \cdot \frac{-2}{E^2} \cdot 2 d_x d_y $$
[S1]

Note: The `correctionFactor` ($1.38$) must also be applied to the gradients of the 3D covariance matrix during the backpropagation step from 2D conic to 3D covariance, as the forward pass scaled the covariance by this factor [S1].

**Pseudocode (Backward Pass):**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def inverse_multiquadric_splat_backward(conic, opacity, d_x, d_y, dL_dalpha):
    """
    conic: tuple (C11, C12, C22, w_opacity)
    opacity: float (omega)
    d_x, d_y: float (displacement from mean)
    dL_dalpha: float (gradient of loss w.r.t alpha)
    """
    C11, C12, C22, w_opacity = conic
    
    ellipse = C11 * d_x * d_x + C22 * d_y * d_y + 2.0 * C12 * d_x * d_y
    
    # Precompute common terms
    E = ellipse + 1.0
    dL_dG = w_opacity * dL_dalpha
    ellipse_pow_term = -2.0 * dL_dG / (E * E)
    
    # Gradients w.r.t displacement (d_x, d_y)
    # These are used to update the 2D mean
    dL_ddx = ellipse_pow_term * (C11 * d_x + C12 * d_y)
    dL_ddy = ellipse_pow_term * (C12 * d_x + C22 * d_y)
    
    # Gradients w.r.t conic matrix entries
    # dL/dC11
    dL_dC11 = ellipse_pow_term * d_x * d_x
    # dL/dC22
    dL_dC22 = ellipse_pow_term * d_y * d_y
    # dL/dC12 (note: ellipse has 2*C12, so gradient is doubled relative to C12 alone if treating C12 as independent, 
    # but here we follow the source structure which updates the symmetric matrix entries directly)
    # Source uses: halfellipsepow * d.x * d.y for off-diagonal accumulation in atomicAdd context
    # The formula derived from source code:
    # dG/dC12 = -2/(E^2) * 2 * d_x * d_y
    # So dL/dC12 = dL_dG * (-4 * d_x * d_y / E^2)
    # However, source code snippet 29-31 shows:
    # atomicAdd(&dL_dconic2D[global_id].y, halfellipsepow * d.x * d.y)
    # where halfellipsepow = 0.5 * ellipsepow
    # ellipsepow = -dL_dG * 2 / E^2
    # So contribution is (-dL_dG * 2 / E^2) * 0.5 * d_x * d_y = -dL_dG * d_x * d_y / E^2
    # This implies the source code treats the off-diagonal term accumulation carefully.
    # We will stick to the explicit formula derivation from Eq 31/32 in S1 for clarity, 
    # but note the code implementation details.
    
    # Explicit gradients for conic matrix (inverse covariance)
    # dL/dC11
    grad_C11 = -2.0 * dL_dG * d_x * d_x / (E * E)
    # dL/dC22
    grad_C22 = -2.0 * dL_dG * d_y * d_y / (E * E)
    # dL/dC12
    grad_C12 = -4.0 * dL_dG * d_x * d_y / (E * E)
    
    return dL_ddx, dL_ddy, grad_C11, grad_C12, grad_C22
```

## Operating characteristics
*   **Complexity:** Same asymptotic time and memory complexity as standard Gaussian Splatting per pixel, as it replaces only the radial decay function evaluation.
*   **Performance:** The inverse multiquadric function involves a division and addition, similar in cost to the Gaussian exponential, but avoids the `exp` function call, potentially offering slight performance gains or parity depending on hardware [S1].
*   **Strengths:** The heavier tails of the inverse multiquadric function can provide better coverage for large or anisotropic primitives, potentially reducing artifacts in regions with sparse sampling or large depth variations [S1].
*   **Limitations:** Requires the specific correction factor ($1.38$) to match the visual density of standard Gaussians; without it, the splats may appear too diffuse or too sharp [S1].

## Background
Inverse Multiquadric Splatting was introduced in the DARB-Splatting paper, which generalizes splatting using decaying anisotropic radial basis functions [S1]. It is one of several alternative radial basis functions explored to improve the flexibility of 3D Gaussian Splatting beyond the standard Gaussian kernel [S1].

## Sources
[S1] DARB-Splatting: Generalizing Splatting with Decaying Anisotropic Radial Basis Functions, https://doi.org/10.1109/WACV61042.2026.00438, fetched 2026-10-30
