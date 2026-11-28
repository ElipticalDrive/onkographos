# LinearMask-GS

Also called: Linear increment activation, LinearMask

## Overview
LinearMask-GS is a learned-mask pruning framework for 3D Gaussian Splatting that replaces the Gumbel-Sigmoid activation with a linear increment activation to keep mask values in a mid-confidence regime, producing a stable unimodal distribution for reliable importance ranking [S1]. It solves the instability of prior learned-mask methods where mask values saturate to 0 or 1 before the importance ranking stabilizes, resulting in a bimodal distribution where the ranking reflects early optimization artifacts rather than true importance [S1]. Unlike its siblings which may use summation-based aggregation or different activation functions, LinearMask-GS combines max-pooling importance scoring with a linear activation to mitigate view-count inflation and ensure deterministic pruning decisions [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of $N$ Gaussian primitives $\{G_i\}_{i=1}^N$ optimized via a standard 3D Gaussian Splatting pipeline (e.g., 3DGS, 2DGS, DropGaussian, OctreeGS, or FastGS) for an initial geometric warm-up phase [S1].
*   **Output:** A pruned set of $K = \rho N$ Gaussians, where $\rho$ is the minimum-survival ratio, and the optimized scene parameters for these remaining primitives [S1].

**Data Structures:**
*   **Gaussian Primitive $i$:** Contains mean position $\mu_i \in \mathbb{R}^3$, covariance matrix $\Sigma_i$ (decomposed into rotation $R_i$ and scaling $S_i$), opacity logit $o_i$, and spherical harmonic coefficients $c_i$ [S1].
*   **Learnable Mask Parameter $m_i$:** A scalar parameter for each Gaussian, initialized to 1 [S1].
*   **Importance Score $S_i$:** A scalar value computed per Gaussian using max-pooling over training views [S1].
*   **Soft Mask Value $\psi_i$:** A scalar in $[0, 1]$ computed as $\psi_i = f_{LI}(m_i \cdot S_i; \tau)$ [S1].

**Parameters:**
*   $\tau$: Slope hyperparameter for the linear activation. Default value is 0.1. Grid search values include $\{0.01, 0.05, 0.1, 0.5\}$ [S1].
*   $\rho$: Minimum-survival ratio. Default value is 0.30 [S1].
*   $\lambda_m$: Mask sparsity loss weight. Value is $5 \times 10^{-4}$ [S1].
*   $\lambda_D$: Photometric loss weight. Value is 0.2 [S1].
*   Learning rate for $m_i$: $5 \times 10^{-2}$ using AdamW optimizer [S1].

**Numbered Steps:**

1.  **Geometric Warm-up:** Run the standard Gaussian Splatting optimization pipeline for 19,500 iterations to ensure geometric convergence [S1].
2.  **Compute Importance Scores:** For each Gaussian $i$, compute the max-pooled importance score $S_i$ over all training views $k \in \{1, \dots, K\}$ and pixels $p$:
    $$ S_i = \max_{k} \max_{p} w_{k,p}^i $$
    where $w_{k,p}^i$ is the contribution weight of Gaussian $i$ to pixel $p$ in view $k$ [S1]. Recompute $S_i$ every 20 iterations during the mask training phase [S1].
3.  **Initialize Mask Parameters:** Initialize learnable scalar parameters $m_i = 1$ for all $i=1, \dots, N$ [S1].
4.  **Mask Training (Iterations 19,500–20,000):**
    a. Compute the soft mask value for each Gaussian:
    $$ \psi_i = f_{LI}(m_i \cdot S_i; \tau) = \text{clip}(\tau \cdot (m_i \cdot S_i - 0.5) + 0.5, 0, 1) $$
    where $f_{LI}$ is the linear increment activation [S1].
    b. Modulate the opacity of each Gaussian during rendering:
    $$ \tilde{\alpha}_{i,k,p} = \psi_i \cdot \alpha_{i,k,p} $$
    where $\alpha_{i,k,p}$ is the original opacity [S1].
    c. Compute the total training loss:
    $$ L_{total} = (1 - \lambda_D) L_1 + \lambda_D (1 - \text{SSIM}) + \lambda_m L_{mask} $$
    where $L_1$ is the pixel-wise L1 loss, SSIM is structural similarity, and $L_{mask}$ is the sparsity term [S1].
    d. Compute the mask sparsity loss:
    $$ L_{mask} = \frac{1}{N} \sum_{i=1}^{N} |m_i| $$
    This encourages $m_i$ to shrink, reducing $\psi_i$ [S1].
    e. Update scene parameters and mask parameters $\{m_i\}$ using their respective optimizers. The mask parameters use AdamW with learning rate $5 \times 10^{-2}$ [S1].
5.  **Hard Pruning (Iteration 20,000):**
    a. Rank all $N$ Gaussians by their final soft mask values $\psi_i$ in descending order.
    b. Select the top-$\rho N$ Gaussians to retain:
    $$ K = \text{top-}\rho N \text{ Gaussians ranked by } \psi_i $$
    c. Permanently prune the remaining $N - K$ Gaussians [S1].
6.  **Fine-tuning:** Fine-tune the remaining $K$ primitives without the mask loss term for the remainder of the training (up to 30,000 iterations total) [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim

def linear_increment_activation(x, tau):
    """
    Linear increment activation function.
    x: input tensor (m_i * S_i)
    tau: slope hyperparameter
    """
    # clip(tau * (x - 0.5) + 0.5, 0, 1)
    return torch.clamp(tau * (x - 0.5) + 0.5, min=0.0, max=1.0)

def compute_max_pool_importance(gaussians, views):
    """
    Compute max-pooled importance score S_i for each Gaussian.
    gaussians: list of Gaussian primitives
    views: list of training views
    Returns: tensor S of shape (N,)
    """
    N = len(gaussians)
    S = torch.zeros(N, device=gaussians[0].position.device)
    
    for i, g in enumerate(gaussians):
        max_val = -float('inf')
        for view in views:
            # Compute contribution weights w_{k,p}^i for this Gaussian in this view
            # This involves projecting the Gaussian to the view and summing contributions
            # to pixels. The exact implementation depends on the rasterizer.
            # For the purpose of this algorithm, we assume a function get_weights(g, view)
            # returns a tensor of weights for all pixels in the view.
            weights = get_weights(g, view) 
            # Max over all pixels in the view
            view_max = torch.max(weights)
            if view_max > max_val:
                max_val = view_max
        S[i] = max_val
    return S

def train_linear_mask_gs(gaussians, views, total_iterations=30000, warmup_iterations=19500, 
                         mask_iterations=500, rho=0.30, tau=0.1, lambda_m=5e-4, lambda_D=0.2):
    """
    Main training loop for LinearMask-GS.
    """
    N = len(gaussians)
    
    # Initialize mask parameters
    m = torch.ones(N, requires_grad=True, device=gaussians[0].position.device)
    
    # Optimizers
    # Scene parameters optimizer (standard 3DGS optimizer)
    scene_optimizer = optim.AdamW(gaussians.parameters(), lr=1.6e-4) # Example LR
    # Mask parameters optimizer
    mask_optimizer = optim.AdamW([m], lr=5e-2)
    
    # Step 1: Geometric Warm-up
    for iteration in range(warmup_iterations):
        # Standard 3DGS optimization step
        render_image = render_scene(gaussians, views)
        target_image = get_target_image(views)
        loss = (1 - lambda_D) * L1_loss(render_image, target_image) + lambda_D * (1 - SSIM(render_image, target_image))
        scene_optimizer.zero_grad()
        loss.backward()
        scene_optimizer.step()
        
    # Step 2: Mask Training
    # Recompute importance scores every 20 iterations
    importance_update_freq = 20
    S = compute_max_pool_importance(gaussians, views)
    
    for iteration in range(warmup_iterations, warmup_iterations + mask_iterations):
        # Recompute importance scores periodically
        if (iteration - warmup_iterations) % importance_update_freq == 0:
            S = compute_max_pool_importance(gaussians, views)
            
        # Compute soft mask values
        # psi_i = f_LI(m_i * S_i; tau)
        input_to_activation = m * S
        psi = linear_increment_activation(input_to_activation, tau)
        
        # Modulate opacity during rendering
        # This is handled internally by the renderer using psi
        render_image = render_scene(gaussians, views, mask_values=psi)
        target_image = get_target_image(views)
        
        # Compute losses
        photometric_loss = (1 - lambda_D) * L1_loss(render_image, target_image) + lambda_D * (1 - SSIM(render_image, target_image))
        mask_loss = torch.mean(torch.abs(m))
        total_loss = photometric_loss + lambda_m * mask_loss
        
        # Update scene parameters
        scene_optimizer.zero_grad()
        total_loss.backward(retain_graph=True) # Retain graph for mask gradient
        scene_optimizer.step()
        
        # Update mask parameters
        mask_optimizer.zero_grad()
        # The gradient of total_loss w.r.t m is already computed in retain_graph
        # We need to ensure only m gets the gradient from the mask part
        # In practice, we might need to detach scene parameters or handle gradients carefully
        # The source implies joint optimization.
        mask_optimizer.step()
        
    # Step 3: Hard Pruning
    # Rank Gaussians by psi
    _, indices = torch.sort(psi, descending=True)
    K = int(rho * N)
    keep_indices = indices[:K]
    
    # Prune Gaussians
    pruned_gaussians = [gaussians[i] for i in keep_indices]
    
    # Step 4: Fine-tuning
    for iteration in range(warmup_iterations + mask_iterations, total_iterations):
        render_image = render_scene(pruned_gaussians, views)
        target_image = get_target_image(views)
        loss = (1 - lambda_D) * L1_loss(render_image, target_image) + lambda_D * (1 - SSIM(render_image, target_image))
        scene_optimizer.zero_grad()
        loss.backward()
        scene_optimizer.step()
        
    return pruned_gaussians
```

**Edge Cases and Pitfalls:**
*   **Saturation:** Using Gumbel-Sigmoid activation causes mask values to saturate to 0 or 1 before importance ranking stabilizes, resulting in a bimodal distribution where the ranking reflects early optimization artifacts rather than true importance [S1].
*   **View-Count Inflation:** Summation-based aggregation introduces view-count inflation that dominates the activation choice and obscures the stability benefit of the linear mask [S1]. Max-pooling is used to mitigate this [S1].
*   **Survival Ratio:** Setting $\rho$ too low (e.g., 0.20) removes too many primitives, degrading quality (-0.28 dB). Setting $\rho$ too high (e.g., 0.40) retains 34% more Gaussians for only +0.03 dB gain [S1].

## Operating characteristics
**Time and Memory Complexity:**
*   **Memory:** Reduces Gaussian count significantly. On Mip-NeRF 360, it reduces Gaussian count from 3.36M (vanilla 3DGS) to 0.94M (3.6x reduction) [S1]. For outdoor scenes, it yields a 1.6x reduction (from 2.18M to 1.36M Gaussians) [S1].
*   **Time:** On the 3DGS backbone, LinearMask-GS is 2.12 minutes faster than its learned-mask parent LP-3DGS while reaching a more compact final model [S1]. On the FastGS backbone, LinearMask-GS⋆ reduces training time (1.93 to 1.81 minutes) [S1].

**Measured Performance:**
*   **Mip-NeRF 360 (3DGS backbone):** Reduces Gaussian count from 3.36M to 0.94M (3.6x reduction) while improving PSNR from 27.21 to 27.70 dB [S1].
*   **Mip-NeRF 360 (vs LP-3DGS):** Surpasses LP-3DGS by 0.23 dB PSNR and 0.0135 SSIM with 1.6x fewer Gaussians [S1].
*   **Outdoor Scenes:** Yields a 1.6x reduction (from 2.18M to 1.36M Gaussians) with PSNR gain of +0.38 dB, SSIM gain of +0.025, and LPIPS reduction of -0.029 [S1].
*   **2DGS Backbone:** Reduces count from 1.06M to 0.36M (2.9x), PSNR +0.22 dB, SSIM +0.009, LPIPS +0.008 [S1].
*   **DropGaussian Backbone:** Improves PSNR by +0.31 to +0.52 dB, retains 41% fewer primitives than Gumbel variant [S1].
*   **Octree-GS Backbone:** Anchors reduced from 1.09M to 0.38M (2.9x), PSNR +0.12 dB [S1].
*   **FastGS Backbone:** 0.38M Gaussians, PSNR 27.69 dB, SSIM 0.8029, LPIPS 0.2227, Storage 94.89 MB, Training time 1.81 min [S1].

**Scaling:**
*   The method scales with the number of Gaussians and views. The importance score computation is $O(N \cdot K \cdot P)$ where $K$ is views and $P$ is pixels, but max-pooling reduces the aggregation cost [S1].

**Strengths and Limitations:**
*   **Strengths:** Stable unimodal mask distribution, deterministic ranking, high compression ratios with minimal quality loss, compatible with various backbones (3DGS, 2DGS, DropGaussian, OctreeGS, FastGS) without architectural modifications [S1].
*   **Limitations:** The slope $\tau$ and ratio $\rho$ are tuned on a held-out set and fixed across scenes. The brief mask window can leave specular or transparent regions under-discriminated. The global $\rho$ sets an overall rather than per-region budget. Inherits the static-scene assumption of its backbones [S1].

## Background
LinearMask-GS was proposed by Ryu and Lee in 2026 [S1]. It addresses the limitations of learned-mask pruning methods like LP-3DGS, which use Gumbel-Sigmoid activation. The authors identified that the steep slope of Gumbel-Sigmoid drives mask values to the extremes within the short mask-training window, before the importance ranking has stabilized, producing a sharply bimodal distribution from which that ranking can no longer be reliably recovered [S1]. LinearMask-GS replaces this with a linear increment activation to keep mask values in a mid-confidence regime, producing a stable, unimodal mask distribution whose ranking tracks importance [S1]. It is used for compact 3D Gaussian Splatting, particularly for resource-constrained devices where storage overhead is a concern [S1].

## Sources
[S1] LinearMask-GS: Stable-Mask Importance Pruning for Compact 3D Gaussian Splatting, https://arxiv.org/abs/2609.10095, fetched 2026-10-29
