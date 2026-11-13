# Photometric Loss


## Overview
Photometric loss minimizes the difference between rendered pixels and input training images during joint optimization [Parent](7799-photometric-reconstruction-loss.md). It typically combines an $\ell_1$ reconstruction term with a differentiable Structural Similarity (D-SSIM) term to balance pixel-wise accuracy and perceptual structure [S1].

## How it works
**Inputs and Outputs**
*   **Inputs:** Predicted color $\hat{C}(r_i)$ and ground-truth color $C(r_i)$ for sample $i$ along ray $r_i$; hyperparameters $\lambda$ (D-SSIM weight), $\lambda_{\text{dist}}$, $\lambda_{\text{normal}}$.
*   **Outputs:** Scalar loss value $\mathcal{L}_{\text{photo}}$ (and optionally total loss $\mathcal{L}$).

**Data Structures**
*   Colors are 3-channel vectors (RGB).
*   D-SSIM is computed over local windows (typically $3\times3$ or $11\times11$ pixels) in the 2D image plane.

**Steps**
1.  Compute the $\ell_1$ loss between predicted and ground-truth colors:
    $$ \mathcal{L}_1 = \frac{1}{N} \sum_{i=1}^N \| \hat{C}(r_i) - C(r_i) \|_1 $$
2.  Compute the differentiable SSIM loss $\mathcal{L}_{\text{D-SSIM}}$ between the rendered and ground-truth images. This involves calculating local means, variances, and covariance within a sliding window, using a small constant $C_1, C_2$ for stability [S1].
3.  Combine them:
    $$ \mathcal{L}_{\text{photo}} = (1-\lambda)\mathcal{L}_1 + \lambda \mathcal{L}_{\text{D-SSIM}} $$
4.  (Optional) If optimizing primitives with depth and normals, add distortion and normal consistency losses to form the total loss:
    $$ \mathcal{L} = \mathcal{L}_{\text{photo}} + \lambda_{\text{dist}}\mathcal{L}_{\text{dist}} + \lambda_{\text{normal}}\mathcal{L}_{\text{normal}} $$
    where $\mathcal{L}_{\text{dist}}$ penalizes spread in rendered depth distributions and $\mathcal{L}_{\text{normal}}$ aligns primitive normals with depth-derived surface normals [S1].

**Parameters**
*   $\lambda$: Weight for D-SSIM. Typical values range from 0.2 to 0.8 depending on the balance between sharpness and color accuracy [S1].
*   $\lambda_{\text{dist}}, \lambda_{\text{normal}}$: Weights for auxiliary losses. Values are scene-dependent and often tuned empirically [S1].

**Edge Cases and Pitfalls**
*   **Precision:** D-SSIM requires careful handling of numerical stability in variance calculations (add small epsilon).
*   **Boundaries:** Window-based SSIM requires padding or boundary handling for edge pixels.
*   **Degenerate Input:** If ground-truth images are missing or corrupted, $\mathcal{L}_1$ may diverge; clamping or masking is required [G].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_photometric_loss(rendered_image, gt_image, lambda_dssim=0.2):
    """
    Compute photometric loss combining L1 and D-SSIM.
    
    Args:
        rendered_image: Tensor of shape (H, W, 3), predicted colors.
        gt_image: Tensor of shape (H, W, 3), ground-truth colors.
        lambda_dssim: Weight for D-SSIM term.
        
    Returns:
        Scalar loss value.
    """
    # L1 Loss
    l1_loss = torch.mean(torch.abs(rendered_image - gt_image))
    
    # D-SSIM Loss
    # Note: D-SSIM is typically computed on luminance or per-channel.
    # Here we assume a standard differentiable SSIM implementation.
    dssim_loss = compute_dssim(rendered_image, gt_image)
    
    # Combined Loss
    photo_loss = (1 - lambda_dssim) * l1_loss + lambda_dssim * dssim_loss
    
    return photo_loss

def compute_dssim(img1, img2, window_size=11, C1=1e-4, C2=1e-4):
    """
    Compute differentiable SSIM loss.
    """
    # Convert to float if necessary
    img1 = img1.float()
    img2 = img2.float()
    
    # Create a 2D Gaussian kernel for local windowing
    kernel = create_gaussian_kernel(window_size)
    
    # Compute local means
    mu1 = conv2d(img1, kernel)
    mu2 = conv2d(img2, kernel)
    
    # Compute local variances and covariance
    sigma1_sq = conv2d(img1**2, kernel) - mu1**2
    sigma2_sq = conv2d(img2**2, kernel) - mu2**2
    sigma12 = conv2d(img1 * img2, kernel) - mu1 * mu2
    
    # SSIM formula
    ssim_map = ((2 * mu1 * mu2 + C1) * (2 * sigma12 + C2)) / \
               ((mu1**2 + mu2**2 + C1) * (sigma1_sq + sigma2_sq + C2))
    
    # Return mean SSIM as loss (1 - SSIM)
    return torch.mean(1 - ssim_map)

def create_gaussian_kernel(window_size, sigma=1.5):
    """Create a 2D Gaussian kernel."""
    coords = torch.arange(window_size).float() - window_size // 2
    kernel_1d = torch.exp(-coords**2 / (2 * sigma**2))
    kernel_2d = torch.outer(kernel_1d, kernel_1d)
    return kernel_2d / kernel_2d.sum()

def conv2d(image, kernel):
    """Apply 2D convolution."""
    # Assuming image is (H, W, 3) and kernel is (K, K)
    # Pad image
    pad = kernel.shape[0] // 2
    padded = torch.nn.functional.pad(image, (pad, pad, pad, pad, 0, 0))
    
    # Convolve
    # Note: In practice, use efficient convolution layers
    # Here we simulate with a simple loop for clarity, though inefficient
    H, W, C = image.shape
    K = kernel.shape[0]
    result = torch.zeros_like(image)
    
    for h in range(H):
        for w in range(W):
            for c in range(C):
                sum_val = 0.0
                for kh in range(K):
                    for kw in range(K):
                        sum_val += padded[h+kh, w+kw, c] * kernel[kh, kw]
                result[h, w, c] = sum_val
    return result
```

## Subtopics

### Weighted Photometric Loss
A variant that re-scales the loss with a weight $w_i$ to focus on hard samples for faster convergence [S2]. The weight is calculated as the L2 norm of the color difference divided by the minimum L2 norm across all samples, clamped to $[1, 10]$ [S2]. The loss is defined as $L_p = \frac{1}{N} \sum_{i=1}^N \langle w_i \rangle \| \hat{C}(r_i) - C(r_i) \|_2^2$, where $\langle \cdot \rangle$ denotes the stop-gradient operation [S2]. It is often combined with an L1 regularizer on view-dependent color: $L = L_p + \lambda L_r$ [S2].

## Operating characteristics
*   **Time Complexity:** Linear in the number of pixels and samples per pixel. D-SSIM adds a constant factor related to window size.
*   **Memory Complexity:** Linear in image resolution for storing intermediate SSIM maps.
*   **Strengths:** Robust to noise; D-SSIM preserves structural details better than pure L1.
*   **Limitations:** Computationally heavier than L1 alone; sensitive to hyperparameter $\lambda$.
*   **When to use:** Standard for novel view synthesis when perceptual quality is prioritized.
*   **When not to use:** Real-time applications where speed is critical; scenes with significant lighting changes where photometric consistency is low.

## Background
Photometric loss is a standard component in differentiable rendering and neural radiance field (NeRF) optimization. The combination of L1 and D-SSIM was popularized in methods like Mip-NeRF and 2D Gaussian Splatting [S1, S2]. The weighted variant was introduced in Lightning NeRF to address convergence speed in autonomous driving scenes [S2].

## Sources
[S1] Fourier Splatting: Generalized Fourier encoded primitives for scalable radiance fields, https://doi.org/10.48550/arXiv.2603.19834, fetched 2026-10-30
[S2] Lightning NeRF: Efficient Hybrid Scene Representation for Autonomous Driving, https://doi.org/10.1109/icra57147.2024.10611130, fetched 2026-10-27
