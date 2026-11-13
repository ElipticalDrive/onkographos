# Inverse Multiquadric Splatting

Also called: 3DIQS, Inverse Multiquadric

## Overview
Inverse Multiquadric Splatting replaces the standard Gaussian radial basis function with an inverse multiquadric function to alter the decay profile of splat contributions. It solves the problem of Gaussian tails extending too far or decaying too slowly, which can cause visual artifacts and inefficient memory usage in large scenes. It sets itself apart from standard 3D Gaussian Splatting by using a rational function rather than an exponential, and from [Opacity-Independent Gaussian Truncation](10266-opacity-independent-gaussian-truncation.md) by retaining opacity weighting but modifying the spatial decay kernel.

## How it works
**Inputs:**
*   3D Gaussian primitive defined by mean $\mu$, covariance $\Sigma$ (parameterized by scale and rotation), opacity $\omega$, and spherical harmonic coefficients.
*   Camera projection matrix.
*   Pixel coordinates $(x, y)$.

**Outputs:**
*   Alpha value $\alpha$ for the pixel.
*   Gradients $\frac{\partial L}{\partial \mu}$, $\frac{\partial L}{\partial \Sigma}$, $\frac{\partial L}{\partial \omega}$.

**Data Structures:**
*   `con_o`: The 2D conic matrix (inverse of the projected 2D covariance $\Sigma'_{2 \times 2}$) and opacity weight. Layout: `x` ($\Sigma'^{-1}_{00}$), `y` ($\Sigma'^{-1}_{01}$), `z` ($\Sigma'^{-1}_{11}$), `w` ($\omega$).
*   `d`: 2D displacement vector $(x - \mu'_x, y - \mu'_y)$.

**Parameters:**
*   `correctionFactor`: A scalar multiplier applied to the covariance matrices to adjust the effective width. Value: $1.38$ [S1].
*   `ellipse_max`: The upper bound for the ellipse term to cull distant splats. Value: $9.0$ [S1].

**Steps:**

1.  **Covariance Correction:**
    Multiply the 3D covariance matrix $\Sigma$ and the projected 2D covariance matrix $\Sigma'_{2 \times 2}$ by `correctionFactor` ($1.38$) [S1].
    $$ \Sigma \leftarrow \Sigma \times 1.38 $$
    $$ \Sigma'_{2 \times 2} \leftarrow \Sigma'_{2 \times 2} \times 1.38 $$
    [G] This scaling effectively widens the Gaussian support to compensate for the faster decay of the inverse multiquadric function.

2.  **Compute Conic Matrix:**
    Compute the inverse of the corrected 2D covariance matrix to obtain the conic matrix entries `con_o.x`, `con_o.y`, `con_o.z` [S1].

3.  **Compute Ellipse Term:**
    Calculate the squared Mahalanobis distance (ellipse term) using the conic matrix and the 2D displacement $d = (d.x, d.y)$:
    $$ \text{ellipse} = \text{con\_o.x} \cdot d.x^2 + \text{con\_o.z} \cdot d.y^2 + 2 \cdot \text{con\_o.y} \cdot d.x \cdot d.y $$
    [S1]

4.  **Culling:**
    If $\text{ellipse} \le 0$ or $\text{ellipse} \ge 9.0$, skip the splat (continue) [S1].
    [G] The lower bound check handles degenerate cases or numerical noise where the distance might be non-positive due to precision errors. The upper bound limits the computational cost and visual influence of distant splats.

5.  **Compute Weight and Alpha:**
    Calculate the inverse multiquadric weight $w$ (denoted as $G$ in the source code):
    $$ G = \frac{1}{\text{ellipse} + 1} $$
    [S1]
    Compute the final alpha:
    $$ \alpha = \min(0.99, \omega \cdot G) $$
    [S1]

6.  **Backward Pass (Gradients):**
    Compute gradients with respect to the loss $L$.
    
    a. Gradient w.r.t $G$:
    $$ \frac{\partial L}{\partial G} = \omega \cdot \frac{\partial L}{\partial \alpha} $$
    [S1]
    
    b. Gradient w.r.t Ellipse:
    $$ \frac{\partial L}{\partial \text{ellipse}} = \frac{\partial L}{\partial G} \cdot \frac{-1}{(\text{ellipse} + 1)^2} $$
    [G] Derived from $\frac{\partial G}{\partial \text{ellipse}} = \frac{-1}{(\text{ellipse} + 1)^2}$.
    
    c. Gradient w.r.t Mean ($\mu'$):
    The gradient flows through the ellipse term. Let $dL_{d\delta x}$ and $dL_{d\delta y}$ be the partial derivatives of the loss with respect to the displacement components $d.x$ and $d.y$.
    $$ \frac{\partial L}{\partial d.x} = \frac{\partial L}{\partial \text{ellipse}} \cdot \frac{\partial \text{ellipse}}{\partial d.x} = \frac{\partial L}{\partial \text{ellipse}} \cdot (2 \cdot \text{con\_o.x} \cdot d.x + 2 \cdot \text{con\_o.y} \cdot d.y) $$
    $$ \frac{\partial L}{\partial d.y} = \frac{\partial L}{\partial \text{ellipse}} \cdot \frac{\partial \text{ellipse}}{\partial d.y} = \frac{\partial L}{\partial \text{ellipse}} \cdot (2 \cdot \text{con\_o.z} \cdot d.y + 2 \cdot \text{con\_o.y} \cdot d.x) $$
    [G] The source code computes `dG_ddelx` and `dG_ddely` which correspond to $\frac{\partial G}{\partial d.x}$ and $\frac{\partial G}{\partial d.y}$. Specifically:
    $$ \frac{\partial G}{\partial d.x} = \frac{-2}{(\text{ellipse} + 1)^2} (\text{con\_o.x} \cdot d.x + \text{con\_o.y} \cdot d.y) $$
    $$ \frac{\partial G}{\partial d.y} = \frac{-2}{(\text{ellipse} + 1)^2} (\text{con\_o.y} \cdot d.x + \text{con\_o.z} \cdot d.y) $$
    Then $\frac{\partial L}{\partial d.x} = \frac{\partial L}{\partial G} \cdot \frac{\partial G}{\partial d.x}$.
    
    d. Gradient w.r.t Conic Matrix ($\Sigma'^{-1}$):
    $$ \frac{\partial L}{\partial \text{con\_o.x}} = \frac{\partial L}{\partial \text{ellipse}} \cdot d.x^2 $$
    $$ \frac{\partial L}{\partial \text{con\_o.y}} = \frac{\partial L}{\partial \text{ellipse}} \cdot 2 \cdot d.x \cdot d.y $$
    $$ \frac{\partial L}{\partial \text{con\_o.z}} = \frac{\partial L}{\partial \text{ellipse}} \cdot d.y^2 $$
    [G] Derived from $\frac{\partial \text{ellipse}}{\partial \text{con\_o.x}} = d.x^2$, etc.
    
    e. Gradient w.r.t Opacity ($\omega$):
    $$ \frac{\partial L}{\partial \omega} = G \cdot \frac{\partial L}{\partial \alpha} $$
    [S1]

    f. Gradient w.r.t 3D Covariance:
    The gradients w.r.t the 2D conic matrix entries must be scaled by `correctionFactor` because the 2D covariance was multiplied by it.
    $$ \frac{\partial L}{\partial \Sigma'_{2 \times 2, ij}} = \frac{\partial L}{\partial \Sigma'^{-1}_{2 \times 2, ij}} \cdot \text{correctionFactor} $$
    [S1]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def inverse_multiquadric_splat_forward(con_o, d, omega):
    """
    con_o: tuple (x, y, z, w) representing the 2D conic matrix and opacity.
    d: tuple (dx, dy) representing the 2D displacement from mean.
    omega: float, the opacity weight.
    Returns: alpha, ellipse
    """
    correction_factor = 1.38
    # Note: In actual implementation, cov2D is scaled by correction_factor before conic computation.
    # Here we assume con_o is already derived from the scaled covariance.
    
    # Compute ellipse term: d^T * Sigma_inv * d
    ellipse = con_o.x * d.dx * d.dx + con_o.z * d.dy * d.dy + 2.0 * con_o.y * d.dx * d.dy
    
    # Culling
    if ellipse <= 0.0 or ellipse >= 9.0:
        return None, ellipse
    
    # Compute weight G (inverse multiquadric)
    G = 1.0 / (ellipse + 1.0)
    
    # Compute alpha
    alpha = min(0.99, omega * G)
    
    return alpha, ellipse

def inverse_multiquadric_splat_backward(dL_dalpha, con_o, d, omega, ellipse):
    """
    dL_dalpha: gradient of loss w.r.t alpha.
    con_o: tuple (x, y, z, w).
    d: tuple (dx, dy).
    omega: float.
    ellipse: float, the computed ellipse term from forward pass.
    Returns: gradients for mean, conic, and opacity.
    """
    if ellipse is None:
        return None, None, None

    correction_factor = 1.38
    
    # Gradient w.r.t G
    dL_dG = omega * dL_dalpha
    
    # Gradient w.r.t ellipse
    # dG/dellipse = -1 / (ellipse + 1)^2
    dL_dellipse = dL_dG * (-1.0 / ((ellipse + 1.0) ** 2))
    
    # Gradient w.r.t mean (via d.x and d.y)
    # dG/ddx = -2 * (con_o.x * d.x + con_o.y * d.y) / (ellipse + 1)^2
    # dG/ddy = -2 * (con_o.y * d.x + con_o.z * d.y) / (ellipse + 1)^2
    
    denom_sq = (ellipse + 1.0) ** 2
    dG_ddx = -2.0 * (con_o.x * d.dx + con_o.y * d.dy) / denom_sq
    dG_ddy = -2.0 * (con_o.y * d.dx + con_o.z * d.dy) / denom_sq
    
    dL_ddx = dL_dG * dG_ddx
    dL_ddy = dL_dG * dG_ddy
    
    # Gradient w.r.t conic matrix entries
    # dL/dcon_o.x = dL_dellipse * d.x^2
    # dL/dcon_o.y = dL_dellipse * 2 * d.x * d.y
    # dL/dcon_o.z = dL_dellipse * d.y^2
    
    dL_dconic_x = dL_dellipse * d.dx * d.dx
    dL_dconic_y = dL_dellipse * 2.0 * d.dx * d.dy
    dL_dconic_z = dL_dellipse * d.dy * d.dy
    
    # Scale conic gradients by correction factor
    dL_dconic_x *= correction_factor
    dL_dconic_y *= correction_factor
    dL_dconic_z *= correction_factor
    
    # Gradient w.r.t opacity
    dL_domega = (1.0 / (ellipse + 1.0)) * dL_dalpha
    
    return (dL_ddx, dL_ddy), (dL_dconic_x, dL_dconic_y, dL_dconic_z), dL_domega
```

## Operating characteristics
*   **Time Complexity:** Same as standard Gaussian splatting per pixel, $O(1)$ per primitive, but with slightly more complex arithmetic (division vs exponential).
*   **Memory Complexity:** Same as standard 3DGS.
*   **Performance:** The inverse multiquadric function decays slower than the Gaussian for small distances but faster for large distances (depending on the correction factor). This can lead to fewer Gaussians needed to cover a surface, potentially reducing memory usage. However, the rational function evaluation is more expensive than the exponential.
*   **Strengths:** Better control over the tail behavior of the splats. Can reduce artifacts caused by long Gaussian tails.
*   **Limitations:** The correction factor $1.38$ is specific to this implementation and may need tuning for different scenes. The culling threshold of $9.0$ is fixed.
*   **When to use:** When standard Gaussian splatting produces artifacts due to excessive tail overlap or when memory is constrained and fewer, wider splats are beneficial.

## Background
Inverse Multiquadric Splatting was introduced in the DARB-Splatting paper (2026) as a generalization of splatting using decaying anisotropic radial basis functions. It replaces the Gaussian RBF with an inverse multiquadric RBF to provide different decay characteristics.

## Sources
[S1] DARB-Splatting: Generalizing Splatting with Decaying Anisotropic Radial Basis Functions, https://doi.org/10.1109/WACV61042.2026.00438, fetched 2026-10-30
