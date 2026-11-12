# DARBF Backpropagation

Also called: DARBF Gradient Calculation, Differentiable DARBF Rasterizer

## Overview
DARBF Backpropagation is the gradient computation mechanism for Decaying Anisotropic Radial Basis Functions (DARBFs) within the 3D Gaussian Splatting pipeline. It solves the problem of differentiating the rendering loss with respect to Gaussian parameters when the alpha splatting function is not a standard Gaussian but a decaying anisotropic function (e.g., half-cosine). It differs from standard 3DGS backpropagation by replacing the fixed Gaussian derivative formulas with specific chain-rule terms derived from the DARBF activation function, while maintaining the same alpha compositing structure defined in [Differentiable Backpropagation Mechanisms](11407-differentiable-backpropagation-mechanisms.md).

## How it works
**Inputs:**
*   Gradient of the loss with respect to the pixel color: $\frac{\partial L}{\partial C_j}$ for each pixel $j$.
*   Forward pass data for splat $s$ contributing to pixel $j$:
    *   Projected mean $\boldsymbol{\mu}'$ (2D vector).
    *   Inverse projected covariance matrix $(\Sigma'_{2\times 2})^{-1}$ (2x2 matrix).
    *   Weight $w_s$ (scalar opacity contribution before compositing).
    *   Distance metric $d_M$ (scalar).
    *   Parameter $\xi$ (scalar frequency/decay control).
    *   Accumulated transparency $\alpha_{prev} = \prod_{k < s} (1 - \alpha_k)$.

**Outputs:**
*   Gradients with respect to the projected mean difference: $\frac{\partial L}{\partial (x - \boldsymbol{\mu}')}$.
*   Gradients with respect to the inverse covariance matrix: $\frac{\partial L}{\partial (\Sigma'_{2\times 2})^{-1}}$.

**Data Structures:**
*   $x$: The 2D pixel coordinate vector.
*   $\boldsymbol{\mu}'$: The 2D projected mean vector.
*   $\Sigma'_{2\times 2}$: The 2D projected covariance matrix.
*   $(\Sigma'_{2\times 2})^{-1}$: The inverse of the projected covariance matrix.
*   $d_M$: The Mahalanobis-like distance metric defined as $d_M = \frac{(x - \boldsymbol{\mu}')^T(\Sigma'_{2\times 2})^{-1}(x - \boldsymbol{\mu}')}{\xi}$.
*   $F(d_M)$: The DARBF activation function (e.g., half-cosine).
*   $F'(d_M)$: The derivative of the activation function.

**Steps:**
1.  **Compute Activation Derivative:** Calculate the derivative of the DARBF activation function $F$ with respect to the distance metric $d_M$ using the chain rule. For the half-cosine example, this is:
    $$ F'(d_M) = -\sin\left(\frac{(x - \boldsymbol{\mu}')^T(\Sigma'_{2\times 2})^{-1}(x - \boldsymbol{\mu}')}{\xi}\right) $$
    [S1]

2.  **Compute Distance Metric Derivatives:** Calculate the partial derivatives of the distance metric $d_M$ with respect to the inverse covariance matrix and the projected mean difference.
    *   With respect to the inverse covariance matrix:
        $$ \frac{\partial d_M}{\partial(\Sigma'_{2 \times 2})^{-1}} = \frac{(x - \boldsymbol{\mu}')(x - \boldsymbol{\mu}')^T}{\xi} $$
        [S1]
    *   With respect to the projected mean difference $(x - \boldsymbol{\mu}')$:
        $$ \frac{\partial d_M}{\partial(x - \boldsymbol{\mu}')} = \frac{2(\Sigma'_{2 \times 2})^{-1}(x - \boldsymbol{\mu}')}{\xi} $$
        [S1]

3.  **Compute Weight Gradients:** Apply the chain rule to compute the gradient of the weight $w$ (which determines the splat's opacity contribution $\alpha_s$) with respect to the Gaussian parameters.
    *   Gradient with respect to the projected mean difference:
        $$ \frac{dw}{d(x - \boldsymbol{\mu}')} = F'(d_M) \frac{\partial d_M}{\partial(x - \boldsymbol{\mu}')} $$
        [S1]
    *   Gradient with respect to the inverse covariance matrix:
        $$ \frac{dw}{d((\Sigma'_{2\times 2})^{-1})} = F'(d_M) \frac{\partial d_M}{\partial(\Sigma'_{2 \times 2})^{-1}} $$
        [S1]

4.  **Accumulate Loss Gradients:** Multiply the weight gradients by the gradient flowing from the loss through the alpha compositing equation. If $\frac{\partial L}{\partial \alpha_s}$ is the gradient of the loss with respect to the splat's opacity $\alpha_s$ (derived from the compositing chain rule in the parent article), then:
    $$ \frac{\partial L}{\partial (x - \boldsymbol{\mu}')} = \frac{\partial L}{\partial \alpha_s} \frac{d\alpha_s}{dw} \frac{dw}{d(x - \boldsymbol{\mu}')} $$
    $$ \frac{\partial L}{\partial (\Sigma'_{2\times 2})^{-1}} = \frac{\partial L}{\partial \alpha_s} \frac{d\alpha_s}{dw} \frac{dw}{d((\Sigma'_{2\times 2})^{-1})} $$
    [G] Note: The term $\frac{d\alpha_s}{dw}$ depends on the specific mapping from weight $w$ to opacity $\alpha_s$ (e.g., $\alpha_s = w \cdot \alpha_{prev}$ or similar). The sources provide the derivatives of $w$ with respect to the geometric parameters, which must be scaled by the compositing gradient.

5.  **Map to Parameter Space:** Convert the gradients with respect to $(x - \boldsymbol{\mu}')$ and $(\Sigma'_{2\times 2})^{-1}$ back to the gradients for the learnable parameters (mean $\boldsymbol{\mu}$, covariance $\Sigma$, etc.) using the Jacobian of the projection and parameterization steps. [G] This step is standard in differentiable rasterizers and relies on the forward projection derivatives.

**Parameters:**
*   $\xi$: A parameter controlling the frequency/decay of the DARBF. In the half-cosine function example, $\xi = 18\pi$. [S1]

**Edge Cases and Pitfalls:**
*   **Precision:** The sine function in $F'(d_M)$ can oscillate rapidly if $\xi$ is small, leading to high-frequency gradients. [S1]
*   **Boundary Conditions:** The half-cosine function typically has a support limit. Gradients should be zero outside the support. [G]
*   **Singular Covariance:** If $(\Sigma'_{2\times 2})^{-1}$ is ill-conditioned, the distance metric $d_M$ may become unstable. [G]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def darbf_backprop(x, mu_prime, inv_cov, xi, grad_L_alpha_s, d_alpha_s_dw):
    """
    Computes gradients for DARBF splatting.
    
    Args:
        x: (2,) pixel coordinate vector.
        mu_prime: (2,) projected mean vector.
        inv_cov: (2, 2) inverse projected covariance matrix.
        xi: scalar decay parameter.
        grad_L_alpha_s: scalar gradient of loss w.r.t. opacity alpha_s.
        d_alpha_s_dw: scalar derivative of alpha_s w.r.t. weight w.
        
    Returns:
        grad_mu_diff: (2,) gradient w.r.t. (x - mu_prime).
        grad_inv_cov: (2, 2) gradient w.r.t. inv_cov.
    """
    # Step 1: Compute distance metric d_M
    diff = x - mu_prime
    d_M = (diff.T @ inv_cov @ diff) / xi
    
    # Step 2: Compute activation derivative F'(d_M)
    # For half-cosine: F'(d_M) = -sin(d_M)
    F_prime = -sin(d_M)
    
    # Step 3: Compute distance metric derivatives
    # d(d_M) / d(inv_cov) = (diff @ diff.T) / xi
    grad_dM_d_inv_cov = (diff.reshape(-1, 1) @ diff.reshape(1, -1)) / xi
    
    # d(d_M) / d(diff) = 2 * inv_cov @ diff / xi
    grad_dM_d_diff = 2.0 * (inv_cov @ diff) / xi
    
    # Step 4: Compute weight gradients
    # dw / d(diff) = F'(d_M) * d(d_M) / d(diff)
    dw_d_diff = F_prime * grad_dM_d_diff
    
    # dw / d(inv_cov) = F'(d_M) * d(d_M) / d(inv_cov)
    dw_d_inv_cov = F_prime * grad_dM_d_inv_cov
    
    # Step 5: Scale by loss gradient and compositing derivative
    # grad_L_d_diff = grad_L_alpha_s * d_alpha_s_dw * dw_d_diff
    grad_mu_diff = grad_L_alpha_s * d_alpha_s_dw * dw_d_diff
    
    # grad_L_d_inv_cov = grad_L_alpha_s * d_alpha_s_dw * dw_d_inv_cov
    grad_inv_cov = grad_L_alpha_s * d_alpha_s_dw * dw_d_inv_cov
    
    return grad_mu_diff, grad_inv_cov
```

## Subtopics

### Backpropagation with Per-Splat Parallelization
Uses a different parallelization strategy for gradient computation, assigning one thread per splat. Choose when memory bandwidth is the bottleneck and splat counts are high. [Link](3533-backpropagation-with-per-splat-parallelization.md)

### Stochastic Gradient Estimation for Differentiable Rasterization
Uses stochastic sampling to approximate gradients, reducing memory usage. Choose when memory is constrained or for large-scale scenes. [Link](11381-stochastic-gradient-estimation-for-differentiable-rasterizat.md)

## Operating characteristics
*   **Time Complexity**: Same as standard 3DGS backpropagation, $O(N)$ where $N$ is the number of splats, as the additional trigonometric operations are constant time per splat.
*   **Memory Complexity**: Same as standard 3DGS, $O(N)$ for storing gradients.
*   **Performance**: Measured on NVIDIA GeForce RTX 4090. [S1] No specific FPS numbers given for DARBF backpropagation alone, but it is implemented on the 3DGS codebase. [S1]
*   **Strengths**: Enables the use of non-Gaussian basis functions (DARBFs) which may offer better representation for certain geometries or materials. [S1]
*   **Limitations**: Requires careful tuning of the $\xi$ parameter. [S1] The oscillatory nature of the derivative may require smaller learning rates. [G]
*   **When to use**: When the scene benefits from the specific shape properties of DARBFs (e.g., sharper edges, specific decay profiles) compared to Gaussians. [S1]
*   **When not to use**: When standard Gaussian Splatting performance is sufficient, or when the overhead of tuning $\xi$ is not justified.

## Background
DARBF Backpropagation was introduced in "DARB-Splatting: Generalizing Splatting with Decaying Anisotropic Radial Basis Functions" (2026). [S1] It extends the 3D Gaussian Splatting framework by replacing the Gaussian basis function with DARBFs, requiring a modification of the differentiable rasterizer's gradient computation. [S1] The implementation is based on the 3DGS codebase updated as of October 2024. [S1]

## Sources
[S1] DARB-Splatting: Generalizing Splatting with Decaying Anisotropic Radial Basis Functions, https://doi.org/10.1109/WACV61042.2026.00438, fetched 2026-10-30
