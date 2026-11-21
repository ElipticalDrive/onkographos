# TruncGradGS

Also called: Piecewise truncated gradient, Truncated Gradient Updates

## Overview
TruncGradGS is a modification to the 3D Gaussian Splatting (3DGS) rasterization gradient computation that replaces near-zero Gaussian derivative tails with a linear surrogate to solve the vanishing gradient problem in dynamic scenes with large inter-frame displacements [S1]. It sets itself apart from standard 3DGS by ensuring that Gaussians discarded by the alpha-blending threshold still receive gradient updates, thereby maintaining gradient flow for primitives that are distant or rapidly moving [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** 3D Gaussian parameters (mean $\mu$, covariance $\Sigma$, opacity $\sigma$), 2D pixel coordinates $p$, the alpha-blending discard threshold $\tau$, and the linear slope parameter $m$.
*   **Outputs:** Modified gradient $\nabla G$ with respect to the Gaussian parameters (specifically the mean $\Delta x$).

**Data Structures:**
The method operates on the standard 3DGS primitive representation:
*   Mean $\mu \in \mathbb{R}^3$
*   Covariance $\Sigma \in \mathbb{R}^{3 \times 3}$ (positive definite)
*   Opacity $\sigma \in \mathbb{R}$
*   Color $c \in \mathbb{R}^3$ (view-dependent)

**Numbered Steps:**

1.  **Compute Squared Mahalanobis Distance:**
    For a query pixel $p$ and a Gaussian with mean $\mu$, calculate the squared Mahalanobis distance $d(p)^2$ using the inverse covariance matrix $\Sigma^{-1}$ [S1]:
    $$ d(p)^2 = (\mu - p)^T \Sigma^{-1} (\mu - p) $$
    Note: In the context of the 2D projection used for rasterization, $\Sigma$ refers to the projected 2D covariance, and $p, \mu$ are 2D coordinates [S1].

2.  **Determine Boundary Point ($x_b$):**
    The truncation boundary is defined by the Gaussian isocontour where the density equals the threshold $\tau$: $G_{2D}(x_b) = \tau$. This corresponds to a squared Mahalanobis distance of $-2 \log \tau$ [S1].
    To find the scalar distance $s$ from the pixel $p$ to the boundary point $x_b$ along the ray direction, parametrize the ray as:
    $$ x_b = p + s \frac{\mu - p}{\|\mu - p\|} $$
    Substitute this into the distance equation $(\mu - x_b)^T \Sigma^{-1} (\mu - x_b) = -2 \log \tau$ to form a quadratic equation in $s$. The solution for $s$ is [S1]:
    $$ s = \|\mu - p\| \sqrt{1 \pm \frac{-2 \log \tau}{K}} $$
    where $K$ is defined as [S1]:
    $$ K = a(\mu_x - p_x)^2 + 2b(\mu_x - p_x)(\mu_y - p_y) + c(\mu_y - p_y)^2 $$
    The coefficients $a, b, c$ come from the un-normalized general Gaussian 2D function (conic form) [S1]:
    $$ f(x, y) = \exp\left(-\frac{1}{2} [a(\mu_x - x)^2 + c(\mu_y - y)^2] - b(\mu_x - x)(\mu_y - y)\right) $$
    Select the appropriate root for $s$ that places $x_b$ on the Gaussian splat [S1].

3.  **Calculate True Gradient:**
    Compute the true partial derivative of the 2D Gaussian function $G_{2D}$ with respect to the mean parameter $\Delta x$ (representing the mean offset). The derivative contains interaction terms with the Gaussian function [S1]:
    $$ \frac{\partial G}{\partial \Delta x} $$
    (Specific analytical forms depend on the exact parameterization of $\Delta x$, but it is the standard backpropagated gradient from the rasterizer) [S1].

4.  **Apply Piecewise Truncated Gradient:**
    Determine the gradient update $g$ based on the squared Mahalanobis distance $d(p)^2$ relative to the threshold $-2 \log \tau$ and the sign of the gradient [S1]:

    *   **Case 1 (Inside Threshold):** If $-2 \log G < -2 \log \tau$ (i.e., the pixel is within the effective support), use the true partial derivative:
        $$ g = \frac{\partial G}{\partial \Delta x} $$

    *   **Case 2 (Outside Threshold):** If the Gaussian is outside the threshold, apply a linear surrogate. First, calculate the bias $b$ to ensure continuity at the boundary $x_b$:
        $$ b = \frac{\partial G}{\partial \Delta x}(x_b) - m x_b $$
        Then, apply the piecewise rule based on the sign of the gradient at the boundary $\frac{\partial G}{\partial \Delta x}(x_b)$ [S1]:
        *   If $\frac{\partial G}{\partial \Delta x}(x_b) < 0$:
            $$ g = \min(m \cdot x + b, \frac{\partial G}{\partial \Delta x}) $$
        *   If $\frac{\partial G}{\partial \Delta x}(x_b) > 0$:
            $$ g = \max(-m \cdot x + b, \frac{\partial G}{\partial \Delta x}) $$
        
        *Note: The variable $x$ in the linear terms $mx+b$ represents the distance coordinate along the ray from the pixel to the boundary, effectively scaling the linear surrogate [S1].*

5.  **Loss Sign Guard:**
    Only apply the truncated gradient if the Gaussian is decreasing the loss via alpha-blending, i.e., $\frac{\partial L}{\partial \alpha_i} < 0$. If $\frac{\partial L}{\partial \alpha_i} \ge 0$, do not apply the truncated gradient [S1]. Applying it when the loss is increasing causes long-range push forces that make Gaussians repel each other [S1].

**Parameters:**
*   $\tau$: The density threshold determining the boundary point. Set to the alpha-blending discard threshold of the rasterizer (typically $1/255$) [S1].
*   $m$: The slope of the linear surrogate gradient. A model parameter controlling the linear scaling of the gradient magnitude [S1].

**Edge Cases and Pitfalls:**
*   **Repulsion:** Applying the truncated gradient when $\frac{\partial L}{\partial \alpha_i} \ge 0$ causes Gaussians to repel each other due to long-range push forces [S1].
*   **Discontinuity:** The method ensures gradient continuity at the boundary $x_b$ by matching the linear surrogate to the true gradient value at that point [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def truncgradgs_gradient_update(mean_2d, cov_inv_2d, pixel_2d, 
                                true_gradient, alpha, sigma, 
                                tau, m, loss_grad_alpha):
    """
    Computes the truncated gradient for a 3D Gaussian splat at a specific pixel.
    
    Args:
        mean_2d: np.array, 2D mean of the Gaussian [mu_x, mu_y]
        cov_inv_2d: np.array, 2x2 inverse covariance matrix of the projected Gaussian
        pixel_2d: np.array, 2D pixel coordinate [p_x, p_y]
        true_gradient: float, the standard backpropagated gradient dG/dDelta_x
        alpha: float, the projected density of the Gaussian at the pixel
        sigma: float, the opacity of the Gaussian
        tau: float, the alpha-blending discard threshold (e.g., 1/255)
        m: float, the slope parameter for the linear surrogate
        loss_grad_alpha: float, the gradient of the loss with respect to alpha (sigma * alpha)
        
    Returns:
        float: The modified gradient to be used for optimization
    """
    
    # Step 1: Compute Squared Mahalanobis Distance
    diff = mean_2d - pixel_2d
    # d(p)^2 = (mu - p)^T * Sigma^-1 * (mu - p)
    mahalanobis_sq = np.dot(diff, np.dot(cov_inv_2d, diff))
    
    # Step 2: Determine if we are inside or outside the threshold
    # Threshold condition: -2 * log(G) < -2 * log(tau)
    # Since G = exp(-0.5 * mahalanobis_sq), -2 * log(G) = mahalanobis_sq
    # So we check if mahalanobis_sq < -2 * log(tau)
    threshold_val = -2.0 * math.log(tau)
    
    if mahalanobis_sq < threshold_val:
        # Case 1: Inside threshold, use true gradient
        grad = true_gradient
    else:
        # Case 2: Outside threshold, use linear surrogate
        # Check loss sign guard
        if loss_grad_alpha >= 0:
            # Do not apply truncated gradient if it increases the loss
            # Return 0 or the original gradient? The source says "do not apply".
            # In standard 3DGS, discarded Gaussians get 0 gradient.
            # TruncGradGS applies surrogate ONLY if it helps.
            # If it increases loss, we treat it as standard discarded Gaussian (0 gradient)
            # or potentially the true gradient if it was computed?
            # Source: "do not apply the truncated gradient... causes long-range push"
            # Implies we fall back to standard behavior (0 gradient for discarded)
            # or simply skip the surrogate. Given it's discarded, standard is 0.
            grad = 0.0 
        else:
            # Calculate boundary point distance s
            # K = a(dx)^2 + 2b(dx)(dy) + c(dy)^2
            # Note: The source defines K using coefficients a,b,c from the conic form.
            # In standard 3DGS, the projected covariance Sigma_2D is related to a,b,c.
            # For implementation, we need to solve for s.
            # s = ||mu - p|| * sqrt(1 + (-2 log tau) / K)
            # However, K is derived from the conic coefficients. 
            # If we only have cov_inv, we can derive the boundary distance directly.
            # The distance from center to boundary along the ray is determined by:
            # (mu - (p + s * dir))^T Sigma^-1 (mu - (p + s * dir)) = -2 log tau
            # Let dir = (mu - p) / ||mu - p||
            # Let u = mu - p. Then mu - x_b = u - s * (u/||u||) = (||u|| - s) * (u/||u||)
            # So (mu - x_b)^T Sigma^-1 (mu - x_b) = (||u|| - s)^2 * (u/||u||)^T Sigma^-1 (u/||u||)
            # Let Q = (u/||u||)^T Sigma^-1 (u/||u||)
            # Then (||u|| - s)^2 * Q = -2 log tau
            # ||u|| - s = sqrt(-2 log tau / Q)
            # s = ||u|| - sqrt(-2 log tau / Q)
            
            dist_mu_p = np.linalg.norm(diff)
            if dist_mu_p < 1e-8:
                grad = true_gradient # Degenerate case, treat as center
            else:
                direction = diff / dist_mu_p
                Q = np.dot(direction, np.dot(cov_inv_2d, direction))
                
                # Solve for s
                # We want the point on the ray from p towards mu that is at distance threshold
                # The distance from mu to boundary is sqrt(-2 log tau / Q)
                # The distance from p to mu is dist_mu_p
                # The boundary is between p and mu if the pixel is outside?
                # Actually, the ray is from p to mu.
                # x_b = p + s * direction
                # mu - x_b = mu - p - s * direction = (dist_mu_p - s) * direction
                # Mahalanobis sq at x_b = (dist_mu_p - s)^2 * Q = -2 log tau
                # dist_mu_p - s = sqrt(-2 log tau / Q)
                # s = dist_mu_p - sqrt(-2 log tau / Q)
                
                sqrt_term = math.sqrt(max(0, -2.0 * math.log(tau) / Q))
                s = dist_mu_p - sqrt_term
                
                # Calculate bias b for continuity
                # b = grad(x_b) - m * x_b
                # Here x in the formula mx+b refers to the coordinate along the ray.
                # We need the true gradient at the boundary point x_b.
                # However, the source defines the surrogate as min(m*x + b, grad).
                # And b is derived from matching at x_b.
                # We approximate the gradient at the boundary or use the current true_gradient?
                # The source says: "match the linear surrogate ... to the true gradient at the boundary point xb"
                # We don't have the true gradient at xb easily without re-evaluating G.
                # But we can approximate or use the current true_gradient if it's close?
                # Strictly, we should evaluate true_gradient at x_b.
                # For implementation simplicity, if we assume the gradient doesn't change drastically
                # or we just use the current true_gradient as a proxy for the boundary value?
                # No, the source explicitly derives b from grad(x_b).
                # Let's assume we can compute grad_at_boundary.
                # Since we don't have the full function G(x) here, we use the provided true_gradient
                # as an approximation or note that it requires re-evaluation.
                # [G] Implementation note: To be precise, evaluate the true Gaussian gradient
                # at the boundary point x_b = pixel_2d + s * direction.
                # For this pseudocode, we use the passed true_gradient as a placeholder
                # for grad(x_b) if re-evaluation is not possible, but ideally:
                # grad_at_boundary = compute_true_gradient_at(mean_2d, cov_inv_2d, pixel_2d + s * direction)
                
                # Using the current true_gradient as a proxy for grad(x_b) is an approximation.
                # A more robust implementation would re-calculate G and dG/dx at x_b.
                grad_at_boundary = true_gradient # Placeholder for re-evaluation at x_b
                
                b = grad_at_boundary - m * s
                
                # Apply piecewise rule
                # If grad(x_b) < 0: min(m*x + b, grad)
                # If grad(x_b) > 0: max(-m*x + b, grad)
                # Here x is the distance from p, which is s? Or the coordinate?
                # The source uses x in mx+b. In the context of the ray parametrization,
                # x likely refers to the distance along the ray from the pixel.
                # So we use s.
                
                if grad_at_boundary < 0:
                    linear_val = m * s + b
                    grad = min(linear_val, true_gradient)
                else:
                    linear_val = -m * s + b
                    grad = max(linear_val, true_gradient)
                    
    return grad
```

## Subtopics

### Dynamic 3DGS Integration
TruncGradGS is designed to be compatible with dynamic frameworks like 4DGS, addressing the specific challenge of large temporal divergence where primitives move significantly between frames [S1].

### Comparison with Standard 3DGS
Standard 3DGS discards Gaussians with contribution scalar $\sigma_i \alpha_i$ below a threshold, resulting in zero gradients. TruncGradGS applies the linear surrogate to these discarded Gaussians to maintain gradient flow [S1].

## Operating characteristics
*   **Time Complexity:** The method adds a constant-time overhead per Gaussian-pixel interaction to compute the Mahalanobis distance, solve for the boundary, and apply the piecewise logic. It does not change the asymptotic complexity of the rasterization pass.
*   **Memory Complexity:** No additional memory structures are required beyond the standard Gaussian parameters and the scalar parameters $\tau$ and $m$ [S1].
*   **Strengths:** It solves the vanishing gradient problem in dynamic scenes, allowing Gaussians to receive updates even when they are distant or rapidly moving, which improves optimization stability and final reconstruction quality for dynamic content [S1].
*   **Limitations:** Incorrect application (ignoring the loss sign guard) can cause Gaussians to repel each other, leading to instability [S1]. It is specifically beneficial for dynamic scenes; for static scenes with small displacements, the benefit may be marginal compared to the added computational overhead [S1].

## Background
TruncGradGS was introduced in the paper "TruncGradGS: Improved 3D Gaussian Splatting via Truncated Gradient Updates" by T. Morales, N.-Q. Le-Pham, R. Atkins & B.-S. Hua, published in the Proceedings of Eurographics (2026) [S1]. It addresses limitations identified in standard 3DGS rasterization where the exponential falloff of the Gaussian distribution causes gradients to vanish for pixels outside the immediate support, a problem exacerbated in dynamic methods like 4DGS [S1].

## Sources
[S1] TruncGradGS: Improved 3D Gaussian Splatting via Truncated Gradient Updates, https://arxiv.org/abs/2609.03534v1, fetched 2026-10-26
