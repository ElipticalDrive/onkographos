# Loss Function Configuration

Also called: Loss Weighting Scheme, Total Training Loss, Training Loss Weights

## Overview
Loss Function Configuration defines the composite objective function and hyperparameter weights used to supervise the optimization of differentiable radiance fields. It combines geometric reconstruction fidelity, perceptual quality metrics, and regularization terms to guide the neural network or primitive-based representation toward the desired visual output. It differs from simple reconstruction objectives by integrating high-level perceptual metrics (SSIM, LPIPS) and specific regularization constraints (normal consistency, depth smoothness) to stabilize training and improve generalization.

## How it works
The total loss $\mathcal{L}$ is a weighted sum of individual loss components. The specific components and weights depend on the rendering representation (e.g., Gaussian Splatting, Voxel Grids) and the training stage (coarse vs. fine).

### Inputs and Outputs
- **Inputs**: Rendered image $I$, ground-truth target image $\hat{I}$, predicted normals $\mathbf{n}$, target normals $\hat{\mathbf{n}}$, predicted depth $d$, target depth $\hat{d}$, and transparent mask $m$.
- **Outputs**: Scalar loss value $\mathcal{L}$ and its gradient with respect to the scene parameters.

### Data Structures
No specific data structures are required beyond standard tensors for images, depths, and normals. The configuration is defined by scalar hyperparameters.

### Formulas and Parameters

#### 1. Joint Reflection-Transmission Model (RT-Splatting)
Used for training joint reflection-transmission Gaussian splatting models [S1, unofficial].

The total loss is:
$$ \mathcal{L} = \mathcal{L}_{\text{img}} + \lambda_{\text{n}}\mathcal{L}_{\text{n}} + \lambda_{\text{mask}}\mathcal{L}_{\text{mask}} $$

Where:
- $\mathcal{L}_{\text{img}}$ is the appearance loss.
- $\mathcal{L}_{\text{n}}$ is the normal consistency loss.
- $\mathcal{L}_{\text{mask}}$ is the transparent mask regularization loss.
- $\lambda_{\text{n}}$ is the normal loss weight.
- $\lambda_{\text{mask}}$ is the mask loss weight.

The appearance loss $\mathcal{L}_{\text{img}}$ is defined as:
$$ \mathcal{L}_{\text{img}} = (1 - \lambda)\mathcal{L}_{1} + \lambda \mathcal{L}_{\text{D-SSIM}} + \lambda_{\text{perc}}\mathcal{L}_{\text{perc}} $$

Where:
- $\mathcal{L}_{1}$ is the L1 loss.
- $\mathcal{L}_{\text{D-SSIM}}$ is the D-SSIM loss.
- $\mathcal{L}_{\text{perc}}$ is the perceptual loss from EnvGS.
- $\lambda$ is the D-SSIM weight.
- $\lambda_{\text{perc}}$ is the perceptual loss weight.

**Typical Values [S1, unofficial]:**
- $\lambda = 0.2$
- $\lambda_{\text{perc}} = 0.01$
- $\lambda_{\text{n}} = 0.05$
- $\lambda_{\text{mask}} = 0.01$

#### 2. Dynamic View Synthesis (Neural Deformable Voxel Grid)
Used for dynamic view synthesis with coarse-to-fine training [S2].

The total loss is:
$$ \mathcal{L} = \mathcal{L}_{\text{photo}} + w_{\text{ptc}} \cdot \mathcal{L}_{\text{ptc}} + w_{\text{bg}} \cdot \mathcal{L}_{\text{bg}} + w_{d\_norm} \cdot \mathcal{L}_{d\_norm} + w_{d\_tv} \cdot \mathcal{L}_{d\_tv} $$

Where:
- $\mathcal{L}_{\text{photo}}$ is the photometric loss.
- $\mathcal{L}_{\text{ptc}}$ is the photometric consistency loss.
- $\mathcal{L}_{\text{bg}}$ is the background loss.
- $\mathcal{L}_{d\_norm}$ is the deformation norm regularization loss.
- $\mathcal{L}_{d\_tv}$ is the total variation loss on depth.
- $w_{\text{ptc}}, w_{\text{bg}}, w_{d\_norm}, w_{d\_tv}$ are the respective weights.

**Typical Values [S2]:**
- **Coarse Stage**: $w_{\text{ptc}}=0.1$, $w_{\text{bg}}=0.01$, $w_{d\_norm}=0.1$, $w_{d\_tv}=1$.
- **Fine Stage**: $w_{\text{ptc}}=0.01$, $w_{\text{bg}}=0.001$, $w_{d\_norm}=0.01$, $w_{d\_tv}=1$.
- **Large Motion Scenes** ('Hell Warrior', 'Jumping Jacks', 'T-Rex'): $w_{d\_norm}$ is reduced to $0.01$ (coarse) and $0.001$ (fine).

#### 3. Feed-Forward 3D Gaussian Splatting (SplatVoxel)
Used for training feed-forward networks with multi-view and voxel transformers [S4].

The loss function is:
$$ \mathcal{L} = \mathcal{L}_{\text{MSE}}(\hat{I}, I) + \lambda \mathcal{L}_{\text{LPIPS}}(\hat{I}, I) $$

Where:
- $\hat{I}$ is the ground-truth target image.
- $I$ is the rendered target image.
- $\mathcal{L}_{\text{MSE}}$ is the Mean Squared Error loss.
- $\mathcal{L}_{\text{LPIPS}}$ is the Learned Perceptual Image Patch Similarity loss.
- $\lambda$ is the LPIPS weight.

**Typical Values [S4]:**
- Multi-view transformer: $\lambda = 0.5$.
- Voxel transformer: $\lambda = 4.0$.
- Batch size: 128.
- Total iterations: 300K.

### Pseudocode
The following pseudocode illustrates the configuration of loss weights for a generic differentiable renderer, incorporating the specific strategies from the sources.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_total_loss(rendered_image, gt_image, normals, gt_normals, 
                       depth, gt_depth, mask, config):
    """
    Computes the total loss based on the configuration.
    
    Args:
        rendered_image: Tensor of shape (H, W, 3)
        gt_image: Tensor of shape (H, W, 3)
        normals: Tensor of predicted normals
        gt_normals: Tensor of ground-truth normals
        depth: Tensor of predicted depth
        gt_depth: Tensor of ground-truth depth
        mask: Tensor of transparent mask
        config: Dictionary containing loss weights and stage info
        
    Returns:
        Total loss scalar
    """
    total_loss = 0.0
    
    # 1. Photometric/Appearance Loss
    # Supports L1, D-SSIM, MSE, LPIPS based on config
    if config.get('use_l1', False):
        l1_loss = torch.mean(torch.abs(rendered_image - gt_image))
        total_loss += (1 - config['lambda_dssim']) * l1_loss
        
    if config.get('use_dssim', False):
        # D-SSIM loss implementation assumed available
        dssim_loss = compute_dssim_loss(rendered_image, gt_image)
        total_loss += config['lambda_dssim'] * dssim_loss
        
    if config.get('use_mse', False):
        mse_loss = torch.mean((rendered_image - gt_image) ** 2)
        total_loss += mse_loss
        
    if config.get('use_lpips', False):
        lpips_loss = compute_lpips_loss(rendered_image, gt_image)
        total_loss += config['lambda_lpips'] * lpips_loss
        
    # 2. Normal Consistency Loss (RT-Splatting)
    if config.get('use_normal_loss', False):
        normal_loss = torch.mean((normals - gt_normals) ** 2)
        total_loss += config['lambda_n'] * normal_loss
        
    # 3. Transparent Mask Regularization (RT-Splatting)
    if config.get('use_mask_loss', False):
        mask_loss = compute_mask_regularization(mask)
        total_loss += config['lambda_mask'] * mask_loss
        
    # 4. Dynamic/Deformation Losses (Neural Deformable Voxel Grid)
    if config.get('stage') == 'coarse':
        w_ptc = 0.1
        w_bg = 0.01
        w_d_norm = 0.1
        w_d_tv = 1.0
    elif config.get('stage') == 'fine':
        w_ptc = 0.01
        w_bg = 0.001
        w_d_norm = 0.01
        w_d_tv = 1.0
    else:
        w_ptc = config.get('w_ptc', 0.1)
        w_bg = config.get('w_bg', 0.01)
        w_d_norm = config.get('w_d_norm', 0.1)
        w_d_tv = config.get('w_d_tv', 1.0)
        
    # Adjust for large motion scenes
    if config.get('large_motion', False):
        if config.get('stage') == 'coarse':
            w_d_norm = 0.01
        elif config.get('stage') == 'fine':
            w_d_norm = 0.001
            
    if config.get('use_ptc', False):
        ptc_loss = compute_photometric_consistency(rendered_image, gt_image)
        total_loss += w_ptc * ptc_loss
        
    if config.get('use_bg', False):
        bg_loss = compute_background_loss(rendered_image, gt_image)
        total_loss += w_bg * bg_loss
        
    if config.get('use_d_norm', False):
        d_norm_loss = compute_deformation_norm(depth)
        total_loss += w_d_norm * d_norm_loss
        
    if config.get('use_d_tv', False):
        d_tv_loss = compute_total_variation(depth)
        total_loss += w_d_tv * d_tv_tv_loss
        
    return total_loss
```

### Edge Cases and Pitfalls
- **Hybrid Gradients**: When initial and target shapes are already aligned very well, the approximated optimal transport method used in RGBXY derivatives may lead to inaccurate matching, making RGBXY derivatives less effective [S3]. In such cases, switching to standard derivatives is recommended.
- **Large Motion Scenes**: For scenes with large motion (e.g., 'Hell Warrior', 'Jumping Jacks', 'T-Rex'), the deformation norm regularization weight $w_{d\_norm}$ must be reduced to prevent over-regularization [S2].
- **Precision**: The choice of loss weights significantly impacts convergence. Incorrect weights can lead to poor geometry or artifacts.

## Subtopics

### [Overall Training Objective](13444-overall-training-objective.md)
The Overall Training Objective defines the composite loss function and optimization schedule used to train differentiable radiance fields, specifically in the context of text-driven stylization and multi-baseline reconstruction. It combines geometric reconstruction fidelity with perceptual and semantic constraints to guide the neural network toward the desired visual output. It differs from simple reconstruction objectives by integrating high-level perceptual metrics (SSIM, LPIPS) and, in stylization variants, contrastive text-image alignment losses. [13444-overall-training-objective.md]

### Hybrid Gradients
A strategy combining RGBXY derivatives and standard derivatives to leverage global matching in early stages and precise local alignment in late stages. Use RGBXY derivatives for the first 75% of optimization iterations, then switch to standard derivatives for the last 25%. Pitfall: When initial and target shapes are already aligned very well, the approximated optimal transport method may lead to inaccurate matching, making RGBXY derivatives less effective. [S3]

### Two-Stage Photometric and Perceptual Training Strategy
A two-stage training strategy for the SplatVoxel feed-forward networks using photometric loss. Stage 1: Train the multi-view Transformer backbone for 200K iterations. Stage 2: Jointly fine-tune both the multi-view and voxel Transformers for an additional 100K iterations. Loss function: $\mathcal{L} = \mathcal{L}_{\text{MSE}}(\hat{I}, I) + \lambda \mathcal{L}_{\text{LPIPS}}(\hat{I}, I)$. LPIPS weight for multi-view transformer: $\lambda = 0.5$. LPIPS weight for voxel transformer: $\lambda = 4.0$. Batch size: 128. Total iterations: 300K. Trained on multi-view static scene datasets [87, 35], specifically DL3DV [35] for cross-dataset generalization. [S4]

## Operating characteristics
- **Time Complexity**: The computation of perceptual losses (LPIPS, D-SSIM) adds overhead compared to simple L1/MSE losses. The hybrid gradient strategy requires managing two derivative computations, but the switch is based on iteration count, not per-step complexity.
- **Memory Complexity**: No additional memory structures are required beyond the standard scene representation and intermediate gradients.
- **Measured Performance**: In RT-Splatting, the configuration with $\lambda=0.2$ and $\lambda_{\text{perc}}=0.01$ provides a good trade-off between reconstruction fidelity and perceptual quality [S1, unofficial]. In Neural Deformable Voxel Grid, the coarse-to-fine weighting scheme allows for faster convergence and better handling of dynamic scenes [S2].
- **Strengths**: Integrates multiple constraints for robust training. Adapts to different scene characteristics (e.g., large motion).
- **Limitations**: Requires careful tuning of hyperparameters. Perceptual losses can be computationally expensive.
- **When to Use**: When training differentiable radiance fields for high-quality view synthesis, especially with dynamic scenes or reflective/transmissive materials.
- **When Not to Use**: For simple static scenes where basic photometric loss is sufficient, or when computational resources are extremely limited.

## Background
Loss function configuration has evolved alongside differentiable rendering techniques. Early methods relied primarily on photometric losses (L1, MSE). As the field advanced, perceptual losses (SSIM, LPIPS) and regularization terms (normal consistency, depth smoothness) were introduced to improve geometric and visual quality [S1, unofficial][S2, S4]. The specific configurations discussed here are derived from recent works on Gaussian Splatting [S1, unofficial][S4], Neural Deformable Voxel Grids [S2], and differentiable rendering with optimal transport [S3].

## Sources
[S1] RT-Splatting: Joint Reflection-Transmission Modeling with Gaussian Splatting, https://github.com/WoojunePark/splat_papers/issues/71, fetched 2026-10-27 (unofficial)
[S2] Neural Deformable Voxel Grid for Fast Optimization of Dynamic View Synthesis, https://doi.org/10.1007/978-3-031-26319-4_27, fetched 2026-10-26
[S3] Differentiable Rendering Using RGBXY Derivatives and Optimal Transport, https://doi.org/10.1145/3550454.3555479, fetched 2026-10-27
[S4] Learning Efficient Fuse-and-Refine for Feed-Forward 3D Gaussian Splatting, https://arxiv.org/abs/2503.14698v2, fetched 2026-10-27
