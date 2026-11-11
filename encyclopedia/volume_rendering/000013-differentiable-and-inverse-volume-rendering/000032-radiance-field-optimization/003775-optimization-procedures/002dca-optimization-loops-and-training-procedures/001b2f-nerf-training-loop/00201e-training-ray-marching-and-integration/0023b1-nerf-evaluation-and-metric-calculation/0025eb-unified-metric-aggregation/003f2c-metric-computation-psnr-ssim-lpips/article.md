# Metric Computation (PSNR, SSIM, LPIPS)
Also called: Image quality metrics, Novel-View Rendering Metrics, PSNR, SSIM, LPIPS Computation, Photorealistic Metrics, Rendering Quality Metrics

## Overview
This method computes standard photorealistic evaluation metrics—Peak Signal-to-Noise Ratio (PSNR), Structural Similarity Index (SSIM), and Learned Perceptual Image Patch Similarity (LPIPS)—to quantify the fidelity of synthesized novel views against ground-truth images [S1, S3]. It differs from depth-specific metrics by focusing exclusively on RGB pixel-space and perceptual feature-space comparisons, serving as the final quantitative step in the NeRF evaluation pipeline after ray marching and integration [S1, S3].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Predicted RGB image tensor $I_{pred}$ and Ground Truth RGB image tensor $I_{gt}$. Both are typically in range $[0, 1]$ or $[0, 255]$. For LPIPS, the network version (AlexNet) is required [S1].
*   **Outputs:** Scalar values for PSNR, SSIM, and LPIPS.

**Data Structures:**
*   Images are stored as tensors with shape $[1, C, H, W]$ (batch size 1, channels, height, width) for metric computation [S2, unofficial].
*   If the input is $[H, W, C]$, it must be permuted to $[1, C, H, W]$ [S2, unofficial].

**Steps:**
1.  **Preprocessing:** Ensure $I_{pred}$ and $I_{gt}$ are in the same coordinate system and range. If $I_{pred}$ is not clamped, clamp it to $[0, 1]$ before LPIPS computation [S2, unofficial].
2.  **PSNR Calculation:** Compute the Mean Squared Error (MSE) between $I_{pred}$ and $I_{gt}$, then convert to decibels.
3.  **SSIM Calculation:** Compute the Structural Similarity Index using a sliding window approach over the image pairs.
4.  **LPIPS Calculation:** Pass $I_{pred}$ and $I_{gt}$ through the AlexNet-based LPIPS network to extract feature maps, then compute the learned perceptual distance [S1].

**Formulas:**
*   **MSE:**
    $$ \text{MSE} = \frac{1}{N} \sum_{i=1}^{N} (I_{pred}(i) - I_{gt}(i))^2 $$
    where $N = H \times W \times C$.
*   **PSNR:**
    $$ \text{PSNR} = 10 \cdot \log_{10} \left( \frac{\text{MAX}_I^2}{\text{MSE}} \right) $$
    where $\text{MAX}_I$ is the maximum possible image value (1.0 for $[0,1]$ range, 255 for $[0,255]$ range).
*   **SSIM:** Computed via the standard SSIM index formula [S1, S3].
*   **LPIPS:** Computed as the weighted $L_2$ distance between normalized feature activations from the AlexNet network [S1, S3].

**Parameters:**
*   **LPIPS Network:** AlexNet version [S1].
*   **Image Range:** Typically $[0, 1]$ for neural network outputs, but must match the PSNR $\text{MAX}_I$ definition [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Range Mismatch:** If $I_{pred}$ is not clamped to $[0, 1]$ before LPIPS, results may be invalid [S2, unofficial].
*   **Tensor Shape:** Metrics libraries often expect $[N, C, H, W]$. Passing $[H, W, C]$ directly will cause errors or incorrect computation [S2, unofficial].
*   **Background:** Ground truth images may require blending with a background color if they contain alpha channels, though standard novel-view synthesis metrics often assume opaque backgrounds or pre-blended images [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F
from lpips import LPIPS # Assuming standard lpips library implementation

def compute_metrics(I_pred: torch.Tensor, I_gt: torch.Tensor, max_val: float = 1.0) -> dict:
    """
    Compute PSNR, SSIM, and LPIPS.
    Inputs: I_pred, I_gt in shape [1, C, H, W] or [H, W, C].
    """
    # Ensure consistent shape [1, C, H, W]
    if I_pred.dim() == 3:
        I_pred = I_pred.permute(2, 0, 1).unsqueeze(0)
        I_gt = I_gt.permute(2, 0, 1).unsqueeze(0)
    
    # Clamp predictions for LPIPS [S2, unofficial]
    I_pred_clamped = torch.clamp(I_pred, 0.0, 1.0)
    
    # 1. PSNR
    mse = F.mse_loss(I_pred, I_gt)
    psnr = 10 * torch.log10((max_val ** 2) / mse)
    
    # 2. SSIM (using a standard implementation, e.g., from skimage or pytorch_msssim)
    # Note: SSIM formula involves local windows, mean, var, and covariance [S1, S3]
    ssim = compute_ssim(I_pred, I_gt) # [G] Standard SSIM implementation
    
    # 3. LPIPS
    # Uses AlexNet network version [S1]
    loss_fn = LPIPS(net='alex')
    lpips_val = loss_fn(I_pred_clamped, I_gt).item()
    
    return {
        "psnr": psnr.item(),
        "ssim": ssim.item(),
        "lpips": lpips_val
    }

def compute_ssim(I_pred: torch.Tensor, I_gt: torch.Tensor) -> float:
    """
    Placeholder for standard SSIM computation.
    Computes structural similarity using sliding windows [S1, S3].
    """
    # [G] Implementation details for SSIM involve calculating local means, variances,
    # and covariances between I_pred and I_gt, then applying the SSIM formula.
    # This is a standard computer vision operation.
    pass
```

## Operating characteristics
*   **Time Complexity:** PSNR and SSIM are $O(N)$ where $N$ is the number of pixels. LPIPS is $O(N \cdot C \cdot K^2)$ where $K$ is the kernel size of the AlexNet convolutions, dominated by the forward pass of the CNN.
*   **Memory Complexity:** PSNR/SSIM are $O(1)$ auxiliary memory. LPIPS requires storing intermediate feature maps from the AlexNet network, which is significant but constant per image resolution.
*   **Strengths:** PSNR and SSIM are fast and standard. LPIPS correlates better with human perception than PSNR/SSIM [S1, S3].
*   **Limitations:** LPIPS is computationally expensive and requires a pre-trained network. PSNR is sensitive to pixel-wise errors and does not correlate well with perceptual quality [S1, S3].
*   **When to use:** Use for final evaluation of novel-view synthesis quality. Do not use for training loss if perceptual quality is the primary goal (use LPIPS or SSIM as loss instead).

## Background
PSNR and SSIM are standard image quality metrics in computer vision [S1, S3]. LPIPS was introduced to address the poor correlation of PSNR/SSIM with human perception [S1, S3]. In the context of NeRF and radiance fields, these metrics are used to evaluate the quality of rendered novel views [S1, S3]. The AlexNet version of LPIPS is commonly used in this domain [S1].

## Sources
[S1] SPARF: Neural Radiance Fields from Sparse and Noisy Poses, https://doi.org/10.1109/cvpr52729.2023.00408, fetched 2026-10-26
[S2] barbararoessle/ganerf, https://github.com/barbararoessle/ganerf, fetched 2026-10-27 (unofficial)
[S3] Digging into Depth Priors for Outdoor Neural Radiance Fields, https://doi.org/10.1145/3581783.3612306, fetched 2026-10-30
