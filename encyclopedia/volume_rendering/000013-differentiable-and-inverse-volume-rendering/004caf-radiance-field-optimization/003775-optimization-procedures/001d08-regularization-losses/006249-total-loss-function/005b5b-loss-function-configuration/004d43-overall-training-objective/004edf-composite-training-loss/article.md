# Composite Training Loss

Also called: L1+SSIM+LPIPS Loss

## Overview
Composite training loss functions aggregate multiple error metrics to optimize the parameters of a radiance field or inverse rendering model. They address the ill-posed nature of inverse problems by combining pixel-wise accuracy, structural similarity, and perceptual fidelity. This method differs from single-metric approaches by balancing reconstruction error with high-frequency detail preservation, often incorporating specific priors (such as normals or semantics) depending on the variant.

## How it works
The general approach involves computing individual loss terms between the rendered image and the ground truth (or estimated priors), then summing them with learned or fixed weights. The specific composition varies significantly by variant, ranging from simple photometric/perceptual combinations to complex inverse rendering objectives involving geometry and lighting constraints.

### Inputs and Outputs
- **Inputs**: Rendered image $I_{render}$, ground truth image $I_{gt}$, and optionally auxiliary data such as estimated normals, semantic maps, or shadow masks.
- **Outputs**: A scalar loss value $\mathcal{L}$ used for gradient-based optimization.

### Data Structures
No specific complex data structures are required beyond standard image tensors and scalar weights. Some variants require pre-computed auxiliary maps (e.g., normals from Omnidata [S2], semantic maps from MMSegmentation [S2]).

### Steps
1. **Render**: Generate the image $I_{render}$ from the current model parameters.
2. **Compute Auxiliary Priors (Variant-dependent)**: If required, estimate normals, semantics, or shadow masks using pretrained networks [S2].
3. **Compute Individual Loss Terms**: Calculate each component loss (L1, SSIM, LPIPS, etc.) between $I_{render}$ and $I_{gt}$ (or priors).
4. **Aggregate**: Sum the weighted loss terms to produce the total loss.

### Formulas
The specific formula depends on the variant:

**General Composite (L1+SSIM+LPIPS):**
$$ \mathcal{L} = \mathcal{L}_{L1} + \mathcal{L}_{SSIM} + \mathcal{L}_{LPIPS} $$
Where $\mathcal{L}_{L1}$ is the mean absolute error, $\mathcal{L}_{SSIM}$ is the structural similarity index loss, and $\mathcal{L}_{LPIPS}$ is the learned perceptual image patch similarity loss [S1].

**UrbanIR:**
$$ \min _{\theta , \mathbf{L}} \mathcal {L}_{\text {render}} + \lambda _1\mathcal {L}_{\text {visibility}} + \lambda _2\mathcal {L}_{\text {normal}} + \lambda _3\mathcal {L}_{\text {semantics}} + \lambda _4\mathcal {L}_{\text {reg}} $$
Where $\mathcal{L}_{\text{render}}$ is the base rendering loss, $\mathcal{L}_{\text{visibility}}$ enforces visibility consistency, $\mathcal{L}_{\text{normal}}$ aligns with estimated normals, $\mathcal{L}_{\text{semantics}}$ aligns with semantic maps, and $\mathcal{L}_{\text{reg}}$ is a regularization term [S2].

**SVG-IR:**
$$ \mathcal{L} = \lambda_1 \mathcal{L}_{L1} + \lambda_{ssim} \mathcal{L}_{ssim} + \lambda_{rc} \mathcal{L}_{rc} + \lambda_n \mathcal{L}_n + \lambda_{s,a} \mathcal{L}_{s,a} + \lambda_{s,r} \mathcal{L}_{s,r} + \lambda_{reg,n} \mathcal{L}_{reg,n} $$
Where $\mathcal{L}_{rc}$ is radiance consistency loss, $\mathcal{L}_n$ is $1 - \text{cosine similarity}$ between normal and pseudo normal, $\mathcal{L}_{s,a}$ and $\mathcal{L}_{s,r}$ are TV losses on albedo and roughness, and $\mathcal{L}_{reg,n}$ is the L2 regular term of normal offsets $\Delta N_M$ [S3].

**INPC:**
$$ \mathcal{L} = \mathcal{L}_R + \mathcal{L}_{D-SSIM} + \lambda_{vgg} \cdot \mathcal{L}_{VGG} + \lambda_{decay} \cdot \mathcal{L}_{Reg} $$
Where $\mathcal{L}_R$ is a robust per-pixel loss, $\mathcal{L}_{D-SSIM}$ is differentiable SSIM, $\mathcal{L}_{VGG}$ is VGG perceptual loss, and $\mathcal{L}_{Reg}$ is normalized weight decay on hash grid parameters [S4].

### Parameters
- **UrbanIR**: $\lambda_1 = 0.001$, $\lambda_2 = 0.01$, $\lambda_3 = 0.04$, $\lambda_4 = 0.1$ [S2].
- **SVG-IR**: Weights $\lambda_1, \lambda_{ssim}, \lambda_{rc}, \lambda_n, \lambda_{s,a}, \lambda_{s,r}, \lambda_{reg,n}$ are used but specific values are not provided in the source text [S3].
- **INPC**: $\lambda_{vgg} = 0.075$, $\lambda_{decay} = 0.1$, robust loss parameters $\alpha = 0$, $c = 0.2$ [S4].
- **General (L1+SSIM+LPIPS)**: Weights are not explicitly specified as distinct scalars in the source, implying equal weighting or default implementations [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_composite_loss(rendered_image, ground_truth, variant="standard", aux_data=None):
    """
    Computes the composite training loss based on the specified variant.
    
    Args:
        rendered_image (Tensor): Rendered RGB image.
        ground_truth (Tensor): Ground truth RGB image.
        variant (str): One of "standard", "urbanir", "svgir", "inpc".
        aux_data (dict): Optional auxiliary data (normals, semantics, etc.).
        
    Returns:
        Tensor: Scalar loss value.
    """
    if variant == "standard":
        # L1 + SSIM + LPIPS
        # Note: Specific weights for L1, SSIM, LPIPS are not given in [S1], 
        # assuming equal contribution or standard implementation defaults.
        loss_l1 = l1_loss(rendered_image, ground_truth)
        loss_ssim = ssim_loss(rendered_image, ground_truth)
        loss_lpips = lpips_loss(rendered_image, ground_truth)
        return loss_l1 + loss_ssim + loss_lpips

    elif variant == "urbanir":
        # Minimize: L_render + lambda1*L_visibility + lambda2*L_normal + lambda3*L_semantics + lambda4*L_reg
        lambda1 = 0.001
        lambda2 = 0.01
        lambda3 = 0.04
        lambda4 = 0.1
        
        loss_render = l1_loss(rendered_image, ground_truth) # Assumed base rendering loss
        loss_visibility = compute_visibility_loss(rendered_image, ground_truth) # Implementation detail [G]
        loss_normal = normal_loss(rendered_image, aux_data['normals']) # Implementation detail [G]
        loss_semantics = semantic_loss(rendered_image, aux_data['semantics']) # Implementation detail [G]
        loss_reg = regularization_loss(rendered_image) # Implementation detail [G]
        
        return loss_render + lambda1 * loss_visibility + lambda2 * loss_normal + lambda3 * loss_semantics + lambda4 * loss_reg

    elif variant == "svgir":
        # L = lambda1*L1 + lambda_ssim*Lssim + lambda_rc*Lrc + lambda_n*Ln + lambda_s,a*Ls,a + lambda_s,r*Ls,r + lambda_reg,n*Lreg,n
        # Weights are not provided in [S3]
        lambda1 = 1.0 # Placeholder [G]
        lambda_ssim = 1.0 # Placeholder [G]
        lambda_rc = 1.0 # Placeholder [G]
        lambda_n = 1.0 # Placeholder [G]
        lambda_s_a = 1.0 # Placeholder [G]
        lambda_s_r = 1.0 # Placeholder [G]
        lambda_reg_n = 1.0 # Placeholder [G]
        
        loss_l1 = l1_loss(rendered_image, ground_truth)
        loss_ssim = ssim_loss(rendered_image, ground_truth)
        loss_rc = radiance_consistency_loss(rendered_image, aux_data['radiance']) # Implementation detail [G]
        loss_n = 1.0 - cosine_similarity(rendered_image['normal'], aux_data['pseudo_normal']) # Implementation detail [G]
        loss_s_a = tv_loss(rendered_image['albedo']) # Implementation detail [G]
        loss_s_r = tv_loss(rendered_image['roughness']) # Implementation detail [G]
        loss_reg_n = l2_loss(rendered_image['normal_offsets']) # Implementation detail [G]
        
        return (lambda1 * loss_l1 + lambda_ssim * loss_ssim + lambda_rc * loss_rc + 
                lambda_n * loss_n + lambda_s_a * loss_s_a + lambda_s_r * loss_s_r + 
                lambda_reg_n * loss_reg_n)

    elif variant == "inpc":
        # L = LR + LD-SSIM + lambda_vgg * LVGG + lambda_decay * LReg
        lambda_vgg = 0.075
        lambda_decay = 0.1
        alpha = 0
        c = 0.2
        
        loss_r = robust_loss(rendered_image, ground_truth, alpha=alpha, c=c)
        loss_d_ssim = d_ssim_loss(rendered_image, ground_truth)
        loss_vgg = vgg_loss(rendered_image, ground_truth)
        loss_reg = normalized_weight_decay_loss(rendered_image['hash_grid_params'])
        
        return loss_r + loss_d_ssim + lambda_vgg * loss_vgg + lambda_decay * loss_reg

    else:
        raise ValueError("Unknown variant")
```

### Edge Cases and Pitfalls
- **Auxiliary Data Availability**: UrbanIR and SVG-IR require pretrained networks (MTMT, Omnidata, MMSegmentation) which may fail on out-of-distribution data [S2, S3].
- **Weight Tuning**: Incorrect weighting of perceptual vs. pixel-wise losses can lead to blurry or artifact-ridden results. INPC uses specific weights for VGG and decay [S4].
- **Robust Loss Parameters**: INPC uses a robust loss with specific parameters ($\alpha=0, c=0.2$) to handle outliers [S4].

## Subtopics

### UrbanIR Training Objective
An optimization objective combining rendering loss, visibility, normal, semantic, and regularization losses, using pretrained 2D priors to address the ill-posed inverse problem. It estimates shadow masks using MTMT, refines geometry using Omnidata, and uses semantic maps from Kitti360 or MMSegmentation. It is chosen for large-scale urban scene inverse rendering from single videos [S2].

### SVG-IR Training Loss
A composite loss including L1, SSIM, radiance consistency, normal cosine similarity, TV losses on albedo/roughness, and normal offset regularization. It is designed for spatially-varying Gaussian splatting for inverse rendering, handling albedo-lighting ambiguity. It is chosen when detailed material and lighting decomposition is required [S3].

### INPC Optimization Loss
A composite loss combining a robust per-pixel loss, D-SSIM, VGG perceptual loss, and normalized weight decay on multi-resolution hash grid parameters. It is used for Implicit Neural Point Clouds, balancing perceptual quality with regularization of the hash grid. It is chosen for high-quality radiance field rendering with fixed model size [S4].

## Operating characteristics
- **Time Complexity**: Depends on the number of loss terms and the complexity of auxiliary networks (e.g., VGG, Omnidata). UrbanIR and SVG-IR are computationally heavier due to auxiliary priors [S2, S3].
- **Memory Complexity**: INPC uses a fixed-size hash grid (1.1 GB) [S4]. UrbanIR uses Instant-NGP style hash tables [S2].
- **Performance**: INPC achieves high PSNR and LPIPS scores on MipNeRF360 and Tanks and Temples datasets [S4]. SVG-IR achieves SOTA relighting results on synthetic data [S3].
- **Hardware**: INPC and SVG-IR tested on RTX 4090 [S3, S4].
- **Strengths**: Combines multiple metrics for robust optimization. Handles ill-posed problems via priors (UrbanIR).
- **Limitations**: Requires careful tuning of weights. Auxiliary networks add complexity and potential failure modes.

## Background
Composite training losses are standard in differentiable rendering and inverse rendering to balance photometric accuracy with perceptual quality. The L1+SSIM+LPIPS combination is a common baseline [S1]. UrbanIR extends this with semantic and normal priors for urban scenes [S2]. SVG-IR adds radiance consistency and material-specific losses for Gaussian splatting [S3]. INPC uses robust losses and hash grid regularization for neural point clouds [S4].

## Sources
[S1] MuRF: Multi-Baseline Radiance Fields, https://doi.org/10.1109/cvpr52733.2024.01894, fetched 2026-10-27
[S2] UrbanIR: Large-Scale Urban Scene Inverse Rendering from a Single Video, https://doi.org/10.1109/3dv66043.2025.00052, fetched 2026-10-30
[S3] SVG-IR: Spatially-Varying Gaussian Splatting for Inverse Rendering, https://doi.org/10.1109/cvpr52734.2025.01505, fetched 2026-10-29
[S4] INPC: Implicit Neural Point Clouds for Radiance Field Rendering, https://doi.org/10.1109/3dv66043.2025.00021, fetched 2026-10-26
