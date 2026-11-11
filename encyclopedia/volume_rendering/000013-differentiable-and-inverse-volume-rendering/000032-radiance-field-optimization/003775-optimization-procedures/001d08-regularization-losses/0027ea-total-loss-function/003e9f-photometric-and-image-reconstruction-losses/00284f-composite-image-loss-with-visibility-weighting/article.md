# Composite Image Loss with Visibility Weighting

Also called: Optimization Loss Function, Visibility-weighted Photometric Loss

## Overview
This method computes the primary photometric reconstruction loss for differentiable volume rendering by weighting pixel-wise errors with a visibility map $V_M$. It solves the problem of supervising rendered images against ground-truth references in scenes with partial observability or occlusions, ensuring that gradients are only propagated through visible regions. It sets itself apart from standard photometric losses by explicitly masking the L1 and SSIM terms with visibility, while retaining the perceptual LPIPS term unmasked to preserve global structural consistency [S1].

## How it works
**Inputs and Outputs:**
- Inputs: Rendered image $\mathbf{I}_r$, ground-truth image $\mathbf{I}_{gt}$, visibility map $\mathbf{V}_M$.
- Output: Composite image loss scalar $L_c$.

**Data Structures:**
- $\mathbf{I}_r, \mathbf{I}_{gt}$: Images of size $H \times W \times C$ (typically $C=3$ for RGB), values in $[0, 1]$.
- $\mathbf{V}_M$: Visibility map of size $H \times W$, values in $[0, 1]$. This map indicates the probability or certainty that a pixel corresponds to a visible surface point.

**Formulas:**
The composite loss $L_c$ is defined as [S1]:
$$
L_c = \lambda_1 L_1(\mathbf{V}_M \odot \mathbf{I}_r, \mathbf{V}_M \odot \mathbf{I}_{gt}) + \lambda_{SSIM} L_{SSIM}(\mathbf{V}_M \odot \mathbf{I}_r, \mathbf{V}_M \odot \mathbf{I}_{gt}) + \lambda_{LPIPS} L_{LPIPS}(\mathbf{I}_r, \mathbf{I}_{gt})
$$
where $\odot$ denotes the Hadamard (element-wise) product. The total training loss $L$ combines this with regularization terms [S1]:
$$
L = L_c + \lambda_{sc} L_{sc} + \lambda_{vm} L_{vm}
$$
Here, $L_{sc}$ is the sampling coordinate regularization loss and $L_{vm}$ is the visibility map regularization loss [S1].

**Parameters:**
Typical values used in the source implementation [S1]:
- $\lambda_1 = 0.8$
- $\lambda_{SSIM} = 0.2$
- $\lambda_{LPIPS} = 0.005$
- $\lambda_{sc} = 0.001$
- $\lambda_{vm} = 0.15$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_composite_image_loss(I_r, I_gt, V_M, lambda_1=0.8, lambda_ssim=0.2, lambda_lpips=0.005):
    """
    Computes the composite image loss L_c.
    
    Args:
        I_r: Rendered image tensor (H, W, C).
        I_gt: Ground truth image tensor (H, W, C).
        V_M: Visibility map tensor (H, W).
        lambda_1: Weight for L1 loss.
        lambda_ssim: Weight for SSIM loss.
        lambda_lpips: Weight for LPIPS loss.
        
    Returns:
        L_c: Scalar tensor representing the composite loss.
    """
    # Mask the images with the visibility map
    # V_M needs to be broadcasted to (H, W, 1) for element-wise multiplication with images
    V_M_expanded = V_M.unsqueeze(-1)
    I_r_masked = I_r * V_M_expanded
    I_gt_masked = I_gt * V_M_expanded
    
    # Calculate L1 loss on masked images
    # L1 is typically mean absolute error
    l1_loss = torch.mean(torch.abs(I_r_masked - I_gt_masked))
    
    # Calculate SSIM loss on masked images
    # Note: Standard SSIM implementations often assume unmasked inputs.
    # Here, we apply the mask before computing SSIM as per the formula.
    # Implementation of SSIM requires a helper function or library.
    # We assume a function ssim_loss(img1, img2) that returns a scalar.
    ssim_loss = ssim_loss_function(I_r_masked, I_gt_masked)
    
    # Calculate LPIPS loss on unmasked images
    # LPIPS is computed on the full images to capture perceptual differences
    lpips_loss = lpips_loss_function(I_r, I_gt)
    
    # Combine losses
    L_c = (lambda_1 * l1_loss) + (lambda_ssim * ssim_loss) + (lambda_lpips * lpips_loss)
    
    return L_c
```

**Edge Cases and Pitfalls:**
- **Zero Visibility:** If $V_M$ is zero for a pixel, the L1 and SSIM terms contribute zero loss for that pixel. This prevents gradients from updating the scene representation for occluded or background regions where the rendered image might differ significantly from the ground truth (e.g., due to different camera poses or incomplete geometry) [S1].
- **LPIPS Masking:** The LPIPS term is *not* masked. This ensures that perceptual quality is maintained globally, even if specific pixels are masked out for photometric accuracy [S1].
- **Precision:** Ensure that the visibility map $V_M$ is normalized to $[0, 1]$ and matches the spatial dimensions of $\mathbf{I}_r$ and $\mathbf{I}_{gt}$.

## Subtopics

### [SRF-Loss](6978-srf-loss.md)
SRF-Loss is the composite objective function used to train SuRFNet for generating high-quality Sparse Radiance Fields (SRFs) from partial or noisy inputs. It addresses the problem of ensuring that generated 3D radiance fields not only match the input density and color statistics but also produce visually accurate renderings from novel views. It sets itself apart from simple pixel-wise reconstruction losses by incorporating a 3D-supervised density/radiance component and an online perceptual loss that explicitly optimizes for rendering quality.

## Operating characteristics
- **Time Complexity:** Dominated by the computation of SSIM and LPIPS, which are more expensive than L1. SSIM involves local window operations, and LPIPS involves deep network feature extraction. The visibility weighting adds negligible overhead (element-wise multiplication).
- **Memory Complexity:** Requires storing the visibility map $\mathbf{V}_M$ and the intermediate masked images. LPIPS requires storing feature maps from the perceptual network.
- **Strengths:** Effectively handles occlusions and partial observations by ignoring unreliable pixels in photometric terms. The inclusion of LPIPS helps maintain perceptual quality.
- **Limitations:** Relies on an accurate visibility map $\mathbf{V}_M$. If $\mathbf{V}_M$ is inaccurate, the loss may incorrectly ignore or penalize pixels. The unmasked LPIPS term may still propagate gradients from occluded regions, potentially conflicting with the masked photometric terms.
- **When to use:** When training radiance fields or 3D Gaussians from image collections with varying coverage, occlusions, or noise, where simple pixel-wise losses fail due to misalignment or missing data [S1].

## Background
The method originates from the "Gaussian in the Wild" paper, which proposes 3D Gaussian Splatting for unconstrained image collections [S1]. It was introduced to address the challenges of training 3D Gaussian Splatting on real-world, unstructured image sets where traditional NeRF-like losses are insufficient due to the lack of precise depth or visibility information. The visibility map is derived from the Gaussian splatting process itself, indicating which pixels are reliably covered by Gaussians [S1].

## Sources
[S1] Gaussian in the Wild: 3D Gaussian Splatting for Unconstrained Image Collections, https://doi.org/10.1007/978-3-031-73116-7_20, fetched 2026-10-27
