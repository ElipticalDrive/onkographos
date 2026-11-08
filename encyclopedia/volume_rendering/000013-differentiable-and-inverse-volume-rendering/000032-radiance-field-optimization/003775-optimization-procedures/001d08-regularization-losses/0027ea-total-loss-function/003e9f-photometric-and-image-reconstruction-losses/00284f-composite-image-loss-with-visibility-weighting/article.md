# Composite Image Loss with Visibility Weighting

Also called: Optimization Loss Function, Visibility-weighted Photometric Loss

## Overview
This method defines the composite image reconstruction loss $\mathcal{L}_c$ used to supervise the training of radiance field models by combining pixel-wise fidelity metrics with perceptual quality measures. It solves the problem of unreliable supervision in occluded or background regions by weighting photometric terms (L1 and SSIM) with a visibility map $\mathbf{V}_M$, while applying perceptual loss (LPIPS) to the unmasked rendered image to preserve global structure. It sets itself apart from its sibling [Composite Photometric and Regularization Loss](9355-composite-photometric-and-regularization-loss.md) by explicitly integrating the visibility-weighted masking strategy into the core photometric term definition within the context of the SuRFNet optimization framework [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Rendered image $\mathbf{I}_r$, ground-truth reference image $\mathbf{I}_{gt}$, visibility map $\mathbf{V}_M$, and regularization loss values $\mathcal{L}_{sc}$ (sampling coordinate regularization) and $\mathcal{L}_{vm}$ (visibility map regularization).
*   **Outputs:** Composite image loss $\mathcal{L}_c$ and total loss $\mathcal{L}$.

**Data Structures:**
*   $\mathbf{I}_r, \mathbf{I}_{gt}$: Images of identical resolution, typically 3-channel RGB tensors.
*   $\mathbf{V}_M$: Visibility map of the same spatial dimensions as the images. Values indicate the reliability of the ground-truth pixel for supervision (e.g., 1.0 for visible foreground, 0.0 for background/occluded). [G] The exact generation of $\mathbf{V}_M$ is defined in the parent article's shared principles or preceding steps; this method assumes it is provided.

**Parameters:**
*   $\lambda_1 = 0.8$: Weight for L1 loss [S1].
*   $\lambda_{SSIM} = 0.2$: Weight for SSIM loss [S1].
*   $\lambda_{LPIPS} = 0.005$: Weight for LPIPS loss [S1].
*   $\lambda_{sc} = 0.001$: Weight for sampling coordinate regularization [S1].
*   $\lambda_{vm} = 0.15$: Weight for visibility map regularization [S1].

**Formulas:**
The composite image loss $\mathcal{L}_c$ is calculated as:
$$
\mathcal{L}_c = \lambda_1 \mathcal{L}_1(\mathbf{V}_M \odot \mathbf{I}_r, \mathbf{V}_M \odot \mathbf{I}_{gt}) + \lambda_{SSIM} \mathcal{L}_{SSIM}(\mathbf{V}_M \odot \mathbf{I}_r, \mathbf{V}_M \odot \mathbf{I}_{gt}) + \lambda_{LPIPS} \mathcal{L}_{LPIPS}(\mathbf{I}_r, \mathbf{I}_{gt})
$$
where $\odot$ denotes the Hadamard (element-wise) product [S1].

The total loss $\mathcal{L}$ combines the composite image loss with regularization terms:
$$
\mathcal{L} = \mathcal{L}_c + \lambda_{sc} \mathcal{L}_{sc} + \lambda_{vm} \mathcal{L}_{vm}
$$
where $\mathcal{L}_{sc}$ and $\mathcal{L}_{vm}$ are defined in the preceding regularization steps [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_composite_image_loss(I_r, I_gt, V_M, L_sc, L_vm):
    """
    Computes the composite image loss and total loss for radiance field optimization.
    
    Args:
        I_r: Rendered image tensor (H, W, 3).
        I_gt: Ground-truth reference image tensor (H, W, 3).
        V_M: Visibility map tensor (H, W) or (H, W, 1). Values in [0, 1].
        L_sc: Sampling coordinate regularization loss (scalar).
        L_vm: Visibility map regularization loss (scalar).
        
    Returns:
        L_c: Composite image loss (scalar).
        L: Total loss (scalar).
    """
    # Hyperparameters from [S1]
    lambda_1 = 0.8
    lambda_ssim = 0.2
    lambda_lpips = 0.005
    lambda_sc = 0.001
    lambda_vm = 0.15

    # Mask images using Hadamard product
    # V_M is broadcasted to match I_r/I_gt dimensions if necessary
    I_r_masked = I_r * V_M
    I_gt_masked = I_gt * V_M

    # Compute L1 loss on masked images
    # L1 is mean absolute error
    L1_loss = torch.mean(torch.abs(I_r_masked - I_gt_masked))

    # Compute SSIM loss on masked images
    # SSIM is typically computed as 1 - SSIM_index to make it a loss
    # Note: SSIM implementation details (window size, channels) are assumed 
    # to be handled by a standard library function, as specific parameters 
    # are not given in [S1].
    ssim_index = compute_ssim(I_r_masked, I_gt_masked)
    SSIM_loss = 1.0 - ssim_index

    # Compute LPIPS loss on unmasked images
    # LPIPS measures perceptual difference using deep features
    LPIPS_loss = compute_lpips(I_r, I_gt)

    # Calculate Composite Image Loss L_c
    L_c = (lambda_1 * L1_loss) + \
          (lambda_ssim * SSIM_loss) + \
          (lambda_lpips * LPIPS_loss)

    # Calculate Total Loss L
    L = L_c + \
        (lambda_sc * L_sc) + \
        (lambda_vm * L_vm)

    return L_c, L
```

**Edge Cases and Pitfalls:**
*   **Visibility Map Boundaries:** If $\mathbf{V}_M$ contains values strictly between 0 and 1, the L1 and SSIM terms are interpolated between full supervision and no supervision. Ensure $\mathbf{V}_M$ is normalized correctly relative to the image intensity range [S1].
*   **LPIPS Unmasked:** The LPIPS term is applied to the full images $\mathbf{I}_r$ and $\mathbf{I}_{gt}$, not the masked versions. This ensures perceptual quality is maintained even in regions where photometric supervision is weak or absent [S1].
*   **Gradient Flow:** The Hadamard product with $\mathbf{V}_M$ masks gradients for pixels where $\mathbf{V}_M=0$. Ensure the visibility map is differentiable or that gradients are correctly zeroed out during backpropagation [S1].

## Subtopics

### [SRF-Loss](6978-srf-loss.md)
SRF-Loss is the composite objective function used to train SuRFNet for generating high-quality Sparse Radiance Fields (SRFs) from partial or noisy inputs. It addresses the problem of ensuring that generated 3D radiance fields not only match the input density and color statistics but also produce visually accurate renderings from novel views. It sets itself apart from simple pixel-wise reconstruction losses by incorporating a 3D-supervised density/radiance component and an online perceptual loss that explicitly optimizes for rendering quality.

## Operating characteristics
*   **Complexity:** The computational cost is dominated by the SSIM and LPIPS calculations, which are more expensive than L1. LPIPS requires forward passes through a pre-trained network (e.g., VGG or AlexNet), making it the most costly term [S1].
*   **Performance:** The specific weights ($\lambda_1=0.8, \lambda_{SSIM}=0.2, \lambda_{LPIPS}=0.005$) were tuned to balance pixel fidelity and perceptual quality, resulting in improved PSNR and SSIM scores compared to baselines like Ha-NeRF and CR-NeRF on unconstrained image collections [S1].
*   **Strengths:** Effectively handles occlusion and background ambiguity by ignoring unreliable pixels in photometric terms while preserving global structure via LPIPS [S1].
*   **Limitations:** Relies on the accuracy of the visibility map $\mathbf{V}_M$. If $\mathbf{V}_M$ is incorrect, supervision may be applied to wrong regions or withheld from correct ones. The LPIPS term adds significant computational overhead during training [S1].

## Background
This loss function was introduced in the context of "Gaussian in the Wild: 3D Gaussian Splatting for Unconstrained Image Collections" [S1]. It is part of the optimization procedure for SuRFNet, which generates Sparse Radiance Fields. The method builds on prior work using L1 and SSIM losses [S1, Ref 44] but extends it with visibility weighting and perceptual loss to handle the challenges of unconstrained image collections [S1].

## Sources
[S1] Gaussian in the Wild: 3D Gaussian Splatting for Unconstrained Image Collections, https://doi.org/10.1007/978-3-031-73116-7_20, fetched 2026-10-27
