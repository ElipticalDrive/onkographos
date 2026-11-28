# AbsGS

Also called: Absolute Positional Gradients, Absgrad, Absolute Gaussian Splatting, Homodirectional view-space positional gradient

## Overview
AbsGS addresses the "over-reconstruction" artifact in 3D Gaussian Splatting, where large Gaussians in high-frequency regions fail to split, resulting in blurry rendering [S4]. It replaces the standard view-space positional gradient with a homodirectional gradient that sums the absolute values of pixel-wise sub-gradients, preventing gradient collision where positive and negative components cancel each other out [S3, unofficial][S4]. Unlike standard 3D-GS, which requires lowering the densification threshold to catch these errors (causing memory overflow), AbsGS accurately identifies over-reconstructed Gaussians using fewer primitives, achieving higher quality with roughly half the memory consumption [S4].

## How it works
**Inputs and Outputs:**
*   **Input:** The standard 3D Gaussian Splatting training state (Gaussian parameters, camera poses, loss gradients $\frac{\partial L}{\partial \mathbf{C}}$) and the rasterization metadata (pixel coverage).
*   **Output:** Updated Gaussian parameters after back-propagation, with densification decisions (split/clone) guided by the homodirectional gradient.

**Data Structures:**
The method modifies the gradient accumulation step within the backward pass. For each Gaussian $G_i$, a scalar value $\hat{g}_i$ (homodirectional gradient magnitude) is computed. This value is stored alongside the standard gradients. The parent article [Gradient Computation and Back-Propagation](10202-gradient-computation-and-back-propagation.md) defines the standard gradient tensor layout; AbsGS adds or replaces the magnitude check logic for the position parameters $\mu$.

**Parameters:**
*   **Scale threshold ($\tau_S$):** Controls the maximum allowed scale for Gaussians to be eligible for splitting. Typical values tested: $0.01$ and $0.001$ [S4]. Lowering $\tau_S$ improves quality but increases memory usage [S4].
*   **Gradient threshold ($\tau_p$):** The threshold for the homodirectional gradient to trigger a split. Since AbsGS yields larger gradient magnitudes than standard 3D-GS, $\tau_p$ can be set higher than in standard 3D-GS to avoid noise [S4]. Typical value: $0.0008$ [S4].
*   **absgrad flag:** A boolean API flag to enable the absolute gradient accumulation in the rasterization call [S3, unofficial].

**Algorithm Steps:**

1.  **Forward Pass:** Render the image using standard differentiable rasterization [18365-3d-gaussian-splatting-backward-pass.md].
2.  **Loss Computation:** Compute the loss $L$ between the rendered image and the ground truth. AbsGS may optionally use a high-pass filter on the residual before back-propagation to sharpen edges, though the core contribution is the gradient metric [S1, unofficial].
3.  **Back-Propagation:** Compute the loss gradients with respect to the Gaussian parameters. Specifically, obtain the pixel-wise sub-gradients of the loss with respect to the view-space position $\mu_{i,x}$ and $\mu_{i,y}$ for all pixels $j$ covered by Gaussian $G_i$ [S4].
4.  **Homodirectional Gradient Accumulation:** For each Gaussian $G_i$, compute the homodirectional gradient $\hat{g}_i$ by summing the absolute values of the x and y components of the sub-gradients separately [S4]:
    $$
    \hat{g}_i = \sum_{j=1}^{m} \left( \left| \frac{\partial L_j}{\partial \mu_{i,x}} \right| + \left| \frac{\partial L_j}{\partial \mu_{i,y}} \right| \right)
    $$
    where $m$ is the number of pixels covered by $G_i$ [S4]. This avoids the cancellation effect of signed summation [S3, unofficial].
5.  **Densification Decision:**
    *   **Split:** If $G_i$'s scale exceeds $\tau_S$ AND $\hat{g}_i > \tau_p$, select $G_i$ for splitting [S4].
    *   **Clone:** If $G_i$'s opacity is below a threshold, select $G_i$ for cloning. AbsGS recommends using the standard gradient (not homodirectional) for cloning decisions to follow the original 3D-GS strategy [S4].
6.  **Update:** Perform the split/clone operations and update parameters via optimizer step [S4].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_homodirectional_gradient(gaussians, pixel_gradients, coverage_mask):
    """
    Compute the homodirectional view-space positional gradient for each Gaussian.
    
    Args:
        gaussians: List of Gaussian primitives.
        pixel_gradients: Tensor of shape (num_pixels, 2) containing 
                         (dL/dmu_x, dL/dmu_y) for each pixel.
        coverage_mask: Sparse matrix or list of lists mapping each Gaussian 
                       to the indices of pixels it covers.
    
    Returns:
        homodirectional_grads: Tensor of shape (num_gaussians,) containing 
                               the sum of absolute sub-gradients for each Gaussian.
    """
    num_gaussians = len(gaussians)
    homodirectional_grads = torch.zeros(num_gaussians)
    
    for i in range(num_gaussians):
        covered_pixel_indices = coverage_mask[i]
        if len(covered_pixel_indices) == 0:
            continue
            
        # Get sub-gradients for covered pixels
        # pixel_gradients[covered_pixel_indices] has shape (m, 2)
        sub_grads = pixel_gradients[covered_pixel_indices]
        
        # Sum absolute values of x and y components separately
        abs_dx = torch.sum(torch.abs(sub_grads[:, 0]))
        abs_dy = torch.sum(torch.abs(sub_grads[:, 1]))
        
        homodirectional_grads[i] = abs_dx + abs_dy
        
    return homodirectional_grads

def densification_step(gaussians, homodirectional_grads, scale_threshold, grad_threshold):
    """
    Determine which Gaussians to split based on homodirectional gradients.
    
    Args:
        gaussians: List of Gaussian primitives with 'scale' and 'opacity' attributes.
        homodirectional_grads: Tensor of homodirectional gradient magnitudes.
        scale_threshold: Maximum scale allowed for splitting (tau_S).
        grad_threshold: Gradient threshold for splitting (tau_p).
        
    Returns:
        split_indices: List of indices of Gaussians to split.
    """
    split_indices = []
    for i, g in enumerate(gaussians):
        # Check scale constraint
        if g.scale > scale_threshold:
            # Check homodirectional gradient threshold
            if homodirectional_grads[i] > grad_threshold:
                split_indices.append(i)
    return split_indices
```

**Edge Cases and Pitfalls:**
*   **Gradient Collision:** In standard 3D-GS, large Gaussians covering many pixels often have sub-gradients with mixed signs (positive and negative) in the view space. Summing these signed values leads to cancellation, resulting in a small total gradient that fails to exceed the densification threshold [S4]. AbsGS mitigates this by using absolute values [S3, unofficial][S4].
*   **Memory Overflow:** Simply lowering the gradient threshold $\tau_p$ in standard 3D-GS to fix over-reconstruction is impractical. It selects many unnecessary Gaussians (noise), leading to significant memory consumption. On a single V100 GPU, lowering $\tau_p$ below $0.0001$ results in out-of-CUDA-memory errors [S4].
*   **Clone vs. Split:** AbsGS uses homodirectional gradients only for splitting. Cloning should follow the original 3D-GS strategy using standard gradients to maintain stability [S4].

## Operating characteristics
**Time and Memory Complexity:**
*   **Memory:** AbsGS uses roughly half the memory of standard 3D-GS for comparable quality [S4]. This is because it avoids the explosion of Gaussians caused by lowering the threshold in standard 3D-GS [S4].
*   **Time:** The additional computation for absolute value summation is negligible compared to the rasterization and back-propagation steps. The reduction in the number of Gaussians can lead to faster training convergence and less memory pressure [S1, unofficial][S4].

**Measured Performance:**
*   **Mip-NeRF360:** AbsGS-0004 ($\tau_S=0.01, \tau_p=0.0008$) achieves SSIM 0.817, PSNR 27.38, LPIPS 0.212, Memory 374MB. AbsGS-0008 ($\tau_S=0.001, \tau_p=0.0008$) achieves SSIM 0.815, PSNR 27.41, LPIPS 0.211, Memory 450MB [S4].
*   **Tanks & Temples:** AbsGS-0004 achieves SSIM 0.842, PSNR 23.64, LPIPS 0.190, Memory 159MB. AbsGS-0008 achieves SSIM 0.845, PSNR 23.54, LPIPS 0.183, Memory 202MB [S4].
*   **Deep Blending:** AbsGS-0004 achieves SSIM 0.902, PSNR 29.55, LPIPS 0.249, Memory 265MB. AbsGS-0008 achieves SSIM 0.903, PSNR 29.69, LPIPS 0.241, Memory 380MB [S4].

**Strengths and Limitations:**
*   **Strengths:** Recovers fine details and sharpens edges in high-frequency regions. Reduces over-reconstruction artifacts. More memory-efficient than threshold-lowering strategies in 3D-GS. Easy to implement as a drop-in replacement for the gradient accumulation step [S4].
*   **Limitations:** Requires careful tuning of $\tau_S$ and $\tau_p$. The absolute value operation may be less sensitive to the direction of error, potentially leading to over-splitting in some specific noise patterns if thresholds are not set correctly [S4].

## Background
AbsGS was introduced in the paper "AbsGS: Recovering Fine Details in 3D Gaussian Splatting" [S4]. It builds upon the 3D Gaussian Splatting framework [17365-3d-gaussian-splatting.md] and addresses a specific flaw in its adaptive density control strategy: gradient collision. The method was developed to improve novel view synthesis quality, particularly for scenes with fine details like vegetation and textured surfaces, where standard 3D-GS tends to produce blurry results due to large, unsplit Gaussians [S4].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S3] gsplat: An Open-Source Library for Gaussian Splatting, https://doi.org/10.48550/arXiv.2409.06765, fetched 2026-10-29 (unofficial)
[S4] AbsGS: Recovering Fine Details in 3D Gaussian Splatting, https://doi.org/10.1145/3664647.3681361, fetched 2026-10-31
