# DARBF Backpropagation

Also called: DARBF Gradient Calculation, Differentiable DARBF Rasterizer

## Overview
DARBF Backpropagation is the gradient computation mechanism for Decaying Anisotropic Radial Basis Functions (DARBFs) within differentiable splatting frameworks. It solves the problem of propagating loss gradients through non-Gaussian activation functions used in Gaussian Avatar Reconstruction, specifically enabling the optimization of splat parameters (mean, covariance, opacity) when the standard Gaussian assumption does not hold. It differs from standard 3D Gaussian Splatting (3DGS) [3D Gaussian Splatting Rasterization](3D Gaussian Splatting Rasterization.md) by explicitly replacing the fixed Gaussian derivative formulas with chain-rule expansions tailored to the specific DARBF activation function, such as the half-cosine function. It extends the [Differentiable Backpropagation Mechanisms](11407-differentiable-backpropagation-mechanisms.md) by providing the specific Jacobian terms required for DARBF-based splats.

## How it works
**Inputs:**
*   Pixel coordinates $x \in \mathbb{R}^2$.
*   Projected splat mean $\boldsymbol{\mu}' \in \mathbb{R}^2$.
*   Projected inverse covariance matrix $(\Sigma'_{2\times 2})^{-1} \in \mathbb{R}^{2\times 2}$.
*   DARBF frequency parameter $\xi$ (scalar).
*   Gradient of the loss with respect to the splat weight $w$, denoted $\frac{\partial L}{\partial w}$.

**Outputs:**
*   Gradient of the loss with respect to the projected mean difference: $\frac{\partial L}{\partial (x - \boldsymbol{\mu}')}$.
*   Gradient of the loss with respect to the inverse covariance matrix: $\frac{\partial L}{\partial (\Sigma'_{2\times 2})^{-1}}$.

**Data Structures:**
The implementation modifies the CUDA kernel of the differentiable rasterizer. The core data structure remains the splat descriptor containing the projected mean $\boldsymbol{\mu}'$ and the $2\times 2$ inverse covariance matrix $(\Sigma'_{2\times 2})^{-1}$. The gradient accumulation buffers for these parameters are updated using the derived partial derivatives.

**Steps:**
1.  Compute the Mahalanobis distance metric $d_M$ scaled by $\xi$:
    $$ d_M = \frac{(x - \boldsymbol{\mu}')^T (\Sigma'_{2\times 2})^{-1} (x - \boldsymbol{\mu}')}{\xi} $$
2.  Compute the derivative of the activation function $F$ with respect to $d_M$, denoted $F'(d_M)$. For the half-cosine DARBF example:
    $$ F'(d_M) = -\sin(d_M) $$
    *Note: The general form provided in the source is $F'(d_M) = -\sin\left(\frac{(x - \boldsymbol{\mu}')^T(\Sigma'_{2\times 2})^{-1}(x - \boldsymbol{\mu}')}{\xi}\right)$ [S1].*
3.  Compute the partial derivative of the distance metric $d_M$ with respect to the projected mean difference vector $(x - \boldsymbol{\mu}')$:
    $$ \frac{\partial d_M}{\partial (x - \boldsymbol{\mu}')} = \frac{2 (\Sigma'_{2\times 2})^{-1} (x - \boldsymbol{\mu}')}{\xi} $$
4.  Compute the partial derivative of the distance metric $d_M$ with respect to the inverse covariance matrix $(\Sigma'_{2\times 2})^{-1}$:
    $$ \frac{\partial d_M}{\partial (\Sigma'_{2\times 2})^{-1}} = \frac{(x - \boldsymbol{\mu}')(x - \boldsymbol{\mu}')^T}{\xi} $$
5.  Apply the chain rule to compute the gradient of the weight $w$ with respect to the projected mean difference:
    $$ \frac{dw}{d(x - \boldsymbol{\mu}')} = F'(d_M) \cdot \frac{\partial d_M}{\partial (x - \boldsymbol{\mu}')} $$
6.  Apply the chain rule to compute the gradient of the weight $w$ with respect to the inverse covariance matrix:
    $$ \frac{dw}{d((\Sigma'_{2\times 2})^{-1})} = F'(d_M) \cdot \frac{\partial d_M}{\partial (\Sigma'_{2\times 2})^{-1}} $$
7.  Multiply the resulting weight gradients by the upstream loss gradient $\frac{\partial L}{\partial w}$ to obtain the final gradients for backpropagation.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def darbf_backprop(x, mu_prime, inv_cov, xi, grad_L_w):
    """
    Computes gradients for DARBF splatting parameters.
    
    Args:
        x: Pixel coordinate (2,)
        mu_prime: Projected mean (2,)
        inv_cov: 2x2 Inverse covariance matrix (2, 2)
        xi: DARBF frequency parameter (scalar)
        grad_L_w: Gradient of loss w.r.t. weight w (scalar)
        
    Returns:
        grad_mu: Gradient w.r.t. projected mean (2,)
        grad_inv_cov: Gradient w.r.t. inverse covariance (2, 2)
    """
    # Step 1: Compute difference vector
    diff = x - mu_prime  # (2,)
    
    # Step 2: Compute scaled Mahalanobis distance d_M
    # d_M = (diff^T @ inv_cov @ diff) / xi
    quadratic_form = diff @ inv_cov @ diff
    d_M = quadratic_form / xi
    
    # Step 3: Compute F'(d_M)
    # For half-cosine DARBF: F'(d_M) = -sin(d_M)
    F_prime = -sin(d_M)
    
    # Step 4: Compute partial derivative of d_M w.r.t. (x - mu')
    # d(d_M) / d(diff) = 2 * inv_cov @ diff / xi
    grad_d_M_diff = 2.0 * (inv_cov @ diff) / xi  # (2,)
    
    # Step 5: Compute partial derivative of d_M w.r.t. inv_cov
    # d(d_M) / d(inv_cov) = (diff @ diff^T) / xi
    # Note: This is a 2x2 matrix
    grad_d_M_inv_cov = (diff[:, None] @ diff[None, :]) / xi  # (2, 2)
    
    # Step 6: Chain rule for weight gradients
    # dw / d(diff) = F'(d_M) * d(d_M) / d(diff)
    grad_w_diff = F_prime * grad_d_M_diff  # (2,)
    
    # dw / d(inv_cov) = F'(d_M) * d(d_M) / d(inv_cov)
    grad_w_inv_cov = F_prime * grad_d_M_inv_cov  # (2, 2)
    
    # Step 7: Scale by upstream loss gradient
    grad_mu = -grad_w_diff * grad_L_w  # Negative because diff = x - mu, so d/dmu = -d/diff
    grad_inv_cov_final = grad_w_inv_cov * grad_L_w
    
    return grad_mu, grad_inv_cov_final
```

**Parameters:**
*   $\xi$: Controls the frequency/decay of the DARBF. In the half-cosine function example, $\xi = 18\pi$ [S1].

**Edge Cases and Pitfalls:**
*   **Precision:** The calculation involves matrix-vector products and trigonometric functions. Standard floating-point precision should be sufficient, but care must be taken with the division by $\xi$ if $\xi$ is very small.
*   **Degenerate Input:** If the inverse covariance matrix is singular or ill-conditioned, the quadratic form may become unstable. This is a general issue with Gaussian/DARBF splatting, not specific to the backpropagation mechanism itself.
*   **Activation Function Specificity:** The derivative $F'(d_M)$ depends on the specific DARBF chosen. The formulas provided are for the half-cosine function. Other DARBFs require their specific $F'(d_M)$ [S1].

## Subtopics

### Backpropagation with Per-Splat Parallelization
Optimizes the gradient computation by parallelizing the backpropagation steps across individual splats on the GPU. It differs by focusing on computational efficiency and memory bandwidth utilization rather than the mathematical derivation of the DARBF gradients. Choose this when performance is critical and the gradient formulas are already established. [Backpropagation with Per-Splat Parallelization](3533-backpropagation-with-per-splat-parallelization.md)

### Stochastic Gradient Estimation for Differentiable Rasterization
Uses stochastic methods to estimate gradients for differentiable rasterization, potentially reducing variance or computational cost. It differs by using sampling-based approaches instead of deterministic chain-rule derivatives. Choose this when dealing with non-differentiable or highly complex rendering equations where deterministic gradients are intractable. [Stochastic Gradient Estimation for Differentiable Rasterization](11381-stochastic-gradient-estimation-for-differentiable-rasterizat.md)

## Operating characteristics
*   **Time Complexity:** The gradient computation per splat is $O(1)$, involving a few matrix-vector multiplications and trigonometric operations. It is comparable to standard 3DGS backpropagation but with slightly more complex arithmetic due to the specific DARBF derivatives.
*   **Memory Complexity:** $O(1)$ per splat for storing intermediate gradient values. No additional global memory structures are required beyond the standard rasterizer buffers.
*   **Measured Performance:** Implemented in the DARBF-Splatting codebase, anchored on the 3DGS codebase as of October 2024 [S1]. Experiments were conducted on a single NVIDIA GeForce RTX 4090 GPU [S1]. Specific timing metrics are not provided in the sources.
*   **Strengths:** Enables the use of non-Gaussian splats (DARBFs) which may offer better representation for certain types of geometry or appearance (e.g., in Gaussian Avatar Reconstruction).
*   **Limitations:** Requires custom CUDA kernel modifications to the rasterizer. The performance benefit of DARBFs over Gaussians is not quantified in the sources, only that the backpropagation mechanism supports them.
*   **When to use:** When using Decaying Anisotropic Radial Basis Functions for splatting and needing to optimize their parameters via gradient descent.
*   **When not to use:** When using standard Gaussian splats, as the standard 3DGS backpropagation is simpler and likely more optimized.

## Background
DARBF Backpropagation was introduced in the paper "DARB-Splatting: Generalizing Splatting with Decaying Anisotropic Radial Basis Functions" [S1]. It extends the 3D Gaussian Splatting framework [11] by replacing the Gaussian activation function with DARBFs, such as the half-cosine function. The backpropagation mechanism was explicitly modified to support the gradient terms for these new basis functions. The implementation is based on the 3DGS codebase updated as of October 2024 [S1].

## Sources
[S1] DARB-Splatting: Generalizing Splatting with Decaying Anisotropic Radial Basis Functions, https://doi.org/10.1109/WACV61042.2026.00438, fetched 2026-10-30
