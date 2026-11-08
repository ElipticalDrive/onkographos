# Metric Computation (PSNR, SSIM, LPIPS)
Also called: Image quality metrics, PSNR, SSIM, LPIPS Computation, Photorealistic Metrics, Rendering Quality Metrics

## Overview
Computes evaluation metrics (PSNR, SSIM, LPIPS) between the ground truth and predicted images [S1, unofficial]. These metrics compare synthesized views at testing viewpoints with the ground truth [S2, unofficial]. Unlike [Unified Metric Aggregation](3644-unified-metric-aggregation.md) which averages scores, this method calculates the raw per-pixel or per-tensor values for each image pair.

## How it works
**Inputs:**
- `gt_rgb`: Ground truth RGB image tensor, shape `[H, W, 3]`, values in `[0, 1]` or `[0, 255]` depending on implementation context.
- `pred_rgb`: Predicted RGB image tensor, shape `[H, W, 3]`, values in `[0, 1]`.

**Outputs:**
- `psnr`: Scalar float.
- `ssim`: Scalar float.
- `lpips`: Scalar float.

**Data Structures:**
- Images are treated as 4D tensors `[1, C, H, W]` for metric library compatibility [S1, unofficial].

**Steps:**
1. **Background Blending:** If the ground truth image has an alpha channel or requires background blending, blend the ground truth RGB with the background. [S1, unofficial]
2. **Reshape:** Move RGB images from `[H, W, C]` to `[1, C, H, W]` format for metric computation [S1, unofficial].
3. **Clamp:** Clamp predicted values to `[0, 1]` before LPIPS computation [S1, unofficial].
4. **Compute PSNR:** Calculate Peak Signal-to-Noise Ratio between the reshaped ground truth and predicted images.
5. **Compute SSIM:** Calculate Structural Similarity Index between the reshaped ground truth and predicted images.
6. **Compute LPIPS:** Calculate Learned Perceptual Image Patch Similarity between the reshaped ground truth and predicted images.

**Formulas:**
- **PSNR:** $PSNR = 10 \cdot \log_{10}\left(\frac{MAX_I^2}{MSE}\right)$, where $MAX_I$ is the maximum possible pixel value (1.0 for normalized, 255 for 8-bit) and $MSE$ is the Mean Squared Error between the two images [S2, unofficial].
- **SSIM:** Calculated using the standard SSIM index formula involving luminance, contrast, and structure comparisons between local windows of the images [S2, unofficial].
- **LPIPS:** Calculated using a learned perceptual metric based on deep network features [S2, unofficial].

**Parameters:**
- `MAX_I`: 1.0 for normalized images, 255 for 8-bit images.
- `ssim_kernel_size`: Typically 11x11 or 11x11 Gaussian kernel for SSIM calculation.
- `lpips_net`: VGG or AlexNet backbone for LPIPS calculation.

**Edge Cases:**
- **Precision:** Ensure consistent data types (float32) for metric calculations to avoid precision loss.
- **Boundaries:** Handle boundary conditions in SSIM calculation (e.g., padding).
- **Degenerate Input:** If images are identical, PSNR should be infinity (or max float), SSIM 1.0, LPIPS 0.0.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F
from lpips import LPIPS
from skimage.metrics import structural_similarity as ssim_fn
import math

def compute_metrics(gt_rgb: torch.Tensor, pred_rgb: torch.Tensor) -> dict:
    """
    Compute PSNR, SSIM, and LPIPS between ground truth and predicted images.
    
    Args:
        gt_rgb: Ground truth RGB image, shape [H, W, 3], values in [0, 1]
        pred_rgb: Predicted RGB image, shape [H, W, 3], values in [0, 1]
        
    Returns:
        Dictionary with 'psnr', 'ssim', 'lpips' keys
    """
    # Step 1: Background blending (if necessary)
    # [G] Assuming gt_rgb is already blended or has no alpha channel.
    # If alpha exists, blend with white background:
    # alpha = gt_rgb[:, :, 3:4]
    # gt_rgb = gt_rgb[:, :, :3] * alpha + 1.0 * (1.0 - alpha)
    
    # Step 2: Reshape to [1, C, H, W]
    # gt_rgb: [H, W, 3] -> [1, 3, H, W]
    gt_4d = gt_rgb.permute(2, 0, 1).unsqueeze(0)
    pred_4d = pred_rgb.permute(2, 0, 1).unsqueeze(0)
    
    # Step 3: Clamp predicted values to [0, 1] for LPIPS
    pred_clamped = torch.clamp(pred_4d, 0.0, 1.0)
    
    # Step 4: Compute PSNR
    mse = F.mse_loss(gt_4d, pred_4d)
    if mse == 0:
        psnr = float('inf')
    else:
        psnr = 10 * math.log10(1.0 / mse)  # Assuming MAX_I = 1.0 for normalized images
    
    # Step 5: Compute SSIM
    # [G] Using skimage's ssim function which expects [H, W, C] or [C, H, W]
    # Convert back to [H, W, C] for skimage
    gt_np = gt_4d.squeeze(0).permute(1, 2, 0).cpu().numpy()
    pred_np = pred_4d.squeeze(0).permute(1, 2, 0).cpu().numpy()
    ssim_val = ssim_fn(gt_np, pred_np, channel_axis=2, data_range=1.0)
    
    # Step 6: Compute LPIPS
    # [G] Initialize LPIPS model once outside this function in practice
    # Assuming lpips_model is a pre-initialized LPIPS network
    # lpips_val = lpips_model(pred_clamped, gt_4d).item()
    # For self-contained pseudocode, we note the call
    lpips_val = 0.0  # Placeholder for actual LPIPS computation
    
    return {
        'psnr': psnr,
        'ssim': ssim_val,
        'lpips': lpips_val
    }
```

## Subtopics

### [LPIPS Evaluation Protocol](4376-lpips-evaluation-protocol.md)
Defines the specific protocol for LPIPS evaluation, including network backbone and normalization. Choose when detailed perceptual metrics are required.

## Operating characteristics
- **Time Complexity:** PSNR and SSIM are $O(H \cdot W)$. LPIPS is $O(H \cdot W \cdot C)$ but involves deep network inference, making it significantly slower.
- **Memory Complexity:** $O(H \cdot W \cdot C)$ for storing intermediate tensors.
- **Measured Performance:** Not given in the sources.
- **Strengths:** PSNR and SSIM are fast and widely understood. LPIPS correlates better with human perception.
- **Limitations:** LPIPS is computationally expensive. PSNR can be misleading for perceptual quality.
- **When to use:** Use PSNR/SSIM for quick checks and LPIPS for final evaluation.

## Background
Originated in the context of novel view synthesis and neural radiance fields (NeRF) evaluation [S2, unofficial]. Commonly used in computer vision for comparing synthesized views with ground truth [S2, unofficial].

## Sources
[S1] barbararoessle/ganerf, https://github.com/barbararoessle/ganerf, fetched 2026-10-27 (unofficial)
[S2] Digging into Depth Priors for Outdoor Neural Radiance Fields, https://doi.org/10.1145/3581783.3612306, fetched 2026-10-30 (unofficial)
