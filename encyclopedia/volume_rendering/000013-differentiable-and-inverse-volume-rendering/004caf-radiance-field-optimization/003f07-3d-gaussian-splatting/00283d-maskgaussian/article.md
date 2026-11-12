# MaskGaussian

Also called: Masked Gaussian Splatting

## Overview
MaskGaussian is a differentiable pruning method for 3D Gaussian Splatting that models Gaussians as probabilistic entities rather than deterministically removing them based on static importance scores. It solves the problem of sub-optimal reconstruction caused by early pruning by introducing **masked-rasterization**, which allows gradients to flow to "masked-out" (inactive) Gaussians, enabling them to be re-evaluated and potentially reactivated as the scene evolves. Unlike sibling methods like [Compact3DGS](1398-compact3dgs.md) which use deterministic pruning or multiply masks with opacity/scale (blocking gradients), MaskGaussian applies masks during the alpha-blending stage, decoupling Gaussian existence from its geometric attributes.

## How it works
MaskGaussian extends the standard 3D Gaussian Splatting pipeline by introducing learnable mask parameters for each Gaussian. The core innovation is a rasterization technique that applies these masks to the blending process, ensuring that even inactive Gaussians participate in the forward pass and receive gradient updates.

### Inputs and Outputs
*   **Inputs:**
    *   Set of $N$ 3D Gaussians, each defined by position $\mathbf{p}_i$, covariance $\mathbf{\Sigma}_i$ (or rotation $\mathbf{R}_i$ and scale $\mathbf{s}_i$), color $\mathbf{c}_i$, and opacity $\alpha_i$.
    *   Camera parameters (view and projection matrices).
    *   Learnable mask scores $\mathbf{z}_i \in \mathbb{R}^2$ for each Gaussian $i$.
*   **Outputs:**
    *   Rendered image $\mathbf{I}$.
    *   Gradients for all Gaussian attributes and mask scores.

### Data Structures
*   **Gaussian Attributes:** Standard 3DGS attributes (position, scale, rotation, SH coefficients, opacity).
*   **Mask Scores:** For each Gaussian $i$, two learnable scalar values $\mathbf{z}_i = [z_{i,0}, z_{i,1}]$. These are not probabilities but logits for a binary distribution.

### Numbered Steps

1.  **Mask Sampling:**
    For each Gaussian $i$, sample a binary mask $M_i \in \{0, 1\}$ from the mask scores $\mathbf{z}_i$ using the Gumbel-Softmax trick to ensure differentiability.
    $$ M_i = \text{GumbelSoftmax}(\mathbf{z}_i) $$
    *Note: During training, this is a soft sample; during inference or pruning evaluation, it is treated as hard binary.*

2.  **Standard Splatting (Alpha Computation):**
    Splat all $N$ Gaussians onto the image plane using standard 3DGS rasterization. Compute the splatted density $\alpha_i$ and transmittance $T_i$ for each Gaussian *without* applying the mask $M_i$ to the opacity or scale. This ensures that all Gaussians, regardless of their mask value, pass through the alpha-filtering stage and are processed by the rasterizer.
    *   $\alpha_i$: Splatted density of Gaussian $i$.
    *   $T_i$: Transmittance reaching Gaussian $i$, defined as $T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)$ (assuming sorted order).

3.  **Masked Rasterization (Color and Transmittance Blending):**
    Compute the final pixel color $\mathbf{c}(\mathbf{x})$ and update transmittance using the masks $M_i$. The mask is applied directly to the color accumulation and transmittance attenuation terms.
    
    **Color Accumulation:**
    $$ \mathbf{c}(\mathbf{x}) = \sum_{i=1}^N M_i \cdot \mathbf{c}_i \cdot \alpha_i \cdot T_i $$
    
    **Transmittance Update:**
    $$ T_{i+1} = M_i \cdot (1 - \alpha_i) \cdot T_i + (1 - M_i) \cdot T_i $$
    
    *   If $M_i = 1$: The Gaussian contributes its color $\mathbf{c}_i \cdot \alpha_i \cdot T_i$ and attenuates transmittance by $(1-\alpha_i)$.
    *   If $M_i = 0$: The Gaussian contributes no color, and transmittance is unchanged ($T_{i+1} = T_i$).

4.  **Loss Computation:**
    Compute the total loss $L$:
    $$ L = L_{render} + \lambda_m \cdot L_m $$
    Where:
    *   $L_{render}$ is the standard rendering loss (e.g., L1 + SSIM) between the rendered image and the ground truth.
    *   $L_m$ is a squared loss on the average number of active Gaussians to constrain the model size:
        $$ L_m = \left( \frac{1}{N} \sum_{i=1}^N M_i \right)^2 $$
    *   $\lambda_m$ is a balancing hyperparameter.

5.  **Backpropagation:**
    Compute gradients for the mask scores $\mathbf{z}_i$. The gradient of the loss with respect to the mask $M_i$ is derived as:
    $$ \frac{\partial L}{\partial M_i} = \alpha_i \cdot T_i \cdot \frac{\partial L}{\partial \mathbf{c}(\mathbf{x})} \cdot (\mathbf{c}_i - \mathbf{b}_{i+1}) $$
    Where $\mathbf{b}_{i+1}$ is the background color rendered behind the $i$-th Gaussian:
    $$ \mathbf{b}_{i+1} = \sum_{j=i+1}^N M_j \cdot \mathbf{c}_j \cdot \alpha_j \cdot T_j $$
    The gradient flows through the Gumbel-Softmax operation to update $\mathbf{z}_i$. This allows $M_i=0$ Gaussians to receive meaningful gradients based on their potential contribution ($\mathbf{c}_i$) versus the background ($\mathbf{b}_{i+1}$).

6.  **Pruning Procedure:**
    Periodically, hard prune Gaussians with near-zero probability of existence.
    *   Sample each Gaussian's mask 10 times.
    *   If a Gaussian is never sampled (i.e., $M_i=0$ in all 10 trials), remove it from the scene.
    *   This is applied at every densification step and every 1000 iterations after densification.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def gumbel_softmax_sample(logits, temperature=1.0):
    """
    Samples from a Gumbel-Softmax distribution.
    logits: Tensor of shape (N, 2)
    Returns: Tensor of shape (N, 2) with one-hot like values, but differentiable.
    """
    gumbels = -torch.empty_like(logits).exponential_().log()
    gumbels = (gumbels + logits) / temperature
    y_soft = F.softmax(gumbels, dim=-1)
    # Straight-through estimator for binary mask
    index = y_soft.max(1, keepdim=True)[1]
    y_hard = torch.zeros_like(logits, memory_format=torch.contiguous_format).scatter_(1, index, 1.0)
    ret = y_hard - y_soft.detach() + y_soft
    return ret

def masked_rasterization(gaussians, camera, mask_scores, lambda_m, iteration):
    """
    Forward pass for MaskGaussian.
    
    Args:
        gaussians: Object containing attributes (pos, scale, rot, sh, opacity) for N Gaussians.
        camera: Camera parameters.
        mask_scores: Tensor of shape (N, 2) containing learnable mask logits.
        lambda_m: Hyperparameter for mask regularization.
        iteration: Current training iteration.
        
    Returns:
        rendered_image: Tensor of shape (H, W, 3)
        loss: Scalar loss value
    """
    N = gaussians.num_gaussians
    
    # Step 1: Sample masks
    # Use straight-through estimator for binary sampling during training
    # In practice, this might be done per-pixel or per-gaussian globally. 
    # The paper implies per-gaussian mask M_i applied to all pixels.
    # We assume M_i is sampled once per Gaussian per iteration.
    mask_samples = gumbel_softmax_sample(mask_scores) # Shape (N, 2)
    M = mask_samples[:, 1] # Probability of being active (index 1)
    # For the forward pass, we use the sampled binary mask
    # Note: The paper uses M_i in {0,1}. We hard sample for forward, soft for backward via straight-through
    M_binary = (M > 0.5).float() 
    
    # Step 2: Standard Splatting (Alpha computation without mask)
    # This part is handled by the standard 3DGS rasterizer
    # We need to get alpha_i and T_i for all Gaussians projected to screen
    # Assuming a function that returns sorted list of Gaussians per pixel
    # For simplicity, we outline the logic assuming we have per-pixel sorted Gaussians
    
    # Let's assume we have a rasterizer that returns:
    # alpha: Tensor (H, W, N_max) - splatted alpha for each Gaussian at each pixel
    # depth: Tensor (H, W, N_max) - depth for sorting
    # We need to sort and compute T_i
    
    # Standard 3DGS rasterization step
    # This is a high-level representation of the CUDA kernel logic
    alpha, depth, indices = standard_splat(gaussians, camera) # Shape (H, W, N)
    
    # Sort Gaussians by depth (back-to-front)
    # In 3DGS, this is done efficiently in CUDA. Here we assume sorted order.
    # alpha_sorted, indices_sorted = sort_by_depth(alpha, depth)
    
    # Compute Transmittance T_i for all Gaussians (without mask)
    # T_i = product(1 - alpha_j) for j < i
    # We compute this cumulatively
    alpha_flat = alpha.view(-1, alpha.shape[-1]) # (H*W, N)
    T_flat = torch.cumprod(1 - alpha_flat, dim=1)
    T_flat = torch.cat([torch.ones_like(T_flat[:, :1]), T_flat[:, :-1]], dim=1) # T_1 = 1
    
    # Reshape back
    T = T_flat.view(alpha.shape) # (H, W, N)
    
    # Step 3: Masked Rasterization
    # Expand M_binary to (H, W, N)
    M_expanded = M_binary.view(1, 1, -1).expand_as(alpha)
    
    # Get colors for each Gaussian
    # c_i is the color of Gaussian i at pixel (u,v)
    # This is computed by projecting SH coefficients
    colors = compute_sh_color(gaussians, camera, indices) # (H, W, N, 3)
    
    # Apply Mask to Color Accumulation and Transmittance
    # c(x) = sum( M_i * c_i * alpha_i * T_i )
    # T_{i+1} = M_i * (1 - alpha_i) * T_i + (1 - M_i) * T_i
    
    # We can compute the final color directly
    # Note: T_i in the formula is the transmittance BEFORE Gaussian i
    # The contribution of Gaussian i is M_i * c_i * alpha_i * T_i
    
    contribution = M_expanded * colors * alpha * T # (H, W, N, 3)
    rendered_image = torch.sum(contribution, dim=2) # (H, W, 3)
    
    # Step 4: Loss Computation
    # L_render
    # Assume ground_truth is available
    L_render = l1_loss(rendered_image, ground_truth) + ssim_loss(rendered_image, ground_truth)
    
    # L_m: Squared loss on average number of Gaussians
    # M_i is the sampled mask. We use the soft probability for the loss gradient
    # Or the hard mask? The paper says "squared loss on the average number of Gaussians"
    # and uses M_i in the formula. It likely uses the soft probability for the loss
    # to allow gradient flow, or the hard mask if it's a constraint.
    # Given it's a regularization term, using the soft probability (M) is more standard for differentiable pruning.
    # However, the paper defines L_m = (1/N sum M_i)^2.
    # Let's use the soft probability M (from gumbel_softmax) for the loss.
    avg_M = torch.mean(M)
    L_m = avg_M ** 2
    
    loss = L_render + lambda_m * L_m
    
    # Step 5: Pruning (if applicable)
    # This is done periodically, not every step.
    # We return the mask scores and current masks for pruning logic.
    
    return rendered_image, loss, M_binary

def compute_mask_gradient(rendered_image, ground_truth, alpha, T, colors, M_binary):
    """
    Computes the gradient for the mask M_i.
    dL/dM_i = alpha_i * T_i * dL/dc(x) * (c_i - b_{i+1})
    """
    # dL/dc(x) is the gradient from the loss w.r.t the rendered image
    dL_dc = (rendered_image - ground_truth) # Simplified for L1
    
    # Compute b_{i+1}: Background color behind Gaussian i
    # b_{i+1} = sum_{j=i+1}^N M_j * c_j * alpha_j * T_j
    # We need to compute this for each Gaussian i
    
    # Contribution of all Gaussians
    contribution = M_binary * colors * alpha * T # (H, W, N, 3)
    
    # Cumulative sum from back to front to get b_{i+1}
    # b_{i+1} is the sum of contributions from j > i
    # Reverse the sum
    reversed_contribution = torch.flip(contribution, [2])
    cumsum_reversed = torch.cumsum(reversed_contribution, dim=2)
    b = torch.flip(cumsum_reversed, [2])
    
    # Shift b to get b_{i+1} (sum of j > i)
    # b[i] includes contribution of i. We want sum of j > i.
    # So b_{i+1} = b[i+1:] padded with 0 at the end
    b_next = torch.cat([b[:, :, 1:], torch.zeros_like(b[:, :, :1])], dim=2)
    
    # Gradient dL/dM_i
    # dL/dM_i = alpha_i * T_i * dL/dc(x) * (c_i - b_{i+1})
    # Note: dL/dc(x) is a vector (H, W, 3). We need to broadcast correctly.
    # The gradient is a scalar per Gaussian per pixel? 
    # Actually, M_i is per Gaussian, not per pixel. 
    # The paper says "sample a mask for each Gaussian".
    # So M_i is a scalar per Gaussian.
    # The gradient dL/dM_i is the sum of gradients over all pixels where Gaussian i is visible.
    
    # Let's compute the gradient per pixel first
    # dL/dM_i_pixel = alpha_i * T_i * dL_dc * (c_i - b_{i+1})
    # This is a vector (H, W, 3)
    
    term = alpha * T.unsqueeze(-1) * dL_dc * (colors - b_next) # (H, W, N, 3)
    
    # Sum over pixels to get gradient for each Gaussian
    # We need to know which pixels each Gaussian covers.
    # This is implicitly handled by the rasterization.
    # In practice, this gradient is accumulated in the CUDA kernel.
    
    # For the purpose of the article, we state the formula.
    # The gradient is backpropagated through the Gumbel-Softmax to update mask_scores.
    pass
```

### Parameters
*   **$\lambda_m$**: Balancing weight for the squared loss $L_m$.
    *   Typical values:
        *   0.1 for iterations 19,000–20,000.
        *   0.0005 or 0.001 for iterations 0–30,000.
*   **Pruning Frequency**: Every densification step and every 1000 iterations after densification.
*   **Sampling Count for Pruning**: 10 samples per Gaussian to determine if it should be removed.

### Edge Cases and Pitfalls
*   **Gradient Blocking**: If masks are applied to opacity or scale (setting them to zero), the Gaussian's density $\alpha$ becomes zero. This causes the Gaussian to be filtered out before rasterization, preventing gradient updates. MaskGaussian avoids this by applying masks during blending.
*   **Outdated Gaussians**: If masked Gaussians do not receive gradients, they remain static and may become outdated as the scene evolves. MaskGaussian ensures they receive gradients via masked-rasterization.
*   **Fine Gaussians**: Methods that bind mask gradients to scales/opacity may disadvantage fine Gaussians if they receive negative gradients intended to shrink them. MaskGaussian decouples these.

## Subtopics

### [Sparsity Regularization for Pruning](9357-sparsity-regularization-for-pruning.md)
A regularization technique applied to learnable mask parameters in 3D Gaussian Splatting to guide the pruning process and encourage condensed scene representations. It solves the problem of inefficient storage and rendering by promoting both fine-grained sparsity (removing non-essential individual Gaussians) and structural coherence (removing entire anchors). Unlike standard L1-style penalties which may leave anchors with only a few sparsely active Gaussians, hindering compactness, this method uses a group L2 norm to encourage switching off entire anchors cohesively. It builds on [MaskGaussian](4242-maskgaussian.md) by providing the optimization signal that determines which probabilistic entities should be deactivated.

## Operating characteristics
*   **Time Complexity**: Similar to standard 3DGS rasterization, with minor overhead for mask sampling and blending. The pruning step adds negligible overhead as it is performed periodically.
*   **Memory Complexity**: Adds $2N$ learnable parameters (mask scores) for $N$ Gaussians.
*   **Measured Performance**:
    *   **Mip-NeRF360**: 27.43 PSNR, 1.205M Gaussians, 384.7 FPS (vs. 3DGS: 27.45 PSNR, 3.204M Gaussians, 187.8 FPS). Pruning ratio: 62.4%. Speedup: 2.05x.
    *   **Tanks & Temples**: 23.72 PSNR, 0.590M Gaussians, 558.3 FPS (vs. 3DGS: 23.74 PSNR, 1.825M Gaussians, 254.5 FPS). Pruning ratio: 67.7%. Speedup: 2.19x.
    *   **Deep Blending**: 29.69 PSNR, 0.694M Gaussians, 637.1 FPS (vs. 3DGS: 29.53 PSNR, 2.815M Gaussians, 201.3 FPS). Pruning ratio: 75.3%. Speedup: 3.16x.
*   **Strengths**:
    *   Achieves significant pruning ratios (62-75%) with minimal PSNR loss (0.02 dB).
    *   Enables real-time rendering with fewer Gaussians.
    *   Allows dynamic re-evaluation of Gaussians, preventing premature pruning.
*   **Limitations**:
    *   Requires modification of the rasterization code to implement masked-rasterization.
    *   Hyperparameter $\lambda_m$ requires tuning for different scenes and training phases.

## Background
MaskGaussian was introduced in the paper "MaskGaussian: Adaptive 3D Gaussian Representation from Probabilistic Masks" (CVPR 2025) [S1]. It builds upon [Compact3DGS](1398-compact3dgs.md) by extending its pruning approach from deterministic to probabilistic. The method addresses the limitations of previous mask-based pruning methods that multiply masks with Gaussian attributes (opacity/scale), which block gradient flow and lead to sub-optimal reconstruction. By treating pruning as a stochastic sampling process and introducing masked-rasterization, MaskGaussian allows for adaptive pruning that evolves with the scene.

## Sources
[S1] MaskGaussian: Adaptive 3D Gaussian Representation from Probabilistic Masks, https://doi.org/10.1109/cvpr52734.2025.00072, fetched 2026-10-30
