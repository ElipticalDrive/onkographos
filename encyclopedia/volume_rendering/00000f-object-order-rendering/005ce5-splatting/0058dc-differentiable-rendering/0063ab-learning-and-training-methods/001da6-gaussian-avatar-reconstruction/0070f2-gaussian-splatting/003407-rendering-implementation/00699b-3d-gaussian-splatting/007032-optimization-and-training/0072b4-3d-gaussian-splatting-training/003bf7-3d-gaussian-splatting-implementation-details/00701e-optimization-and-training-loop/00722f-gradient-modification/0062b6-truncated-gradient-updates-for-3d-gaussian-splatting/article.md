# Truncated Gradient Updates for 3D Gaussian Splatting

Also called: TruncGradGS

## Overview
Truncated Gradient Updates is a gradient modification technique that applies a surrogate "truncated" gradient to 'dead' Gaussians (those with low opacity) to enable long-range movement and convergence, addressing vanishing gradients in dynamic scenes or poor initialization [S1]. It sets itself apart from standard 3D Gaussian Splatting by intercepting the backward pass to provide gradient signals to primitives that are rasterization-culled due to distance or low density, while distinguishing itself from siblings like Bézier-boundary gradient approximation by focusing specifically on the Mahalanobis distance threshold and opacity-based dead Gaussian filtering to prevent divergence [S1].

## How it works

### Inputs and Outputs
*   **Inputs:**
    *   Gaussian parameters: Mean $\mu \in \mathbb{R}^3$, Opacity $\sigma \in [0, 1]$, Alpha $\alpha$ (derived from density and scale).
    *   Pixel center $p \in \mathbb{R}^2$ in the image plane.
    *   Loss gradient with respect to alpha: $\frac{\partial L}{\partial \alpha_i}$.
    *   Pruning threshold $\tau_{pruning}$ (typically used in Adaptive Density Control).
    *   Slope parameter $m$ controlling the decay of the truncated gradient.
    *   Coefficients $a, b, c$ derived from the Gaussian covariance structure (Eq. 8 of [S1]).
*   **Outputs:**
    *   Modified gradient $\tilde{\nabla}_\mu$ for the Gaussian mean $\mu$.
    *   Modified gradient $\tilde{\nabla}_\sigma$ for the opacity $\sigma$ (only in specific low-opacity cases).

### Data Structures
The method operates on the standard 3D Gaussian Splatting data structures. Key fields accessed during the backward pass include:
*   `mu`: 3D mean position.
*   `sigma`: Opacity scalar.
*   `alpha`: Alpha value (often $\alpha = \sigma \cdot \text{density}$).
*   `covariance`: Used to derive $a, b, c$ for the Mahalanobis distance.

### Numbered Steps

1.  **Calculate Squared Mahalanobis Distance:**
    Compute the squared Mahalanobis distance $r_s$ between the Gaussian mean $\mu$ and the pixel center $p$ in the image plane:
    $$ r_s = \|\mu - p\|_2^2 $$
    The threshold condition for truncation relies on a value $K$ defined as:
    $$ K = a(\mu_x - p_x)^2 + 2b(\mu_x - p_x)(\mu_y - p_y) + c(\mu_y - p_y)^2 $$
    where $a, b, c$ are coefficients from the Gaussian's 2D projection covariance [S1].

2.  **Determine Gradient Update Path:**
    Evaluate the product of opacity $\sigma$ and alpha $\alpha$ against the threshold $1/255$ and compare $r_s$ against a cut-off threshold $\tau$ (derived from $K$ and the slope $m$). Three paths exist:
    *   **Normal Path:** If $\sigma\alpha \ge 1/255$ (high-pass filter), the Gaussian receives standard gradients from the loss. No truncation is applied [S1].
    *   **Truncated Path:** If $\sigma\alpha < 1/255$ (low density or low opacity) AND the Mahalanobis distance is above the threshold ($r_s > \tau$), the Gaussian receives the truncated gradient.
        *   If the low contribution is due to low density $\alpha$, compute the truncated gradient for the mean $\mu$.
        *   If the low contribution is due to low opacity $\sigma$, compute the gradient *only* for the opacity $\sigma$ [S1].
    *   **Skip Path:** If $\sigma\alpha < 1/255$ AND the Mahalanobis distance is below the threshold ($r_s < \tau$), the Gaussian receives no gradients [S1].

3.  **Check Gradient Sign:**
    Only apply the truncated gradient if $\frac{\partial L}{\partial \alpha_i} < 0$. If the gradient is positive, applying the truncated gradient would cause a long-range push, making Gaussians repel each other [S1].

4.  **Compute Truncated Gradient:**
    For Gaussians on the Truncated Path, the cut-off gradient is defined as $G'(x)$, where $x$ is the solution to $G(x) = \tau$. The solution for the displacement is given by:
    $$ r_s = \|\mu - p\|_2^2 \left( 1 \pm \sqrt{\frac{-2 \log \tau}{K}} \right) $$
    The choice of slope $m$ influences the range of action: a lower $m$ yields a near-constant long-range gradient approximating the cut-off gradient, while a higher $m$ yields a sharp decay from the cut-off gradient [S1].

5.  **Apply Dead Gaussian Filter:**
    To prevent divergence, the truncated gradient is applied **only** to "dead" Gaussians, defined as those where $\sigma_i < \tau_{pruning}$. This prevents the accumulated truncated gradient from surpassing the true gradient for well-fitted Gaussians [S1].

6.  **Interleaved Optimization:**
    Alternate between optimization via the truncated gradient and normal optimization with Adaptive Density Control (ADC) turned on. This prevents perturbations to the accumulated viewspace gradient from affecting ADC [S1].

7.  **Delay Pruning:**
    Delay the pruning of primitives until the end of training. This allows dead Gaussians to receive long-range gradients and converge to distant pixels before being pruned [S1].

8.  **Expand Tile-Coverage Radius:**
    Expand the tile-coverage radius of each dead Gaussian if its radius is below a threshold. This addresses tile-based culling of the rasterizer, enabling long-distance pixel error contributions using the truncated gradient field [S1].

9.  **Dynamic Scene Handling (Optional):**
    For dynamic methods, create viewpoint-dependent motion masks via frame-difference and image processing on input videos. Ignore contributions of static pixels to reduce computation [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_truncated_gradient(gaussian, pixel, loss_grad_alpha, tau_pruning, m, a, b, c):
    """
    Computes the modified gradient for a Gaussian based on truncated gradient logic.
    
    Args:
        gaussian: Object containing mu (3D), sigma (opacity), alpha (alpha/density product).
        pixel: 2D pixel center (px, py).
        loss_grad_alpha: Gradient of loss w.r.t alpha (dL/d_alpha).
        tau_pruning: Pruning threshold for opacity.
        m: Slope parameter for truncated gradient decay.
        a, b, c: Coefficients from Gaussian covariance projection.
        
    Returns:
        grad_mu: Modified gradient for mean.
        grad_sigma: Modified gradient for opacity.
    """
    # Step 1: Calculate Mahalanobis-related term K
    dx = gaussian.mu[0] - pixel[0]
    dy = gaussian.mu[1] - pixel[1]
    K = a * dx**2 + 2 * b * dx * dy + c * dy**2
    
    # Step 2: Determine Path
    # Threshold for high-pass filter
    alpha_threshold = 1.0 / 255.0
    
    # Check if Gaussian is "dead" (low opacity)
    is_dead = gaussian.sigma < tau_pruning
    
    # Check if Gaussian contributes significantly
    contributes = (gaussian.sigma * gaussian.alpha) >= alpha_threshold
    
    # Initialize gradients
    grad_mu = 0.0
    grad_sigma = 0.0
    
    if contributes:
        # Normal Path: Return standard gradients (handled by upstream backprop)
        # This function is only called for modification, so we return 0 or pass through
        return None, None 
        
    else:
        # Gaussian does not contribute via standard rasterization
        # Check Mahalanobis distance threshold condition
        # The cut-off threshold tau is implicitly handled by the distance check
        # In practice, we check if distance is "large enough" to warrant truncation
        # Based on [S1], if distance < threshold, it's a Skip Path
        
        # Simplified logic based on [S1] description:
        # Truncated Path: sigma*alpha < 1/255 AND distance > threshold
        # Skip Path: sigma*alpha < 1/255 AND distance < threshold
        
        # We need to determine if distance is above threshold.
        # The threshold is derived from tau and K. 
        # For implementation, we assume a function `is_distance_above_threshold` exists
        # which checks if r_s > tau(K, m).
        
        if is_distance_above_threshold(K, m):
            # Truncated Path
            if not is_dead:
                # Only apply to dead Gaussians to prevent divergence [S1]
                return None, None
            
            # Check sign of alpha gradient [S1]
            if loss_grad_alpha >= 0:
                # Positive gradient causes repulsion; do not apply truncated gradient
                return None, None
            
            # Compute Truncated Gradient
            # The cut-off gradient G'(x) where x is solution to G(x) = tau
            # The displacement is proportional to the gradient of the surrogate loss
            
            # Calculate the magnitude based on slope m and distance
            # Lower m -> near-constant long-range gradient
            # Higher m -> sharp decay
            
            # Note: Exact formula for G'(x) requires solving G(x)=tau.
            # The source gives the solution for r_s:
            # r_s = ||mu - p||^2 * (1 +/- sqrt(-2 log(tau) / K))
            # The gradient direction is towards the pixel p.
            
            direction = pixel - gaussian.mu[:2] # 2D projection of mean
            dist = np.linalg.norm(direction)
            if dist == 0:
                return 0.0, 0.0
                
            unit_direction = direction / dist
            
            # Compute the truncated gradient magnitude
            # This is a surrogate gradient. The exact scaling depends on the loss function L.
            # Typically, it scales with the loss value or a constant factor related to tau.
            # [S1] defines the cut-off gradient as G'(x).
            
            # Placeholder for the specific derivative of the surrogate loss function
            # In practice, this is often a constant or linearly decaying value
            truncated_grad_mag = compute_truncated_gradient_magnitude(m, K, tau_pruning)
            
            grad_mu = truncated_grad_mag * unit_direction
            
            # If low opacity is the cause, also compute gradient for sigma
            # [S1]: "In the latter case, we only compute the gradient for the opacity sigma."
            # This implies if alpha is high but sigma is low, we might only update sigma.
            # However, the condition is sigma*alpha < 1/255.
            # If alpha is high, sigma must be very low.
            if gaussian.alpha > 0: # Check if alpha is the limiting factor or sigma
                 # If the low contribution is primarily due to low sigma
                 # We compute grad_sigma.
                 # The gradient for sigma is typically the loss gradient w.r.t alpha
                 # scaled by the derivative of alpha w.r.t sigma.
                 grad_sigma = loss_grad_alpha * gaussian.alpha # Simplified chain rule
                 
            return grad_mu, grad_sigma
            
        else:
            # Skip Path: No gradients
            return None, None

def is_distance_above_threshold(K, m):
    """
    Determines if the Mahalanobis distance is above the cut-off threshold.
    The threshold depends on K and the slope m.
    """
    # The source states: "The choice of slope m influences the range of action...
    # a lower value of m yields a near-constant long-range gradient...
    # a higher value of m yields a sharp decay."
    # The threshold tau is the point where the gradient is cut off.
    # We assume a threshold function tau(K, m).
    # If the actual squared distance r_s > tau, we apply truncated gradient.
    
    # Since the exact tau function is not explicitly given as a simple formula
    # in the provided text (it refers to Eq 8 and supplementary material),
    # we note that this check is implementation-specific.
    # However, the condition is simply: r_s > tau.
    
    # For the purpose of this article, we assume the rasterizer or a helper
    # provides the threshold.
    return True # Placeholder: In real implementation, compare r_s with tau(K, m)

def compute_truncated_gradient_magnitude(m, K, tau_pruning):
    """
    Computes the magnitude of the truncated gradient.
    """
    # The source defines the cut-off gradient as G'(x).
    # This is likely a constant or a function of m.
    # A lower m yields a near-constant gradient.
    # We assume a base magnitude related to the loss or a hyperparameter.
    return 1.0 / m # Simplified approximation for the article
```

### Parameters
*   **Slope parameter $m$:** Influences the range of action of the truncated gradient in the image plane. A lower $m$ yields a near-constant long-range gradient approximating the cut-off gradient; a higher $m$ yields a sharp decay from the cut-off gradient [S1].
*   **Pruning threshold $\tau_{pruning}$:** Used to identify "dead" Gaussians. Only these Gaussians receive truncated gradients to prevent divergence [S1].
*   **Threshold $1/255$:** Used to distinguish between high-pass (normal) and low-pass (truncated/skip) paths [S1].

### Edge Cases and Pitfalls
*   **Model Divergence:** Accumulated truncated gradients may surpass the true gradient, especially for "very bad fits" (no normal gradients) or "good fits" (true gradient near zero). This is prevented by applying the truncated gradient **only** for dead Gaussians ($\sigma_i < \tau_{pruning}$) [S1].
*   **Repulsion:** If the sign of $\frac{\partial L}{\partial \alpha_i}$ is positive, applying the truncated gradient causes a long-range push, making Gaussians repel each other. The method checks for negative gradients before applying [S1].
*   **Tile Culling:** Dead Gaussians with small radii may be culled by the tile-based rasterizer, preventing them from receiving any gradient. Expanding the tile-coverage radius for dead Gaussians addresses this [S1].
*   **Performance Overhead:** The method is 1–10× slower to train than the baseline, depending on the scene and primitive count. Inference speed is unaffected [S1].

## Operating characteristics

*   **Time Complexity:** Training is 1–10× slower than the baseline 3DGS due to the additional gradient computation and radius expansion for dead Gaussians [S1].
*   **Memory Complexity:** No significant additional memory overhead compared to standard 3DGS, as it modifies the gradient computation rather than adding new data structures.
*   **Measured Performance:**
    *   **Static Scene (Mip-NeRF360, Random Init):** 3DGS + Truncated Gradient achieves PSNR 25.16 (vs 24.74), SSIM 0.8511 (vs 0.8351), LPIPS 0.1974 (vs 0.2106), and #GS 0.799M (vs 1.126M) [S1].
    *   **Dynamic Scene (Neural 3D Video):** 4DGS + Truncated Gradient achieves PSNR 31.70 (vs 30.22), SSIM 0.9493 (vs 0.9443), LPIPS 0.1339 (vs 0.1360), and #GS 1.733M (vs 2.806M) [S1].
*   **Scaling:** The method scales with the number of primitives, but the overhead is mitigated by only applying the truncated gradient to dead Gaussians and using motion masks for dynamic scenes [S1].
*   **Strengths:** Improves reconstruction quality for poorly initialized scenes and dynamic scenes with large spatio-temporal divergence. Reduces the number of Gaussians needed for high-quality reconstruction.
*   **Limitations:** Slower training time. Requires careful tuning of the slope parameter $m$ and pruning threshold. Not effective if the initial camera poses are extremely poor (though it helps convergence).
*   **When to Use:** Use for static and dynamic scene reconstruction where standard 3DGS fails to converge due to vanishing gradients or poor initialization. It is a plug-and-play module for 3DGS, 2DGS, 4DGS, CEM-4DGS, and 4DScaffold [S1].
*   **When Not to Use:** Avoid if training time is a critical constraint and the scene is well-initialized. Avoid if the scene has no dynamic elements and standard 3DGS converges well.

## Background

*   **Origin:** The method was introduced by T. Morales, N.-Q. Le-Pham, R. Atkins & B.-S. Hua in the paper "TruncGradGS: Improved 3D Gaussian Splatting via Truncated Gradient Updates" [S1].
*   **History:** It was published in the Proceedings of Eurographics - The European Association for Computer Graphics in 2026 [S1].
*   **Development:** The method builds on the standard 3D Gaussian Splatting framework, addressing its limitations in handling "dead" Gaussians and long-range optimization. It integrates with existing dynamic Gaussian Splatting methods like 4DGS and CEM-4DGS [S1].

## Sources

[S1] TruncGradGS: Improved 3D Gaussian Splatting via Truncated Gradient Updates, https://arxiv.org/abs/2609.03534v1, fetched 2026-10-26
