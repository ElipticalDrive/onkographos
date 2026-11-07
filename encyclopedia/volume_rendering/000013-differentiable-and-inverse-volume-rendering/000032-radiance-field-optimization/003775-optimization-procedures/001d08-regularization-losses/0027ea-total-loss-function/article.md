# Total Loss Function

Also called: Combined Loss, Full Training Objective, Loss function, Total Loss

## Overview

The Total Loss Function is the composite objective function used to optimize radiance field models (such as neural volumes, implicit surfaces, or Gaussian splats) by combining multiple regularization and fidelity terms. It solves the ill-posed nature of inverse volume rendering by balancing photometric reconstruction accuracy against geometric constraints, latent space regularization, and artifact suppression [S1][S2, unofficial][S8]. It sets itself apart from sibling methods by serving as the aggregation layer that weights and sums distinct loss components (e.g., [Composite Image Loss](4262-composite-image-loss-with-visibility-weighting.md), [Distortion Loss](2755-distortion-loss.md)) rather than defining a single specific metric or visibility mask [S1, Siblings].

## How it works

**Inputs and Outputs:**
*   **Inputs:** Rendered image $\hat{\mathbf{I}}$ (or color $\hat{\mathbf{C}}$), ground-truth image $\mathbf{I}^*$ (or color $\mathbf{C}^*$), scene parameters (density $\sigma$, color $\mathbf{c}$, SDF $f$, latent codes $\mathbf{z}$), auxiliary outputs (depth, throughput, normals), and hyperparameters (weights $\lambda$).
*   **Outputs:** A scalar loss value $L$ used for gradient-based optimization.

**Data Structures:**
*   **Images/Tensors:** Rendered and ground-truth images are typically tensors of shape $(H, W, 3)$ or $(N, 3)$ for batched rays.
*   **Parameters:** Scene parameters are stored in differentiable tensors (e.g., MLP weights, voxel grids, Gaussian attributes).
*   **Weights:** Scalar hyperparameters $\lambda_i$ controlling the balance of loss terms.

**Formulas and Steps:**
The total loss is generally a weighted sum of individual loss terms. The specific composition varies by model architecture, but common components include photometric loss, regularization, and auxiliary constraints.

1.  **Photometric Loss ($L_{\text{photo}}$):** Measures the difference between rendered and ground-truth colors.
    *   L1 Loss: $L_1 = \frac{1}{N} \sum_{i=1}^N |C_i - \hat{C}_i|$ [S2, unofficial][S8].
    *   L2 Loss: $L_2 = \frac{1}{P} \sum_{\mathbf{p}} \| \hat{\mathbf{I}}_{\text{rgb}}(\mathbf{p}) - \mathbf{I}_{\text{rgb}}^*(\mathbf{p}) \|^2$ [S1].
    *   SSIM Loss ($L_{\text{ssim}}$): Structural Similarity Index [S8].
    *   Composite Photometric: $L_{\text{rgb}} = (1-\lambda_r) L_1 + \lambda_r L_{\text{ssim}}$ [S8].

2.  **Geometric/Regularization Loss ($L_{\text{reg}}$):** Enforces physical plausibility or smoothness.
    *   Eikonal Loss: $L_{\text{eik}} = \frac{1}{MN} \sum_{i,j=1}^{MN} \| \nabla f(p_{i,j}) - 1 \|^2$ [S2, unofficial].
    *   Total Variation (TV) Loss: $L_{\text{tv}} = \frac{1}{B} \sum_{i=2}^{D} \frac{\sum_{j} (x_{..., j+1, ...} - x_{..., j, ...})^2}{N_i}$ [S6, unofficial].
    *   KL Divergence: $D_{\text{KL}}(\mathbf{z} \parallel \mathcal{N}(0, 1))$ for latent encodings [S1].

3.  **Auxiliary/Specific Losses:**
    *   Entropy Loss: $L_{\text{entropy}} = -\sum (\mathbf{O}_{\text{fg}}\log \mathbf{O}_{\text{fg}} + (1-\mathbf{O}_{\text{fg}})\log (1-\mathbf{O}_{\text{fg}}))$ [S8].
    *   Empty Space/Opacity Loss: Based on throughput $T$ and mask $m$ [S3, unofficial].
    *   Depth Consistency: Aligns rendered depth with sparse depth estimates [S1].

4.  **Total Loss Aggregation:**
    The final loss is computed by summing the weighted components. Examples include:
    *   **Neural Volumes:** $\ell(\theta) = L_2 + \lambda_{\text{KL}} D_{\text{KL}} + J_{\text{tv}} + J_{\text{B}}$ [S1].
    *   **Point-NeuS:** $\mathcal{L} = (L_{\text{rgb}} + 0.1 L_{\text{eik}}) + (\lambda_1 L_{\text{usdf}} + \lambda_2 L_{\text{bias}} + \lambda_3 L_{\text{pc}})$ [S2, unofficial].
    *   **HybridNeRF:** $\mathcal{L}(\mathbf{r}) = \mathcal{L}_{\text{photo}} + \lambda_{\text{Eik}} \mathcal{L}_{\text{Eik}} + \mathcal{L}_{\text{prop}}$ [S5].
    *   **EVolSplat:** $\mathcal{L} = L_{\text{rgb}} + \lambda_e L_{\text{entropy}}$ [S8].
    *   **Mobile-GS:** $\mathcal{L} = \mathcal{L}_{\text{rgb}} + \lambda_{\text{distill}} \mathcal{L}_{\text{distill}} + \lambda_{\text{depth}} \mathcal{L}_{\text{depth}}$ [S11, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_total_loss(rendered_color, gt_color, scene_params, auxiliary_outputs, config):
    """
    Computes the total loss for radiance field optimization.
    
    Args:
        rendered_color: Tensor of shape (N, 3) or (H, W, 3), rendered RGB values.
        gt_color: Tensor of shape (N, 3) or (H, W, 3), ground truth RGB values.
        scene_params: Dict containing density, color, SDF, latent codes, etc.
        auxiliary_outputs: Dict containing depth, throughput, normals, etc.
        config: Dict containing loss weights (lambda) and loss types.
        
    Returns:
        scalar: Total loss value.
    """
    total_loss = 0.0
    
    # 1. Photometric Loss
    # Option A: L1 Loss
    l1_loss = torch.mean(torch.abs(rendered_color - gt_color))
    
    # Option B: SSIM Loss (requires external function or implementation)
    # ssim_loss = compute_ssim(rendered_color, gt_color)
    
    # Composite Photometric (e.g., from EVolSplat/Mobile-GS)
    lambda_r = config.get('lambda_r', 0.2)
    l_rgb = (1 - lambda_r) * l1_loss # + lambda_r * ssim_loss if available
    
    total_loss += l_rgb
    
    # 2. Regularization Losses
    # Eikonal Loss (for SDF fields)
    if 'sdf' in scene_params:
        sdf = scene_params['sdf']
        # Compute gradient of SDF w.r.t. spatial coordinates
        grad_sdf = compute_gradient(sdf)
        l_eik = torch.mean(torch.norm(grad_sdf - 1.0, dim=-1) ** 2)
        lambda_eik = config.get('lambda_eik', 0.01) # Default from HybridNeRF/Point-NeuS
        total_loss += lambda_eik * l_eik
        
    # Total Variation Loss (for voxel grids/bilateral grids)
    if 'voxel_grid' in scene_params:
        grid = scene_params['voxel_grid']
        l_tv = compute_total_variation(grid)
        lambda_tv = config.get('lambda_tv', 0.01) # Default from Neural Volumes
        total_loss += lambda_tv * l_tv
        
    # KL Divergence (for latent codes)
    if 'latent_z' in scene_params:
        z = scene_params['latent_z']
        # Assume z is sampled from a distribution with mean mu and logvar logvar
        mu = scene_params.get('mu', 0.0)
        logvar = scene_params.get('logvar', 0.0)
        l_kl = -0.5 * torch.mean(1 + logvar - mu**2 - torch.exp(logvar))
        lambda_kl = config.get('lambda_kl', 0.001) # Default from Neural Volumes
        total_loss += lambda_kl * l_kl
        
    # 3. Auxiliary Losses
    # Entropy Loss (for foreground alpha)
    if 'foreground_alpha' in auxiliary_outputs:
        alpha_fg = auxiliary_outputs['foreground_alpha']
        # Clamp alpha to avoid log(0)
        alpha_fg = torch.clamp(alpha_fg, min=1e-7, max=1-1e-7)
        l_entropy = -torch.mean(alpha_fg * torch.log(alpha_fg) + 
                                (1 - alpha_fg) * torch.log(1 - alpha_fg))
        lambda_e = config.get('lambda_e', 0.1) # Default from EVolSplat
        total_loss += lambda_e * l_entropy
        
    # Empty Space / Opacity Loss (for masks)
    if 'throughput' in auxiliary_outputs and 'mask' in auxiliary_outputs:
        throughput = auxiliary_outputs['throughput']
        mask = auxiliary_outputs['mask']
        # Empty Space Loss: encourage throughput=0 where mask=0
        l_empty = torch.mean((1 - mask) * torch.abs(throughput - 0.0))
        lambda_empty = config.get('lambda_empty', 1.0)
        total_loss += lambda_empty * l_empty
        
        # Opacity Loss: encourage throughput=1 where mask=1 (or alpha=1)
        # Note: Throughput is transmittance. If mask=1, we want object to be opaque,
        # so transmittance should be low? Or throughput refers to accumulated alpha?
        # Based on S3, Opacity Loss encourages throughput=1 where mask=1.
        # This implies 'throughput' might be interpreted as 'accumulated opacity' in this context.
        l_opacity = torch.mean(mask * torch.abs(throughput - 1.0))
        lambda_opacity = config.get('lambda_opacity', 1.0)
        total_loss += lambda_opacity * l_opacity
        
    # 4. Curriculum Learning Schedule (Optional)
    # If using curriculum learning (e.g., DBARF), weight terms by iteration
    if 'use_curriculum' in config and config['use_curriculum']:
        t = config['current_iteration']
        beta = config.get('beta', -1e-5)
        # Formula: L_final = 2^(beta*t) * (L_depth + L_photo) + (1 - 2^(beta*t)) * L_rgb
        # This is a specific weighting strategy. The base L_rgb and L_depth must be computed separately.
        # For simplicity, this example assumes L_rgb is already computed and L_depth is available.
        if 'depth' in auxiliary_outputs:
            depth_pred = auxiliary_outputs['depth']
            depth_gt = auxiliary_outputs['depth_gt']
            l_depth = torch.mean(torch.abs(depth_pred - depth_gt))
            
            # Recalculate weighted sum if curriculum is active
            # Note: This overrides the simple sum above if implemented strictly
            weight_photo = 2 ** (beta * t)
            weight_rgb = 1 - 2 ** (beta * t)
            
            # Re-weight existing losses or replace with curriculum-specific terms
            # This is a placeholder for the specific curriculum logic
            pass 
            
    return total_loss

def compute_total_variation(x):
    """
    Computes Total Variation loss for a tensor x.
    x shape: (B, C, ...)
    """
    loss = 0.0
    # Iterate over spatial dimensions (skip batch and channel)
    for i in range(2, x.dim()):
        # Slice tensor to get adjacent elements
        # x[..., 1:, ...] and x[..., :-1, ...] along dimension i
        slices = []
        for j in range(x.shape[i] - 1):
            # Construct slice indices
            idx_before = [slice(None)] * x.dim()
            idx_after = [slice(None)] * x.dim()
            idx_before[i] = slice(j, j+1)
            idx_after[i] = slice(j+1, j+2)
            slices.append((x[idx_before] - x[idx_after]) ** 2)
            
        diff_sq = torch.stack(slices).sum()
        # Normalize by number of elements in dimension i excluding batch and channel
        n_elements = x.shape[i] - 1
        for k in range(2, x.dim()):
            if k != i:
                n_elements *= x.shape[k]
        
        loss += diff_sq / n_elements
        
    # Normalize by batch size
    loss = loss / x.shape[0]
    return loss
```

**Parameters:**
*   $\lambda_{\text{KL}}$: Weight for KL divergence. Typical value: 0.001 [S1].
*   $\lambda_{\text{Eik}}$: Weight for Eikonal loss. Typical value: 0.01 [S5] or 0.1 [S2, unofficial].
*   $\lambda_{\text{tv}}$: Weight for Total Variation. Typical value: 0.01 [S1].
*   $\lambda_r$: Weight for SSIM in composite photometric loss. Typical value: 0.2 [S8].
*   $\lambda_e$: Weight for entropy loss. Typical value: 0.1 [S8].
*   $\lambda_{\text{distill}}$: Weight for distillation loss. Typical value: 1 [S11, unofficial].
*   $\lambda_{\text{depth}}$: Weight for depth loss. Typical value: 0.1 [S11, unofficial].
*   $\lambda_1, \lambda_2, \lambda_3$: Weights for specific module losses (e.g., uncertain SDF, bias, point cloud). Values: 1, 1, 0.25 [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Gradient Vanishing/Exploding:** If weights are not balanced, some terms may dominate, causing unstable gradients [S2, unofficial].
*   **Mask Mismatch:** In visibility-weighted or mask-based losses (e.g., Empty Space Loss), incorrect mask alignment with rendered pixels leads to incorrect supervision [S3, unofficial].
*   **Log(0):** Entropy and cross-entropy losses require clamping probabilities to avoid $\log(0)$ [S8].
*   **Memory Usage:** Computing gradients for all loss terms simultaneously can exceed GPU memory. Gradient checkpointing or iterative loss computation may be required [S1].

## Subtopics

### [Composite Image Loss with Visibility Weighting](4262-composite-image-loss-with-visibility-weighting.md)
The Composite Image Loss with Visibility Weighting ($L_c$) is a composite objective function used to train radiance field models, specifically SuRFNet, by aligning rendered images with ground-truth references while suppressing errors in occluded or unobserved regions. It sets itself apart from standard photometric losses by explicitly masking the L1 and SSIM terms with a visibility map $V_M$, ensuring that supervision is only applied to pixels corresponding to visible surfaces, while retaining a full-image perceptual loss.

### [Loss Function Configuration](6782-loss-function-configuration.md)
This method defines the specific hyperparameters and weighting strategies used to balance multiple loss components during the optimization of radiance fields. It addresses the problem of stabilizing training by preventing any single loss term (e.g., photometric error vs. regularization) from dominating the gradient updates. It differs from sibling methods by focusing on the static configuration of scalar weights for the total loss function rather than the dynamic computation of loss values or visibility masks.

### [Sparse Depth Consistency Loss](7294-sparse-depth-consistency-loss.md)
The Sparse Depth Consistency Loss enforces geometric consistency between the rendered depth maps of a radiance field and sparse depth estimates provided by a Structure-from-Motion (SfM) pipeline. It serves as a regularization term within the total loss function to mitigate floaters and undesired depth discontinuities common in sparse-input settings. It is distinguished from sibling losses like Composite Image Loss with Visibility Weighting by focusing specifically on depth metric alignment rather than photometric color fidelity or visibility-weighted compositing.

### [Loss Weight Configuration](7883-loss-weight-configuration.md)
Loss weight configuration determines the scalar parameter $\lambda$ that balances the relative contribution of distinct loss terms during radiance-field optimization. It addresses the trade-off between preserving color fidelity and enforcing geometric consistency or adversarial realism. It differs from [Composite Image Loss with Visibility Weighting](4262-composite-image-loss-with-visibility-weighting.md) by weighting loss *terms* rather than pixel *visibility*, and from [Sparse Depth Consistency Loss](7294-sparse-depth-consistency-loss.md) by focusing on the global balance of the total loss function rather than specific depth constraints.

### [Composite Photometric and Regularization Loss](9355-composite-photometric-and-regularization-loss.md)
This method defines the composite objective function used to optimize radiance field models (such as dynamic voxel grids or Gaussian splats) from monocular video inputs. It solves the ill-posed problem of reconstructing 3D appearance and motion by combining photometric fidelity with multiple regularization terms that enforce physical plausibility, such as spatial smoothness, static background, and geometric constraints. It differs from simple photometric loss by explicitly penalizing artifacts like thin Gaussians, non-static background noise, and discontinuous motion fields, which are critical for stable convergence in dynamic scenes.

### Empty Space Loss
An AOV loss that encourages the rendering throughput to equal 0.0 in regions where a binary object mask is inactive [S3, unofficial].
**Formula:** $L = w \cdot \frac{1}{N} \sum_{i=1}^{N} (1 - m_i) \cdot |T_i - 0|$, where $w$ is the weight, $N$ is the number of samples, $m_i \in \{0, 1\}$ is the binary mask value, and $T_i$ is the throughput value [S3, unofficial].
**Parameters:** `masks`: Binary mask array/tensor indicating regions where objects exist. Values should be 0 or 1. `weight`: Float scalar multiplying the loss term. Default/typical value: 1.0 [S3, unofficial].

### Opacity Loss
An AOV (Arbitrary Output Variable) loss that encourages the rendering throughput to equal 1.0 in regions where a binary object mask is active [S3, unofficial].
**Formula:** $L = w \cdot \frac{1}{N} \sum_{i=1}^{N} m_i \cdot |T_i - 1|$, where $w$ is the weight, $N$ is the number of samples, $m_i \in \{0, 1\}$ is the binary mask value, and $T_i$ is the throughput value [S3, unofficial].
**Parameters:** `masks`: Binary mask array/tensor indicating regions where objects should be opaque. Values should be 0 or 1. `weight`: Float scalar multiplying the loss term. Default/typical value: 1.0 [S3, unofficial].

### Auxiliary Losses
Adds per-point RGB loss to supervise the color emission of each query point directly, rather than the accumulated color, and adds background entropy loss to enforce the rendered background probability to concentrate on the background or foreground [S4]. Adding both auxiliary losses yields PSNR 35.54, compared to 35.37 (none) or 35.53 (one loss) [S4].

### HybridNeRF Optimization Loss
The total loss $\mathcal{L}(\mathbf{r})$ is the sum of photometric loss $\mathcal{L}_{\text{photo}}$, Eikonal loss $\mathcal{L}_{\text{Eik}}$ weighted by $\lambda_{\text{Eik}}$, and interlevel proposal loss $\mathcal{L}_{\text{prop}}$ [S5].
**Parameter:** $\lambda_{\text{Eik}} = 0.01$ is the weight for the Eikonal loss term [S5].

### Total Variation Loss
A regularization loss that computes the total variation of the bilateral grid parameters to encourage smoothness in the learned color transformations [S6, unofficial].
**Formula:** The total variation loss for a tensor $x$ is $L_{TV} = \frac{1}{B} \sum_{i=2}^{D} \frac{\sum_{j} (x_{..., j+1, ...} - x_{..., j, ...})^2}{N_i}$, where $B$ is batch size, $D$ is number of dimensions, and $N_i$ is the number of elements in dimension $i$ excluding batch and channel [S6, unofficial].

### Discriminator Loss with Gradient Penalty
A loss function for training the discriminator, including a gradient penalty term to enforce Lipschitz continuity [S7, unofficial].
**Formula:**
*   $loss\_d\_real = \text{mean}(\text{softplus}(-pred\_real))$
*   $loss\_d\_fake = \text{mean}(\text{softplus}(pred\_fake))$
*   $r1\_penalty = \text{mean}(\sum(\text{gradients}^2, \text{dim}=[1,2,3]))$
*   $gradient\_penalty = r1\_penalty / 2$
**Parameter:** `gp_loss_mult`: The multiplier for the gradient penalty term in the discriminator loss [S7, unofficial].

### Background Blending for Loss Computation
Blends the predicted RGB image with the ground truth background color for accurate loss computation, especially when the background is not black [S7, unofficial].

### Composite RGB and Entropy Loss
**Formula:**
*   $\mathcal {L}_{\mathrm {rgb}}=\left (1-\lambda _r\right ) \mathcal {L}_1+\lambda _r \mathcal {L}_{\mathrm {ssim}}$ where $\mathcal{L}_1$ is L1 loss, $\mathcal{L}_{\mathrm{ssim}}$ is SSIM loss, and $\lambda_r$ is the weight.
*   $\mathcal {L}_{\text {entropy}}=-\sum \left (\mathbf {O}_{\text {fg}}\log \mathbf {O}_{\text {fg}}+\left (1-\mathbf {O}_{\text {fg}}\right ) \log \left (1-\mathbf {O}_{\text {fg}}\right )\right )$ where $\mathbf{O}_{\text{fg}}$ is the foreground accumulated alpha value.
*   $\mathcal {L}=\left (1-\lambda _r\right ) \mathcal {L}_1+\lambda _r \mathcal {L}_{\mathrm {ssim}}+\lambda _e \mathcal {L}_{\text {entropy}}$ where $\lambda_e$ is the entropy loss weight.
**Parameter:** $\lambda_r$: Weight for SSIM loss in RGB loss. Value: 0.2. $\lambda_e$: Weight for entropy loss. Value: 0.1 [S8].

### Curriculum Learning Loss
A curriculum learning loss that transitions from depth/photometric supervision to RGB supervision during training, controlled by an exponential schedule based on iteration count [S9].
**Formula:** $\mathcal {L}_{\text {final}} = 2^{\beta \cdot t} (\mathcal {L}_{\text {depth}} + \mathcal {L}_{\text {photo}}) + (1-2^{\beta \cdot t}) \mathcal {L}_{\text {rgb}}$, where $\beta = -1e5$ and $t$ is the current training iteration number [S9].
**Parameter:** $\beta$: exponential decay/growth rate parameter for the loss weighting schedule. Value: $-1e5$. $t$: current training iteration number [S9].

### RocNet Loss Function
**Formula:** Total loss $L = L_l + L_r$, where $L_l$ is node labeling loss and $L_r$ is leaf reconstruction loss. Node labeling loss $L_l = - \frac{1}{4} \frac{4}{X} \frac{i}{5} c_i \text{log}(s_i)$, where $c_i$ is ground-truth and $s_i$ is node classifier score for 4 node types. The loss of an octree is the sum of label losses of all nodes [S10].

### Mobile-GS Training Loss
**Formula:** $\mathcal{L}_{\text{rgb}} = \lambda \mathcal{L}_1(\mathbf{C}, \mathbf{C}_{\text{gt}}) + (1 - \lambda) \mathcal{L}_{\text{DSSIM}}(\mathbf{C}, \mathbf{C}_{\text{gt}})$, where $\lambda$ balances L1 and DSSIM losses. Total loss: $\mathcal{L} = \mathcal{L}_{\text{rgb}} + \lambda_{\text{distill}} \mathcal{L}_{\text{distill}} + \lambda_{\text{depth}} \mathcal{L}_{\text{depth}}$.
**Parameter:** $\lambda$: Weight for L1 vs DSSIM in rendering loss. Value: 0.8. $\lambda_{\text{distill}}$: Weight for distillation loss. Value: 1. $\lambda_{\text{depth}}$: Weight for depth loss. Value: 0.1 [S11, unofficial].

## Operating characteristics

**Time and Memory Complexity:**
*   **Memory:** The total loss computation requires storing intermediate gradients for all loss terms. Standard reverse-mode AD requires $O(kn)$ intermediate states, leading to high memory usage [S5]. Adjoint methods or analytic inversion can reduce this to $O(n)$ [S1][S3, unofficial][S5].
*   **Time:** The computation time is proportional to the number of loss terms and the complexity of each term (e.g., SSIM is more expensive than L1). Training duration can be significant; for example, Neural Volumes training takes about 10 days on a single NVIDIA Tesla V100 for 500,000 iterations [S1].

**Measured Performance:**
*   Performance varies by scene and dataset. For example, adding auxiliary losses in [S4] improved PSNR from 35.37 to 35.54.
*   RocNet achieves competitive model sizes (e.g., 1.42M parameters for RocNet-64-32) [S10].
*   Mobile-GS achieves real-time performance on mobile devices [S11, unofficial].

**Scaling:**
*   Scaling to higher resolutions or larger scenes requires careful management of batch size and sampling strategies (e.g., randomly sampling 128x128 pixels to reduce memory usage [S1]).
*   Curriculum learning can help scale training by gradually increasing complexity [S9].

**Strengths and Limitations:**
*   **Strengths:** Provides a unified framework for optimizing complex radiance field models. Balances multiple objectives to prevent overfitting to noise or artifacts.
*   **Limitations:** Requires careful tuning of hyperparameters ($\lambda$ values). Computationally expensive due to multiple loss evaluations. Sensitive to initialization and learning rate schedules [S1].

**When to Use:**
*   Use when optimizing radiance fields that require multiple constraints (photometric, geometric, regularization).
*   Use when standard photometric loss leads to artifacts or unstable training.

**When Not to Use:**
*   Use simple photometric loss if computational resources are extremely limited and high fidelity is not required.
*   Avoid if the scene is well-constrained and regularization is not needed.

## Background

The Total Loss Function originated with the development of neural volume rendering and implicit surface reconstruction. Key early works include Neural Volumes [S1] (2019), which introduced a composite loss with KL divergence and total variation, and NeuS [S2, unofficial] (2021/2023), which popularized the combination of photometric and Eikonal losses. The concept has since been adapted for various radiance field representations, including Gaussian Splatting [S8][S11, unofficial], HybridNeRF [S5], and RocNet [S10]. The method is widely used in modern differentiable rendering pipelines to ensure stable and high-quality 3D reconstruction.

## Sources

[S1] Neural Volumes: Learning Dynamic Renderable Volumes from Images, https://arxiv.org/abs/1906.07751, fetched 2026-10-26
[S2] Point-NeuS: Point-Guided Neural Implicit Surface Reconstruction by Volume Rendering, https://doi.org/10.48550/arXiv.2310.07997, fetched 2026-10-30 (unofficial)
[S3] JackZhouSz/tlir, https://github.com/JackZhouSz/tlir, fetched 2026-10-27 (unofficial)
[S4] Direct Voxel Grid Optimization: Super-fast Convergence for Radiance Fields Reconstruction, https://doi.org/10.1109/cvpr52688.2022.00538, fetched 2026-10-26
[S5] HybridNeRF: Efficient Neural Rendering via Adaptive Volumetric Surfaces, https://doi.org/10.1109/cvpr52733.2024.01858, fetched 2026-10-26
[S6] nerfstudio-project/gsplat, https://github.com/nerfstudio-project/gsplat, fetched 2026-10-27 (unofficial)
[S7] barbararoessle/ganerf, https://github.com/barbararoessle/ganerf, fetched 2026-10-27 (unofficial)
[S8] EVolSplat: Efficient Volume-based Gaussian Splatting for Urban View Synthesis, https://doi.org/10.1109/CVPR52734.2025.01054, fetched 2026-11-03
[S9] DBARF: Deep Bundle-Adjusting Generalizable Neural Radiance Fields, https://doi.org/10.1109/cvpr52729.2023.00011, fetched 2026-10-29
[S10] RocNet: Recursive Octree Network for Efficient 3D Deep Representation, https://doi.org/10.1109/3dv50981.2020.00051, fetched 2026-10-26
[S11] Mobile-GS: Real-time Gaussian Splatting for Mobile Devices, https://doi.org/10.48550/arXiv.2603.11531, fetched 2026-10-29 (unofficial)
