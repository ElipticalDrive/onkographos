# Variance-aware Adaptive Pixel Sampling

Also called: Adaptive pixel sampling for inverse rendering, Image-Space Adaptive Sampling, Pixel-level mini-batching, Variance-minimizing pixel sampling

## Overview
Variance-aware adaptive pixel sampling is a technique for inverse rendering that accelerates gradient-based optimization by dynamically allocating a fixed per-iteration pixel sampling budget based on the estimated variance of the loss gradient contributions [S1, unofficial]. Unlike uniform mini-batching or sampling proportional only to loss gradient magnitude, this method accounts for the variance of the differentiable rendering derivative, minimizing the total variance of the gradient estimator [S1, unofficial]. It acts as a form of pixel-level mini-batching, rendering only a subset of pixels per iteration while maintaining low gradient noise [S1, unofficial].

## How it works
The method operates within an inverse rendering pipeline where the goal is to minimize a loss $\mathcal{L}$ with respect to scene parameters $\boldsymbol{\theta}$ [S1, unofficial]. It assumes the availability of an adjoint method or equivalent mechanism to compute the derivative of the loss with respect to the image, $\partial_{\mathbf{I}} \mathcal{L}$, and the derivative of the image with respect to parameters, $d\mathbf{I}/d\boldsymbol{\theta}$ [S1, unofficial].

### Inputs and Outputs
**Inputs:**
- Scene parameters $\boldsymbol{\theta} \in \mathbb{R}^{m_\theta}$.
- Target image $\mathbf{I}^{\text{target}}$.
- Per-iteration sampling budget $N_{\text{pix}}$ (number of pixels to render per iteration).
- Current estimate of pixel sampling probabilities $p_j$ (initialized uniformly or from previous iteration).

**Outputs:**
- Updated scene parameters $\boldsymbol{\theta}$ (via gradient descent step).
- Updated sampling probabilities $p_j$ for the next iteration.

### Data Structures
- **Pixel Grid:** A 2D array of size $H \times W$ (image resolution).
- **Sampling Weights:** A 2D array of size $H \times W$ storing the unnormalized weights $w_j$ for each pixel.
- **Probability Map:** A 2D array of size $H \times W$ storing normalized probabilities $p_j$.
- **Gradient Estimators:** Buffers to store $\langle (\partial_{\mathbf{I}} \mathcal{L})_j \rangle$ and $\langle dI_j/d\theta \rangle$ (or their squared second moments) for variance estimation.

### Numbered Steps

1. **Primal Pass (Image Rendering):**
   Render the current scene parameters $\boldsymbol{\theta}$ to obtain the current image $\mathbf{I}(\boldsymbol{\theta})$. This pass may use a higher sample budget per pixel to ensure accurate image values for loss computation and weight estimation [S2].

2. **Compute Loss Gradient Derivatives:**
   Compute the derivative of the loss with respect to the image, $\partial_{\mathbf{I}} \mathcal{L}$, for all pixels. This is typically a simple analytical derivative based on the loss function (e.g., MSE) and the target image [S1, unofficial].

3. **Estimate Rendering Gradient Variance:**
   Estimate the second moment of the rendering derivative, $p_{j2}$, for each pixel. This requires rendering the gradient $dI_j/d\theta$ (or its components) for a subset of pixels or using a low-cost approximation. In practice, this is often estimated by rendering the adjoint radiance or using a running average of squared gradient estimates from previous iterations [S1, unofficial][S2].
   - For scalar $\theta$: $p_{j2} \approx \mathbb{E}[\langle dI_j/d\theta \rangle^2]$.
   - For vector $\boldsymbol{\theta}$: $p_{j2} \approx \sum_{k=1}^{m_\theta} \mathbb{E}[\langle \partial I_j / \partial \theta_k \rangle^2]$.

4. **Compute Sampling Weights:**
   Calculate the unnormalized weight $w_j$ for each pixel $j$ using the formula:
   $$ w_j = \sqrt{p_{j1} p_{j2}} $$
   where $p_{j1} = \mathbb{E}[\langle (\partial_{\mathbf{I}} \mathcal{L})_j \rangle^2]$ is the second moment of the loss derivative estimator. Note that $p_{j1}$ can be computed analytically from the loss function and current image, while $p_{j2}$ is estimated as in Step 3 [S1, unofficial].
   - *Alternative (Simpler) Approach:* Some implementations approximate $p_{j1}$ using the magnitude of the loss gradient $|\partial_{\mathbf{I}} \mathcal{L}|_j$ and $p_{j2}$ using the magnitude of the adjoint radiance $|\partial_I \mathcal{L}|_j$ (if available from an adjoint pass), leading to weights proportional to $|\partial_{\mathbf{I}} \mathcal{L}|_j \cdot |\partial_I \mathcal{L}|_j$ or similar heuristics [S2]. However, the theoretically optimal form uses the second moments [S1, unofficial].

5. **Normalize Probabilities:**
   Normalize the weights to obtain sampling probabilities:
   $$ p_j = \frac{w_j}{\sum_{k=1}^{m_I} w_k} $$
   where $m_I$ is the total number of pixels [S1, unofficial].

6. **Sample Pixels:**
   Randomly select $N_{\text{pix}}$ pixels according to the probability distribution $p_j$. This is the "mini-batch" of pixels to render in this iteration [S1, unofficial].

7. **Adjoint/Gradient Pass:**
   For each sampled pixel $j$, render the gradient $dI_j/d\theta$ (or compute it via the adjoint method) to obtain the estimator $\langle dI_j/d\theta \rangle$ [S1, unofficial].

8. **Estimate Gradient:**
   Compute the unbiased gradient estimator for the selected pixels:
   $$ \left\langle \frac{d\mathcal{L}}{d\boldsymbol{\theta}} \right\rangle = \frac{1}{N_{\text{pix}}} \sum_{j \in \text{sampled}} \frac{\langle (\partial_{\mathbf{I}} \mathcal{L})_j \rangle}{p_j} \left\langle \frac{dI_j}{d\boldsymbol{\theta}} \right\rangle $$
   Note: The factor $1/N_{\text{pix}}$ is included if the sum is averaged over the batch. If the sum is not averaged, the estimator is $\sum_{j \in \text{sampled}} \frac{\langle (\partial_{\mathbf{I}} \mathcal{L})_j \rangle}{p_j} \left\langle \frac{dI_j}{d\boldsymbol{\theta}} \right\rangle$ [S1, unofficial].

9. **Update Parameters:**
   Use the estimated gradient to update $\boldsymbol{\theta}$ via a gradient descent step.

10. **Update Variance Estimates:**
    Update the running estimates of $p_{j1}$ and $p_{j2}$ using exponential moving averages or similar techniques to adapt to changes in the scene parameters [S1, unofficial].

### Formulas
- **Loss Minimization:**
  $$ \boldsymbol{\theta}^* = \arg \min_{\boldsymbol{\theta}} \mathcal{L} \left( \mathbf{I}(\boldsymbol{\theta}); \mathbf{I}^{\text{target}} \right) \quad (4.1) $$
- **Chain Rule:**
  $$ \frac{d\mathcal{L}}{d\boldsymbol{\theta}} = \sum_{j=1}^{m_I} (\partial_{\mathbf{I}} \mathcal{L})_j \frac{dI_j}{d\boldsymbol{\theta}} \quad (4.2) $$
- **Monte Carlo Estimator:**
  $$ \left\langle \frac{d\mathcal{L}}{d\boldsymbol{\theta}} \right\rangle = \frac{\langle (\partial_{\mathbf{I}} \mathcal{L})_j \rangle}{p_j} \left\langle \frac{dI_j}{d\boldsymbol{\theta}} \right\rangle \quad (4.4) $$
- **Optimal Probability (Scalar Case):**
  $$ p_j \propto \sqrt{p_{j1} p_{j2}} \quad (4.8) $$
  where $p_{j1} := \mathbb{E}[\langle (\partial_{\mathbf{I}} \mathcal{L})_j \rangle^2]$ and $p_{j2} := \mathbb{E}[\langle dI_j/d\theta \rangle^2]$.
- **Optimal Probability (Vector Case):**
  $$ p_j \propto \sqrt{p_{j1} p_{j2}} \quad (4.12) $$
  where $p_{j1} := \mathbb{E}[\langle (\partial_{\mathbf{I}} \mathcal{L})_j \rangle^2]$ and $p_{j2} := \sum_{k=1}^{m_\theta} \mathbb{E}[\langle \partial I_j / \partial \theta_k \rangle^2]$.

### Parameters
- **Sampling Budget ($N_{\text{pix}}$):** The number of pixels sampled per iteration. This is fixed per iteration [S1, unofficial].
- **Primal:Adjoint Sample Ratio:** The optimal ratio found in practice is 3:1 (75% of samples for primal rendering, 25% for adjoint/gradient rendering) [S2].

### Edge Cases and Pitfalls
- **Independence Assumption:** The derivation assumes that the loss derivative estimator $\langle (\partial_{\mathbf{I}} \mathcal{L})_j \rangle$ and the rendering derivative estimator $\langle dI_j/d\theta \rangle$ are independent. If they are correlated, the variance minimization derivation does not hold directly [S1, unofficial].
- **Negative Adjoint Radiance:** The adjoint radiance $\partial_I \mathcal{L}$ can have negative values. Sampling proportional to its absolute value $|\partial_I \mathcal{L}|$ introduces sign variance. While positivization (sampling positive and negative components separately) removes this variance, it was found to have little effect on overall estimation variance in practice, so the simpler absolute value method is used [S2].
- **High Variance with Uniform Sampling:** Naively reducing the sampling budget by uniformly sampling pixels results in high gradient variance, degrading performance [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def variance_aware_adaptive_sampling(
    theta, 
    I_target, 
    render_func, 
    render_grad_func, 
    loss_func, 
    loss_grad_func, 
    N_pix, 
    N_primal_ratio=0.75, 
    N_adjoint_ratio=0.25,
    alpha_ema=0.9,
    max_iter=1000,
    lr=0.01
):
    """
    Variance-aware adaptive pixel sampling for inverse rendering.
    
    Args:
        theta: Initial scene parameters.
        I_target: Target image.
        render_func: Function to render image I(theta).
        render_grad_func: Function to render gradient dI/dtheta for a pixel.
        loss_func: Function to compute loss L(I, I_target).
        loss_grad_func: Function to compute gradient dL/dI.
        N_pix: Number of pixels to sample per iteration.
        N_primal_ratio: Ratio of samples for primal pass.
        N_adjoint_ratio: Ratio of samples for adjoint pass.
        alpha_ema: Exponential moving average factor for variance estimates.
        max_iter: Maximum number of iterations.
        lr: Learning rate.
        
    Returns:
        theta: Optimized scene parameters.
    """
    H, W = I_target.shape[:2]
    m_I = H * W
    
    # Initialize variance estimates
    # p_j1: Second moment of loss derivative
    # p_j2: Second moment of rendering derivative
    p_j1 = np.ones((H, W))
    p_j2 = np.ones((H, W))
    
    # Initialize probabilities uniformly
    p = np.ones((H, W)) / m_I
    
    for iteration in range(max_iter):
        # Step 1: Primal Pass
        # Render the current image
        I_curr = render_func(theta)
        
        # Compute loss and its gradient w.r.t. image
        loss = loss_func(I_curr, I_target)
        dL_dI = loss_grad_func(I_curr, I_target)
        
        # Update p_j1 (second moment of loss derivative)
        # Using current dL_dI as an estimate
        p_j1 = alpha_ema * p_j1 + (1 - alpha_ema) * (dL_dI ** 2)
        
        # Step 2: Estimate p_j2 (second moment of rendering derivative)
        # This requires rendering gradients. To keep it efficient, 
        # we might use a subset or a low-cost approximation.
        # For simplicity, we assume render_grad_func can return the squared gradient magnitude
        # or we approximate it using the adjoint radiance from a previous step.
        # Here, we use a placeholder for the actual gradient rendering.
        # In practice, this might be done with a small number of samples per pixel.
        dI_dtheta_est = render_grad_func(theta) # Returns HxW array of gradient magnitudes or squared magnitudes
        p_j2 = alpha_ema * p_j2 + (1 - alpha_ema) * (dI_dtheta_est ** 2)
        
        # Step 3: Compute Sampling Weights
        # w_j = sqrt(p_j1 * p_j2)
        weights = np.sqrt(p_j1 * p_j2)
        
        # Handle potential zeros in weights to avoid division by zero
        weights = np.maximum(weights, 1e-8)
        
        # Step 4: Normalize Probabilities
        p = weights / np.sum(weights)
        
        # Step 5: Sample Pixels
        # Flatten probabilities for sampling
        p_flat = p.flatten()
        p_flat = p_flat / np.sum(p_flat) # Ensure normalization
        
        # Sample N_pix pixels
        pixel_indices = np.random.choice(m_I, size=N_pix, replace=False, p=p_flat)
        
        # Reshape to 2D
        pixel_indices_2d = np.unravel_index(pixel_indices, (H, W))
        
        # Step 6: Adjoint/Gradient Pass
        # Render gradients for sampled pixels
        grad_estimates = np.zeros_like(theta)
        
        for idx in pixel_indices:
            y, x = np.unravel_index(idx, (H, W))
            
            # Get the loss derivative for this pixel
            dL_dI_j = dL_dI[y, x]
            
            # Get the rendering derivative for this pixel
            # render_grad_func_pixel might return the gradient vector or scalar
            dI_dtheta_j = render_grad_func_pixel(theta, y, x)
            
            # Compute the unbiased gradient estimator for this pixel
            # <dL/dtheta> = (dL_dI_j / p_j) * <dI_j/dtheta>
            p_j = p[y, x]
            grad_est = (dL_dI_j / p_j) * dI_dtheta_j
            
            # Accumulate gradient
            # Note: The estimator is unbiased for the full gradient sum.
            # We need to average over the batch if we want the mean gradient.
            grad_estimates += grad_est
            
        # Average the gradient over the sampled pixels
        grad_estimates /= N_pix
        
        # Step 7: Update Parameters
        theta -= lr * grad_estimates
        
        # Optional: Update variance estimates with new gradient info
        # This can help adapt p_j2 if the gradient distribution changes significantly
        
    return theta
```

## Operating characteristics
- **Time Complexity:** $O(N_{\text{pix}} \cdot C_{\text{render}})$ per iteration, where $C_{\text{render}}$ is the cost of rendering a single pixel. This is significantly faster than full-image rendering $O(m_I \cdot C_{\text{render}})$ when $N_{\text{pix}} \ll m_I$ [S1, unofficial].
- **Memory Complexity:** $O(m_I)$ to store the probability map and variance estimates [S1, unofficial].
- **Measured Performance:** Reduces loss gradient variance by 1–2 orders of magnitude compared to uniform sampling or BRDF-only sampling [S2]. Outperforms uniform mini-batching by allowing faster convergence at equal time [S1, unofficial].
- **Strengths:** Accelerates inverse rendering by reducing sampling budget without sacrificing performance. Lowers gradient variance by accounting for rendering derivative variance [S1, unofficial].
- **Limitations:** Requires estimation of rendering derivative variance, which can be costly if not approximated. Assumes independence between loss and rendering derivative estimators [S1, unofficial].
- **When to Use:** Inverse rendering pipelines for SVBRDF reconstruction, shape optimization, and inverse volume rendering where computational cost is a bottleneck [S1, unofficial].
- **When Not to Use:** When the independence assumption is strongly violated, or when the cost of estimating $p_{j2}$ outweighs the benefits of adaptive sampling.

## Background
The method was introduced in the context of differentiable rendering and inverse rendering to address the high computational cost of rendering all pixels during optimization [S1, unofficial]. It builds on the concept of mini-batching in gradient-based optimization and applies it to the image space [S1, unofficial]. The theoretical foundation for optimal sampling probabilities is derived from minimizing the variance of the Monte Carlo estimator for the loss gradient [S1, unofficial]. The technique has been applied to various inverse rendering problems, including SVBRDF reconstruction and shape optimization [S1, unofficial].

## Sources
[S1] Variance-aware Differentiable Rendering, https://core.ac.uk/works/337282320, fetched 2026-11-02 (unofficial)
[S2] Path Sampling Methods for Differentiable Rendering, https://doi.org/10.2312/sr.20241148, fetched 2026-10-29
