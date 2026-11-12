# Loss Function Configuration

Also called: Loss Weighting Scheme, Training Loss Weights

## Overview
Loss function configuration defines the mathematical structure and hyperparameter weights of the total objective used to optimize radiance fields or Gaussian splatting representations. It combines appearance, geometric, and regularization terms to supervise the joint reflection-transmission model or feed-forward networks. It differs from sibling articles by focusing specifically on the *composition* of the loss equation and the *values* of the weighting coefficients, rather than the derivation of individual loss components (like Total Variation) or the broader training schedule.

## How it works
The total loss $\mathcal{L}$ is a weighted sum of distinct loss components. The specific components and weights depend on the rendering representation (e.g., Gaussian Splatting vs. Voxel Grid) and the scene characteristics (static vs. dynamic, reflective vs. transparent).

### Inputs and Outputs
- **Inputs**: Rendered images $\hat{I}$, ground-truth images $I$, predicted normals $\hat{n}$, ground-truth normals $n$, depth maps, and training iteration count $t$.
- **Outputs**: Scalar loss value $\mathcal{L}$ and its gradient with respect to the scene parameters.

### Data Structures
No specific complex data structures are required beyond standard image tensors and scalar hyperparameters.

### Formulas and Parameters
Two primary configurations are documented in the sources.

#### Configuration A: Joint Reflection-Transmission (RT-Splatting)
Used for Gaussian splatting models handling both reflection and transmission [S1, unofficial].

The total loss is:
$$ \mathcal{L} = \mathcal{L}_{\text{img}} + \lambda_{\text{n}}\mathcal{L}_{\text{n}} + \lambda_{\text{mask}}\mathcal{L}_{\text{mask}} $$

Where:
- $\mathcal{L}_{\text{img}}$ is the appearance loss.
- $\mathcal{L}_{\text{n}}$ is the normal consistency loss.
- $\mathcal{L}_{\text{mask}}$ is the transparent mask regularization loss.
- $\lambda_{\text{n}}$ is the normal loss weight.
- $\lambda_{\text{mask}}$ is the mask loss weight.

The appearance loss $\mathcal{L}_{\text{img}}$ is a combination of L1, D-SSIM, and perceptual losses:
$$ \mathcal{L}_{\text{img}} = (1 - \lambda)\mathcal{L}_{1} + \lambda \mathcal{L}_{\text{D-SSIM}} + \lambda_{\text{perc}}\mathcal{L}_{\text{perc}} $$

Where:
- $\mathcal{L}_{1}$ is the L1 loss.
- $\mathcal{L}_{\text{D-SSIM}}$ is the D-SSIM loss.
- $\mathcal{L}_{\text{perc}}$ is the perceptual loss (from EnvGS).
- $\lambda$ is the D-SSIM weight.
- $\lambda_{\text{perc}}$ is the perceptual loss weight.

**Typical Values [S1, unofficial]:**
- $\lambda = 0.2$
- $\lambda_{\text{perc}} = 0.01$
- $\lambda_{\text{n}} = 0.05$
- $\lambda_{\text{mask}} = 0.01$

#### Configuration B: Dynamic View Synthesis (Neural Deformable Voxel Grid)
Used for dynamic scenes with deformable voxel grids [S2].

The total loss is:
$$ \mathcal{L} = \mathcal{L}_{\text{photo}} + w_{\text{ptc}} \cdot \mathcal{L}_{\text{ptc}} + w_{\text{bg}} \cdot \mathcal{L}_{\text{bg}} + w_{\text{d\_norm}} \cdot \mathcal{L}_{\text{d\_norm}} + w_{\text{d\_tv}} \cdot \mathcal{L}_{\text{d\_tv}} $$

Where:
- $\mathcal{L}_{\text{photo}}$ is the photometric loss.
- $\mathcal{L}_{\text{ptc}}$ is the point consistency loss.
- $\mathcal{L}_{\text{bg}}$ is the background loss.
- $\mathcal{L}_{\text{d\_norm}}$ is the deformation norm regularization loss.
- $\mathcal{L}_{\text{d\_tv}}$ is the deformation total variation loss.
- $w_{\text{ptc}}, w_{\text{bg}}, w_{\text{d\_norm}}, w_{\text{d\_tv}}$ are the respective weights.

**Typical Values [S2]:**
- **Coarse Stage**: $w_{\text{ptc}}=0.1$, $w_{\text{bg}}=0.01$, $w_{\text{d\_norm}}=0.1$, $w_{\text{d\_tv}}=1$.
- **Fine Stage**: $w_{\text{ptc}}=0.01$, $w_{\text{bg}}=0.001$, $w_{\text{d\_norm}}=0.01$, $w_{\text{d\_tv}}=1$.
- **Large Motion Adjustment**: For scenes with large motion (e.g., 'Hell Warrior', 'Jumping Jacks', 'T-Rex'), $w_{\text{d\_norm}}$ is reduced to $0.01$ (coarse) and $0.001$ (fine).

### Pseudocode
The following pseudocode illustrates the calculation of the total loss for Configuration A (RT-Splatting), which is the most detailed configuration provided.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_loss_rt_splatting(rendered_image, gt_image, 
                              rendered_normals, gt_normals, 
                              transparent_mask, alpha, sigma, 
                              iteration, total_iterations):
    """
    Computes the total loss for RT-Splatting.
    
    Args:
        rendered_image: Tensor of shape (H, W, 3), rendered RGB image.
        gt_image: Tensor of shape (H, W, 3), ground truth RGB image.
        rendered_normals: Tensor of shape (H, W, 3), rendered surface normals.
        gt_normals: Tensor of shape (H, W, 3), ground truth normals.
        transparent_mask: Tensor of shape (H, W), binary mask for transparent regions.
        alpha: Tensor of shape (N, 3), optical opacity of Gaussians.
        sigma: Tensor of shape (N), geometric occupancy of Gaussians.
        iteration: Current training iteration.
        total_iterations: Total number of training iterations.
        
    Returns:
        total_loss: Scalar tensor.
    """
    
    # 1. Appearance Loss Components
    # L1 Loss
    l1_loss = mean_absolute_error(rendered_image, gt_image)
    
    # D-SSIM Loss (Assuming a standard D-SSIM implementation is available)
    # D-SSIM typically returns a value between 0 and 1, where 1 is identical.
    # Loss is usually 1 - D-SSIM.
    d_ssim_value = compute_d_ssim(rendered_image, gt_image)
    d_ssim_loss = 1.0 - d_ssim_value
    
    # Perceptual Loss (from EnvGS)
    # Requires feature extraction from a pretrained network
    perc_loss = compute_perceptual_loss(rendered_image, gt_image)
    
    # 2. Appearance Loss Combination
    lambda_d_ssim = 0.2
    lambda_perc = 0.01
    
    l_img = (1 - lambda_d_ssim) * l1_loss + lambda_d_ssim * d_ssim_loss + lambda_perc * perc_loss
    
    # 3. Normal Consistency Loss
    # Ensure normals are normalized before computing difference
    rendered_normals_norm = normalize(rendered_normals)
    gt_normals_norm = normalize(gt_normals)
    l_n = mean_squared_error(rendered_normals_norm, gt_normals_norm)
    
    # 4. Transparent Mask Regularization Loss
    # Applied only to transparent regions
    # Assuming L_mask is defined as MSE on the mask or related opacity/occupancy terms
    # Note: The exact definition of L_mask is not fully detailed in the source, 
    # but it is applied to transparent regions.
    # [G] Assuming L_mask is the MSE between rendered alpha and a target transparency,
    # masked by the transparent mask.
    l_mask = mean_squared_error(alpha, target_transparency) * transparent_mask
    
    # 5. Total Loss Combination
    lambda_n = 0.05
    lambda_mask = 0.01
    
    total_loss = l_img + lambda_n * l_n + lambda_mask * l_mask
    
    return total_loss
```

### Edge Cases and Pitfalls
- **Precision**: When initial and target shapes are already well-aligned, approximated optimal transport methods (used in some gradient calculations) may lead to inaccurate matching, making certain derivatives less effective [S3].
- **Boundaries**: In dynamic scenes, large motions require reduced weights for deformation norm regularization to prevent instability [S2].
- **Degenerate Input**: Background pixels do not cover any geometry and do not contribute to optimal transport gradients, but must be handled to avoid NaNs if not masked out [S3].

## Subtopics

### Hybrid Gradients
A strategy combining RGBXY derivatives and standard derivatives to leverage global matching in early stages and precise local alignment in late stages [S3]. Use RGBXY derivatives for the first 75% of optimization iterations and switch to standard derivatives for the last 25% [S3]. Pitfall: When initial and target shapes are already aligned very well, the approximated optimal transport method may lead to inaccurate matching, making RGBXY derivatives less effective [S3].

### Two-Stage Photometric and Perceptual Training Strategy
A two-stage training strategy for SplatVoxel feed-forward networks using photometric loss [S4]. Stage 1: Train the multi-view Transformer backbone for 200K iterations. Stage 2: Jointly fine-tune both the multi-view and voxel Transformers for an additional 100K iterations [S4]. Loss function: $\mathcal{L} = \mathcal{L}_{\text{MSE}}(\hat{I}, I) + \lambda \mathcal{L}_{\text{LPIPS}}(\hat{I}, I)$ [S4]. LPIPS weight $\lambda = 0.5$ for multi-view transformer and $\lambda = 4.0$ for voxel transformer [S4]. Batch size: 128. Total iterations: 300K [S4]. Used for cross-dataset generalization on DL3DV [S4].

## Operating characteristics
- **Time Complexity**: Depends on the number of loss components and the complexity of perceptual/SSIM calculations. Perceptual losses require forward passes through feature extractors, increasing compute time [S1, unofficial][S4].
- **Memory Complexity**: Low, as it primarily involves scalar weights and standard image tensors.
- **Scaling**: Weights often need adjustment for scene complexity (e.g., large motion scenes require smaller deformation weights) [S2].
- **Strengths**: Provides a unified objective for multi-component optimization (appearance, geometry, regularization).
- **Limitations**: Sensitive to weight selection; improper weights can lead to poor convergence or artifacts (e.g., floaters, specular artifacts) [S1, unofficial][S2].
- **When to Use**: When optimizing radiance fields or Gaussian splatting representations where multiple constraints (appearance, normal, depth, regularization) must be balanced.
- **When Not to Use**: For simple single-objective optimization tasks.

## Background
Loss function configuration is a fundamental aspect of differentiable rendering optimization. The specific configurations cited here originate from recent works in Gaussian Splatting (RT-Splatting [S1, unofficial], SplatVoxel [S4]) and dynamic view synthesis (Neural Deformable Voxel Grid [S2]). The hybrid gradient strategy comes from work on RGBXY derivatives [S3]. These methods represent the evolution of loss design from simple photometric reconstruction to complex, multi-modal objectives incorporating perceptual, geometric, and regularization terms.

## Sources
[S1] RT-Splatting: Joint Reflection-Transmission Modeling with Gaussian Splatting, https://github.com/WoojunePark/splat_papers/issues/71, fetched 2026-10-27 (unofficial)
[S2] Neural Deformable Voxel Grid for Fast Optimization of Dynamic View Synthesis, https://doi.org/10.1007/978-3-031-26319-4_27, fetched 2026-10-26
[S3] Differentiable Rendering Using RGBXY Derivatives and Optimal Transport, https://doi.org/10.1145/3550454.3555479, fetched 2026-10-27
[S4] Learning Efficient Fuse-and-Refine for Feed-Forward 3D Gaussian Splatting, https://arxiv.org/abs/2503.14698v2, fetched 2026-10-27
