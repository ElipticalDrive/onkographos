# MaskGaussian

Also called: Masked Gaussian Splatting

## Overview
MaskGaussian is a differentiable pruning method for 3D Gaussian Splatting that models Gaussian existence probabilistically rather than deterministically, mitigating sub-optimized reconstruction caused by permanent removal of components [S1]. It extends Compact3DGS by introducing a "masked-rasterization" technique that allows gradients to flow to unsampled (masked-out) Gaussians, enabling dynamic assessment of their contribution to the evolving scene [S1]. Unlike methods that multiply masks with Gaussian attributes (opacity/scale) which zero out density and prevent gradient updates, MaskGaussian applies masks during the alpha-blending stage, ensuring all Gaussians participate in rasterization and receive meaningful gradients regardless of their current mask state [S1].

## How it works
MaskGaussian operates by learning a binary mask distribution for each 3D Gaussian, representing its probability of existence. The method modifies the standard 3DGS rasterization pipeline to incorporate these masks into the color accumulation and transmittance attenuation steps, while preserving the integrity of the Gaussian's physical attributes (opacity, scale) for the alpha-filtering stage.

### Inputs and Outputs
**Inputs:**
- Set of $N$ 3D Gaussians, each defined by mean $\mu_i$, covariance $\Sigma_i$ (or scale/rotation), color $\mathbf{c}_i$, and base opacity $\alpha_i$.
- Camera parameters (intrinsics $K$, extrinsics $c2w$) for the current view.
- Learnable mask scores for each Gaussian.

**Outputs:**
- Rendered image color $\hat{C}(\mathbf{x})$ for each pixel $\mathbf{x}$.
- Updated Gaussian attributes and mask scores via backpropagation.

### Data Structures
For each Gaussian $i$, in addition to the standard 3DGS attributes, the method maintains:
- **Mask Scores**: Two learnable scalar parameters, denoted as $s_{i,0}$ and $s_{i,1}$. These are not probabilities but logits for a binary categorical distribution.
- **Mask Variable**: A binary variable $\mathcal{M}_i \in \{0, 1\}$ sampled from the distribution defined by the scores.

### Numbered Steps

1.  **Mask Sampling**:
    For each Gaussian $i$, sample a differentiable binary mask $\mathcal{M}_i$ using the Gumbel-Softmax trick applied to the two learnable mask scores $s_{i,0}$ and $s_{i,1}$ [S1]. This allows the mask to be treated as a stochastic sample during training, enabling gradient flow through the sampling process [S1].

2.  **Standard Splatting (Alpha Filtering)**:
    Project all $N$ Gaussians to the 2D image plane. Compute the 2D covariance and opacity $\alpha_i$ for each Gaussian based on its 3D parameters and view direction. Crucially, **do not** apply the mask $\mathcal{M}_i$ to the Gaussian's opacity or scale at this stage. All Gaussians, regardless of their mask value, are splatted to compute valid alpha values [S1].

3.  **Alpha Filtering**:
    Sort the Gaussians by depth and apply the standard alpha-filtering threshold. Since masks were not applied to opacity, Gaussians with $\mathcal{M}_i=0$ still have non-zero $\alpha_i$ and pass through the filter, ensuring they participate in the rasterization loop [S1].

4.  **Masked Rasterization (Forward Pass)**:
    Iterate through the sorted Gaussians $i=1 \dots N$ for each pixel. Update the accumulated color $\mathbf{c}(\mathbf{x})$ and transmittance $T$ using the following equations [S1]:
    
    Color accumulation:
    $$
    \mathbf{c}(\mathbf{x}) = \sum_{i=1}^N \mathcal{M}_i \cdot \mathbf{c}_i \cdot \alpha_i \cdot T_i
    $$
    
    Transmittance update:
    $$
    T_{i+1} = \mathcal{M}_i \cdot (1 - \alpha_i) \cdot T_i + (1 - \mathcal{M}_i) \cdot T_i
    $$
    
    Here, $T_i$ is the transmittance before Gaussian $i$. If $\mathcal{M}_i = 1$, the Gaussian contributes its color and attenuates transmittance normally. If $\mathcal{M}_i = 0$, the Gaussian's color contribution is zeroed out, and transmittance is not attenuated (i.e., $T_{i+1} = T_i$) [S1].

5.  **Loss Computation**:
    Compute the total loss $L$ as the sum of the rendering loss $L_{render}$ (e.g., MSE between rendered and ground truth color) and a regularization term $L_m$ that constrains the average number of Gaussians [S1]:
    
    $$
    L_m = \left( \frac{1}{N} \sum_{i=1}^N \mathcal{M}_i \right)^2
    $$
    
    $$
    L = L_{render} + \lambda_m \cdot L_m
    $$
    
    where $\lambda_m$ is a balancing hyperparameter [S1].

6.  **Backpropagation**:
    Compute gradients with respect to the mask scores. The gradient of the loss with respect to the mask $\mathcal{M}_i$ is derived as [S1]:
    
    $$
    \frac{\partial L}{\partial \mathcal{M}_i} = \alpha_i \cdot T_i \cdot \frac{\partial L}{\partial \mathbf{c}(\mathbf{x})} \cdot (\mathbf{c}_i - \mathbf{b}_{i+1})
    $$
    
    where $\mathbf{b}_{i+1}$ is the background color rendered behind the $i$-th Gaussian, defined as:
    
    $$
    \mathbf{b}_{i+1} = \sum_{j=i+1}^N \mathcal{M}_j \cdot \mathbf{c}_j \cdot \alpha_j \cdot T_j
    $$
    
    This gradient allows masked-out Gaussians ($\mathcal{M}_i=0$) to receive updates based on their potential contribution (difference between their color and the background) [S1].

7.  **Pruning Procedure**:
    Periodically, sample each Gaussian's mask 10 times. If a Gaussian is never sampled (probability near zero) across these 10 trials, it is permanently removed from the scene [S1]. This pruning is applied at every densification step and every 1000 iterations after densification [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def mask_gaussian_rasterization(gaussians, cameras, image_size):
    """
    Forward pass for MaskGaussian.
    
    Args:
        gaussians: List of Gaussian objects. Each has:
            - mu: (3,) mean position
            - cov: (3,3) covariance matrix (or scale/rotation)
            - c: (3,) color
            - alpha: (1,) base opacity
            - mask_scores: (2,) learnable logits for mask
        cameras: Camera parameters
        image_size: (H, W)
        
    Returns:
        rendered_color: (H, W, 3)
    """
    # 1. Project Gaussians and compute 2D properties
    # This step is identical to standard 3DGS rasterization
    projected_gaussians = project_gaussians(gaussians, cameras)
    
    # 2. Sample Masks using Gumbel-Softmax
    # mask_scores shape: (N, 2)
    mask_probs = F.softmax(gaussians.mask_scores, dim=-1)
    # Sample binary masks. In training, use straight-through estimator or 
    # differentiable sampling. Here we assume a differentiable sample function.
    masks = sample_gumbel_softmax(gaussians.mask_scores, hard=True) 
    # masks shape: (N, 1), values in {0, 1}
    
    # 3. Alpha Filtering
    # Sort by depth. Do NOT filter based on masks.
    sorted_indices = sort_by_depth(projected_gaussians)
    sorted_gaussians = [projected_gaussians[i] for i in sorted_indices]
    sorted_masks = [masks[i] for i in sorted_indices]
    
    # 4. Masked Rasterization
    H, W = image_size
    rendered_color = torch.zeros(H, W, 3)
    transmittance = torch.ones(H, W, 1)
    
    for i, (g, m) in enumerate(zip(sorted_gaussians, sorted_masks)):
        # Compute 2D alpha for this Gaussian
        alpha_i = g.alpha * compute_2d_gaussian_alpha(g)
        
        # Get pixel coverage for this Gaussian
        pixel_mask = get_pixel_coverage(g) # (H, W) boolean or soft mask
        
        # Update Transmittance: T_{i+1} = M_i * (1 - alpha_i) * T_i + (1 - M_i) * T_i
        # If M_i=1: T_new = (1 - alpha_i) * T_old
        # If M_i=0: T_new = T_old
        transmittance = (m * (1 - alpha_i) + (1 - m)) * transmittance
        
        # Update Color: c(x) = sum M_i * c_i * alpha_i * T_i
        # Note: T_i is the transmittance BEFORE this Gaussian
        contribution = m * g.c * alpha_i * transmittance
        rendered_color = rendered_color + contribution * pixel_mask
        
    return rendered_color

def compute_mask_gradient(loss, rendered_color, gaussians, masks, transmittance_history):
    """
    Backward pass logic for mask gradients.
    Formula: dL/dM_i = alpha_i * T_i * dL/dc(x) * (c_i - b_{i+1})
    """
    # dL/dc(x) is provided by the loss function's backward pass
    dL_dc = torch.autograd.grad(loss, rendered_color, retain_graph=True)[0]
    
    gradients = []
    for i, (g, m) in enumerate(zip(gaussians, masks)):
        alpha_i = g.alpha
        T_i = transmittance_history[i] # Transmittance before Gaussian i
        
        # Compute background color b_{i+1}
        # Sum of contributions from Gaussians j > i
        b_i_plus_1 = sum(
            m_j * g_j.c * g_j.alpha * T_j 
            for j, (g_j, m_j, T_j) in enumerate(zip(gaussians[i+1:], masks[i+1:], transmittance_history[i+1:]))
        )
        
        # Gradient calculation
        grad = alpha_i * T_i * dL_dc * (g.c - b_i_plus_1)
        gradients.append(grad)
        
    return gradients

def prune_gaussians(gaussians, masks, num_samples=10):
    """
    Prune Gaussians that are never sampled as 'present' (M=1) in num_samples trials.
    """
    # Re-sample masks multiple times to estimate probability
    # In practice, this might be done by sampling from the Gumbel-Softmax distribution
    # without the straight-through estimator, or by checking the probability threshold.
    # The source specifies: "Sample each Gaussian 10 times. Remove Gaussians that are never sampled."
    
    present_counts = torch.zeros(len(gaussians))
    for _ in range(num_samples):
        # Sample hard masks
        current_masks = sample_gumbel_softmax(gaussians.mask_scores, hard=True)
        present_counts += (current_masks == 1).float().sum(dim=1)
        
    # If count is 0, probability is near zero
    to_remove = present_counts == 0
    return to_remove
```

### Parameters
- **$\lambda_m$**: Balancing weight for the squared loss $L_m$.
    - Typical values: 0.1 for iterations 19,000–20,000; 0.0005 or 0.001 for iterations 0–30,000 [S1].
- **Pruning Frequency**: Every densification step and every 1000 iterations after densification [S1].
- **Sampling Count for Pruning**: 10 samples per Gaussian [S1].

### Edge Cases and Pitfalls
- **Gradient Vanishing in Attribute-Based Masking**: If masks are applied to opacity or scale (multiplying them), the resulting alpha becomes zero. This causes the Gaussian to be filtered out before rasterization, preventing any gradient update to the mask or attributes [S1]. MaskGaussian avoids this by applying masks during blending [S1].
- **Outdated Gaussians**: In deterministic pruning, unsampled Gaussians remain static and may become outdated as the scene evolves. MaskGaussian mitigates this by allowing unsampled Gaussians to receive gradients through the mask, keeping their probability of existence dynamic [S1].
- **Fine Gaussians Disadvantage**: Methods that bind mask gradients to scales/opacity may disadvantage fine Gaussians if they receive negative gradients intended to shrink them. MaskGaussian decouples mask gradients from physical attributes [S1].

## Operating characteristics
- **Pruning Ratio**: Achieves pruning ratios of 62.4% (Mip-NeRF360), 67.7% (Tanks & Temples), and 75.3% (Deep Blending) [S1].
- **Rendering Speedup**: 2.05x (Mip-NeRF360), 2.19x (Tanks & Temples), and 3.16x (Deep Blending) compared to standard 3DGS [S1].
- **Quality**: PSNR decline of only 0.02 on Mip-NeRF360 and Tanks & Temples; improvement on Deep Blending [S1].
- **Performance Metrics (Ours-α setting)**:
    - **Mip-NeRF360**: 27.43 PSNR, 1.205M Gaussians, 384.7 FPS (vs 3DGS: 27.45 PSNR, 3.204M Gaussians, 187.8 FPS) [S1].
    - **Tanks & Temples**: 23.72 PSNR, 0.590M Gaussians, 558.3 FPS (vs 3DGS: 23.74 PSNR, 1.825M Gaussians, 254.5 FPS) [S1].
    - **Deep Blending**: 29.69 PSNR, 0.694M Gaussians, 637.1 FPS (vs 3DGS: 29.53 PSNR, 2.815M Gaussians, 201.3 FPS) [S1].
- **Hardware**: Experiments conducted on NVIDIA RTX 4090 [S1].
- **Strengths**: Better rendering quality with fewer Gaussians than previous pruning methods; adaptive pruning that evolves with the scene; efficient rendering due to reduced Gaussian count.
- **Limitations**: Requires modification of the CUDA rasterization code; introduces additional learnable parameters (mask scores); pruning is stochastic, requiring careful tuning of $\lambda_m$ and sampling frequency.

## Background
MaskGaussian was introduced in the paper "MaskGaussian: Adaptive 3D Gaussian Representation from Probabilistic Masks" (CVPR 2025) [S1]. It builds upon Compact3DGS, extending its pruning approach from deterministic to probabilistic. The method is inspired by techniques in other fields (CNNs, Vision Transformers, LLMs) where masks are optimized to receive gradients from both active and inactive components, balancing benefit and cost [S1]. It addresses the limitation of previous mask-based pruning methods that fail to update unsampled Gaussians, leading to sub-optimal reconstruction [S1].

## Sources
[S1] MaskGaussian: Adaptive 3D Gaussian Representation from Probabilistic Masks, https://doi.org/10.1109/cvpr52734.2025.00072, fetched 2026-10-30
