# Inverse Multiquadric Splatting

Also called: 3DIQS, Inverse Multiquadric

## Overview
Inverse Multiquadric Splatting is a rendering technique that replaces the standard Gaussian radial basis function with an inverse multiquadric function to define the spatial response of 3D Gaussian primitives. It solves the problem of artifact generation in regions with high curvature or sharp edges by providing a different decay profile than the Gaussian kernel. It differs from its sibling [Opacity-Independent Gaussian Truncation](10266-opacity-independent-gaussian-truncation.md) by altering the kernel shape itself rather than just the truncation threshold, and from [Directed Distance Field (DDF) for Gaussian Splatting](12854-directed-distance-field-ddf-for-gaussian-splatting.md) by maintaining a scalar field splatting approach rather than a signed distance field representation.

## How it works
**Inputs and Outputs:**
*   **Input:** 3D Gaussian primitive $g$ with mean $\mu$, 3D covariance $\Sigma$, opacity $\omega$, and color $c$. Projected 2D mean $\mu'$ and 2D conic matrix (inverse covariance) $\Sigma'^{-1}$ (denoted as `con_o` in implementation).
*   **Output:** Alpha value $\alpha$ for the pixel, and gradients $\frac{\partial L}{\partial \mu'}$, $\frac{\partial L}{\partial \Sigma'^{-1}}$, and $\frac{\partial L}{\partial \omega}$ for backpropagation.

**Data Structures:**
The 2D conic matrix `con_o` is a symmetric $2 \times 2$ matrix stored as a vector `con_o = (x, y, z, w)` where:
*   `con_o.x` = $\Sigma'^{-1}_{00}$
*   `con_o.y` = $\Sigma'^{-1}_{01}$
*   `con_o.z` = $\Sigma'^{-1}_{11}$
*   `con_o.w` = Opacity $\omega$

**Parameters:**
*   **Correction Factor:** A scalar multiplier applied to the covariance matrices to adjust the effective spread. Value: $1.38$ [S1].
*   **Ellipse Bounds:** The 2D ellipse metric is clamped/culled between $0$ and $9$ [S1].

**Formulas:**
The 3D inverse multiquadric function is defined as [S1]:
$$ f(d_M) = \frac{1}{\frac{1}{\xi} d_M^2 + 1} $$
where $d_M$ is the Mahalanobis distance. In the 2D projection, the splat weight $w$ is:
$$ w = \frac{1}{\frac{1}{\xi} (\mathbf{x} - \mu')^T (\Sigma'_{2 \times 2})^{-1} (\mathbf{x} - \mu') + 1} $$
Let $E$ be the quadratic form (ellipse metric):
$$ E = (\mathbf{x} - \mu')^T (\Sigma'_{2 \times 2})^{-1} (\mathbf{x} - \mu') $$
The final alpha is:
$$ \alpha = \min(0.99, \omega \cdot \frac{1}{E + 1}) $$
Note: The source code applies a correction factor to the covariance matrices, which effectively scales the $E$ term or the resulting weight. The specific implementation in [S1] uses `cov *= 1.38f` before projection, which modifies the resulting 2D conic matrix. The gradient derivation assumes the form $\frac{1}{E+1}$.

**Steps:**

1.  **Covariance Correction:** Multiply the 3D covariance matrix $\Sigma$ by the correction factor $1.38$ [S1].
2.  **Projection:** Compute the 2D projected covariance $\Sigma'_{2 \times 2}$ and its inverse (conic matrix) `con_o` using standard projective geometry [G].
3.  **Correction on 2D:** Multiply the 2D conic matrix entries by the correction factor $1.38$ [S1].
4.  **Ellipse Calculation:** For a pixel at $\mathbf{x}$, compute the displacement $\mathbf{d} = \mathbf{x} - \mu'$. Compute the ellipse metric $E$:
    $$ E = \text{con\_o}.x \cdot d_x^2 + \text{con\_o}.z \cdot d_y^2 + 2 \cdot \text{con\_o}.y \cdot d_x \cdot d_y $$
5.  **Culling:**
    *   If $E \le 0$, skip the splat (continue) [S1].
    *   If $E \ge 9$, skip the splat (continue) [S1].
6.  **Alpha Computation:**
    $$ G = \frac{1}{E + 1} $$
    $$ \alpha = \min(0.99, \text{con\_o}.w \cdot G) $$
7.  **Backward Pass (Gradients):**
    *   Compute gradient w.r.t $G$: $dL_dG = \text{con\_o}.w \cdot \frac{\partial L}{\partial \alpha}$ [S1].
    *   Compute gradient w.r.t $E$:
        $$ \frac{\partial L}{\partial E} = dL_dG \cdot \frac{-1}{(E + 1)^2} $$
        Let $H = \frac{-2}{(E + 1)^2} \cdot dL_dG$ [S1].
    *   Gradient w.r.t mean $\mu'$ (via $E$):
        $$ \frac{\partial E}{\partial d_x} = 2 \cdot (\text{con\_o}.x \cdot d_x + \text{con\_o}.y \cdot d_y) $$
        $$ \frac{\partial E}{\partial d_y} = 2 \cdot (\text{con\_o}.y \cdot d_x + \text{con\_o}.z \cdot d_y) $$
        $$ \frac{\partial L}{\partial d_x} = \frac{\partial L}{\partial E} \cdot \frac{\partial E}{\partial d_x} $$
        $$ \frac{\partial L}{\partial d_y} = \frac{\partial L}{\partial E} \cdot \frac{\partial E}{\partial d_y} $$
        Accumulate gradients to `dL_dmean2D` [S1].
    *   Gradient w.r.t conic matrix `con_o`:
        $$ \frac{\partial E}{\partial \text{con\_o}.x} = d_x^2 $$
        $$ \frac{\partial E}{\partial \text{con\_o}.y} = 2 \cdot d_x \cdot d_y $$
        $$ \frac{\partial E}{\partial \text{con\_o}.z} = d_y^2 $$
        $$ \frac{\partial L}{\partial \text{con\_o}.x} = \frac{\partial L}{\partial E} \cdot d_x^2 $$
        $$ \frac{\partial L}{\partial \text{con\_o}.y} = \frac{\partial L}{\partial E} \cdot 2 \cdot d_x \cdot d_y $$
        $$ \frac{\partial L}{\partial \text{con\_o}.z} = \frac{\partial L}{\partial E} \cdot d_y^2 $$
        Accumulate gradients to `dL_dconic2D` [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def inverse_multiquadric_splatting(pixel_x, pixel_y, gaussian_mean_2d, conic_matrix, opacity, correction_factor=1.38):
    """
    Forward and backward pass for Inverse Multiquadric Splatting.
    
    Args:
        pixel_x, pixel_y: Pixel coordinates.
        gaussian_mean_2d: Tuple (mu_x, mu_y).
        conic_matrix: Tuple (c_x, c_y, c_z, c_w) representing the 2x2 inverse covariance matrix 
                      [[c_x, c_y], [c_y, c_z]] and opacity c_w.
        opacity: Scalar opacity (redundant with conic_matrix[3] in some implementations, 
                 but kept for clarity if separate).
        correction_factor: Scalar multiplier for covariance (1.38).
        
    Returns:
        alpha: The computed alpha value.
        grad_mean: Tuple (grad_mu_x, grad_mu_y).
        grad_conic: Tuple (grad_c_x, grad_c_y, grad_c_z).
        grad_opacity: Scalar gradient w.r.t opacity.
    """
    # 1. Apply correction factor to conic matrix (inverse covariance)
    # Note: In the source, cov *= factor. Since conic is inverse cov, 
    # if cov is scaled by k, inverse cov is scaled by 1/k? 
    # However, Listing 7 says "cov2D *= correctionFactor" and then uses con_o derived from it.
    # The source code explicitly multiplies the conic matrix entries by 1.38 in the gradient 
    # calculation context or assumes the input conic is already scaled. 
    # Based on Listing 7: "cov2D *= correctionFactor" happens before projection or 
    # is applied to the 2D matrix. The gradient code uses "correctionFactor * denom2inv ...".
    # To match the source logic exactly:
    
    c_x = conic_matrix[0] * correction_factor
    c_y = conic_matrix[1] * correction_factor
    c_z = conic_matrix[2] * correction_factor
    c_w = conic_matrix[3] # Opacity
    
    # 2. Compute displacement
    d_x = pixel_x - gaussian_mean_2d[0]
    d_y = pixel_y - gaussian_mean_2d[1]
    
    # 3. Compute ellipse metric E
    E = c_x * d_x * d_x + c_z * d_y * d_y + 2.0 * c_y * d_x * d_y
    
    # 4. Culling
    if E <= 0.0 or E >= 9.0:
        return 0.0, (0.0, 0.0), (0.0, 0.0, 0.0), 0.0
        
    # 5. Forward Alpha
    G = 1.0 / (E + 1.0)
    alpha = min(0.99, c_w * G)
    
    # 6. Backward Gradients
    # Assume dL_dalpha is provided or returned as part of a larger context.
    # Here we return the local gradients dL/dalpha -> dL/dparams
    # Let's assume an incoming gradient dL_dalpha = 1.0 for the purpose of returning 
    # the Jacobian components. In practice, this is multiplied by the upstream gradient.
    
    # Gradient w.r.t G
    # dL_dG = c_w * dL_dalpha
    # We return the factor that multiplies dL_dalpha
    
    # Gradient w.r.t E
    # dL_dE = dL_dG * (-1 / (E + 1)^2)
    #       = c_w * dL_dalpha * (-1 / (E + 1)^2)
    
    # Gradient w.r.t Mean
    # dL_ddx = dL_dE * (2 * (c_x * d_x + c_y * d_y))
    # dL_ddy = dL_dE * (2 * (c_y * d_x + c_z * d_y))
    
    # Gradient w.r.t Conic
    # dL_dc_x = dL_dE * (d_x^2)
    # dL_dc_y = dL_dE * (2 * d_x * d_y)
    # dL_dc_z = dL_dE * (d_y^2)
    
    # Note: The source code Listing 7/8 shows specific gradient updates.
    # The source uses `dL_dG = con_o.w * dL_dalpha`.
    # Then `ellipsepow = - dL_dG * 2.f / pow(ellipse + 1.f, 2.f)`.
    # Wait, the source derivative of 1/(E+1) is -1/(E+1)^2.
    # The source code has `ellipsepow = - dL_dG * 2.f / ...`. 
    # This implies the derivative of E w.r.t variables includes a factor of 2?
    # Let's re-verify: E = c_x dx^2 + ...
    # dE/d(dx) = 2 c_x dx + 2 c_y dy.
    # The source code: `dG_ddelx = ellipsepow * (d.x * con_o.x + d.y * con_o.y)`.
    # `ellipsepow` includes the -2.
    # So `dG_ddelx` = `(- dL_dG * 2 / (E+1)^2)` * `(c_x dx + c_y dy)`.
    # This matches `dL_dE * (2 * (c_x dx + c_y dy))` if `dL_dE = dL_dG * (-1/(E+1)^2)`.
    # Yes: `dL_dE * 2 * (...)` = `dL_dG * (-1/(E+1)^2) * 2 * (...)`.
    
    # We return the coefficients for dL_dalpha
    
    coeff_dE = c_w * (-1.0 / ((E + 1.0) ** 2))
    
    grad_mu_x = coeff_dE * 2.0 * (c_x * d_x + c_y * d_y)
    grad_mu_y = coeff_dE * 2.0 * (c_y * d_x + c_z * d_y)
    
    grad_c_x = coeff_dE * (d_x ** 2)
    grad_c_y = coeff_dE * (2.0 * d_x * d_y)
    grad_c_z = coeff_dE * (d_y ** 2)
    
    grad_op = G # d(alpha)/d(c_w) = G
    
    return alpha, (grad_mu_x, grad_mu_y), (grad_c_x, grad_c_y, grad_c_z), grad_op
```

## Subtopics

### Opacity-Independent Gaussian Truncation
Uses a fixed threshold for alpha culling regardless of opacity. Choose when you want consistent visibility of low-opacity primitives. [10266-opacity-independent-gaussian-truncation.md]

### Directed Distance Field (DDF) for Gaussian Splatting
Uses a signed distance field representation for sharper boundaries. Choose for high-fidelity geometry reconstruction. [12854-directed-distance-field-ddf-for-gaussian-splatting.md]

### Atomized Proliferation
Focuses on generating new primitives during training. Choose when dealing with sparse initial reconstructions. [12924-atomized-proliferation.md]

### Voxel-based Gaussian Splatting
Discretizes space into voxels for efficient rendering. Choose for large-scale scenes. [13071-voxel-based-gaussian-splatting.md]

### Gaussian Voxel Kernel Functions (GVKF)
Uses voxel-based kernel functions. Choose for memory-efficient storage. [13979-gaussian-voxel-kernel-functions-gvkf.md]

### 3D Gaussian Particle Approximation of VDB Datasets
Maps VDB data to Gaussians. Choose for simulating fluid/soft body data. [14723-3d-gaussian-particle-approximation-of-vdb-datasets.md]

### Pruning and Reduction
Removes redundant primitives. Choose to reduce memory footprint. [14909-pruning-and-reduction.md]

## Operating characteristics
*   **Time Complexity:** Same as standard Gaussian splatting, $O(N)$ per pixel where $N$ is the number of overlapping Gaussians. The inverse multiquadric function involves a division and addition, comparable to the exponential in Gaussian splatting.
*   **Memory Complexity:** Same as standard Gaussian splatting. No additional storage required beyond the standard Gaussian parameters.
*   **Performance:** The method introduces a correction factor ($1.38$) and a different kernel shape. The culling bounds ($0$ to $9$) are similar to standard Gaussian splatting (where $E \approx 4-5$ is often used for $3\sigma$).
*   **Strengths:** Provides a different decay profile that may reduce artifacts in high-curvature areas compared to the Gaussian kernel.
*   **Limitations:** The inverse multiquadric function has heavier tails than the Gaussian, which might require careful culling (set to $E=9$ in [S1]) to prevent distant artifacts.
*   **When to use:** When standard Gaussian splatting produces artifacts in sharp edges or high-frequency details.
*   **When not to use:** When maximum rendering speed is critical and the Gaussian kernel is sufficient, or when memory is extremely constrained (though memory usage is identical).

## Background
Inverse Multiquadric Splatting was introduced in the paper "DARB-Splatting: Generalizing Splatting with Decaying Anisotropic Radial Basis Functions" [S1]. It generalizes the standard Gaussian splatting by allowing different radial basis functions, specifically the inverse multiquadric function, to improve rendering quality in certain scenarios.

## Sources
[S1] DARB-Splatting: Generalizing Splatting with Decaying Anisotropic Radial Basis Functions, https://doi.org/10.1109/WACV61042.2026.00438, fetched 2026-10-30
