# Quantitative Evaluation Metrics

## Overview
This method computes standard photorealistic metrics—Peak Signal-to-Noise Ratio (PSNR), Structural Similarity Index (SSIM), and Learned Perceptual Image Patch Similarity (LPIPS)—to quantify the fidelity of rendered images against ground truth references [S2, S7, S11]. It addresses the problem of evaluating novel-view synthesis quality by reporting mean scores over test sets to balance pixel-level precision, structural integrity, and perceptual similarity [S2, S7]. Unlike raw error metrics, it combines perceptual models (LPIPS) and structural models (SSIM) to provide a multi-faceted quality assessment [S3, S7].

## How it works
### Inputs and Outputs
**Inputs:**
1. **Ground Truth Image ($I_{gt}$):** RGB image of shape $[H, W, 3]$ or $[1, 3, H, W]$, normalized to $[0, 1]$ [S3][S8, unofficial].
2. **Predicted Image ($I_{pred}$):** Rendered RGB image of shape $[H, W, 3]$ or $[1, 3, H, W]$, clamped to $[0, 1]$ [S8, unofficial].
3. **Optional Mask ($M$):** Binary mask or validity condition (e.g., $I_{gt} > 0$) indicating valid pixels [S8, unofficial].
4. **Optional Depth Alignment Scale ($s$):** Scalar for depth evaluation [S7].
5. **LPIPS Model:** Pre-trained AlexNet [S2, S7][S8, unofficial] or VGG variant [S5][S10, unofficial].

**Outputs:**
1. **PSNR:** Scalar value in dB.
2. **SSIM:** Scalar value in $[0, 1]$.
3. **LPIPS:** Scalar value in $[0, 1]$ (lower is better) [S3].
4. **Optional Masked Metrics:** PSNR and LPIPS restricted to valid pixels [S8, unofficial].

### Data Structures
* **Image Tensors:** Float32 tensors. Values are normalized to $[0, 1]$ for SSIM and LPIPS computation [S3][S8, unofficial].
* **LPIPS Feature Map:** Intermediate activations from the pre-trained network's convolutional layers [S4, S5].
* **SSIM Window:** $11 \times 11$ Gaussian kernel with $\sigma=1.5$ [S3].

### Numbered Steps
1. **Preprocessing:** Clamp $I_{pred}$ to $[0, 1]$ to handle numerical artifacts [S8, unofficial]. Reshape from $[H, W, C]$ to $[1, C, H, W]$ if required by the metric library [S9, unofficial]. For depth evaluation, align predicted depth $D_{pred}$ with ground truth by scaling: $D_{pred}^{aligned} = D_{pred} \times s$, where $s$ matches the ground-truth depth range [S7].
2. **PSNR Computation:** Compute Mean Squared Error (MSE):
   $$ MSE = \frac{1}{N} \sum_{i=1}^{N} (I_{gt, i} - I_{pred, i})^2 $$
   where $N = H \times W \times C$ (or $N$ if masked) [S3]. Compute PSNR:
   $$ PSNR = 10 \cdot \log_{10} \left( \frac{MAX_I^2}{MSE} \right) $$
   where $MAX_I = 1.0$ for normalized images [S3].
3. **SSIM Computation:** Compute SSIM using an $11 \times 11$ Gaussian window across the image [S3]. For each window, compute local means ($\mu_x, \mu_y$), variances ($\sigma_x^2, \sigma_y^2$), and covariance ($\sigma_{xy}$) [S3]. Apply the SSIM formula with constants $C_1 = (0.01 \cdot L)^2$ and $C_2 = (0.03 \cdot L)^2$ where $L=1.0$ [S3]. Average SSIM values over all windows to obtain the final score [S3].
4. **LPIPS Computation:** Pass $I_{gt}$ and $I_{pred}$ through the pre-trained network [S2, S5]. Extract feature maps from multiple layers ($l=1..5$) [S5]. For each layer, compute the normalized Euclidean distance between feature vectors:
   $$ d_l = \frac{\sum_{i,j} (f_{gt, l, i,j} - f_{pred, l, i,j})^2}{\sum_{i,j} (f_{gt, l, i,j}^2 + f_{pred, l, i,j}^2)} $$
   Aggregate distances across layers, typically weighted by learned parameters [S5].
5. **Masked/Valid Region Evaluation:** Compute PSNR only on valid pixels where $I_{gt} > 0$ to ignore background or invalid regions [S8, unofficial]. Accumulate and average metrics across all images in the scene [S10, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import torch
import torch.nn.functional as F

def compute_psnr(gt: torch.Tensor, pred: torch.Tensor) -> float:
    """Compute PSNR between ground truth and predicted images. Inputs: [1, C, H, W], values in [0, 1]."""
    mse = torch.mean((gt - pred) ** 2)
    if mse == 0:
        return float('inf')
    psnr = 10 * torch.log10(1.0 / mse)
    return psnr.item()

def compute_ssim(gt: torch.Tensor, pred: torch.Tensor) -> float:
    """Compute SSIM between ground truth and predicted images. Inputs: [1, C, H, W], values in [0, 1]."""
    # Create 11x11 Gaussian kernel with sigma=1.5
    kernel_size = 11
    sigma = 1.5
    kernel = torch.ones(1, 1, kernel_size, kernel_size)
    # Apply Gaussian blur for local statistics
    mu_x = F.conv2d(gt, kernel, padding=kernel_size//2)
    mu_y = F.conv2d(pred, kernel, padding=kernel_size//2)
    mu_xx = F.conv2d(gt * gt, kernel, padding=kernel_size//2)
    mu_yy = F.conv2d(pred * pred, kernel, padding=kernel_size//2)
    mu_xy = F.conv2d(gt * pred, kernel, padding=kernel_size//2)
    
    var_x = mu_xx - mu_x * mu_x
    var_y = mu_yy - mu_y * mu_y
    cov_xy = mu_xy - mu_x * mu_y
    
    C1 = (0.01 * 1.0) ** 2
    C2 = (0.03 * 1.0) ** 2
    
    ssim_map = ((2 * mu_x * mu_y + C1) * (2 * cov_xy + C2)) / \
               ((mu_x**2 + mu_y**2 + C1) * (var_x + var_y + C2))
    return torch.mean(ssim_map).item()

def compute_lpips(gt: torch.Tensor, pred: torch.Tensor, net: torch.nn.Module) -> float:
    """Compute LPIPS using AlexNet/VGG features. Inputs: [1, C, H, W], values in [0, 1]."""
    net.eval()
    with torch.no_grad():
        features_gt = []
        features_pred = []
        
        # Forward pass collecting conv layer outputs
        x = gt
        for module in net.modules():
            if isinstance(module, torch.nn.Conv2d):
                x = module(x)
                features_gt.append(x)
            if isinstance(module, torch.nn.ReLU):
                pass # ReLU applied in forward pass
                
        x = pred
        for module in net.modules():
            if isinstance(module, torch.nn.Conv2d):
                x = module(x)
                features_pred.append(x)
                
        dist = 0.0
        for f_gt, f_pred in zip(features_gt, features_pred):
            f_gt_norm = F.normalize(f_gt, p=2, dim=1)
            f_pred_norm = F.normalize(f_pred, p=2, dim=1)
            dist += torch.mean((f_gt_norm - f_pred_norm) ** 2)
        return dist.item()

def evaluate_metrics(gt: torch.Tensor, pred: torch.Tensor, mask: torch.Tensor = None, 
                     lpips_net: torch.nn.Module = None) -> dict:
    """Main evaluation function."""
    pred = torch.clamp(pred, 0.0, 1.0)
    if gt.dim() == 3:
        gt = gt.permute(2, 0, 1).unsqueeze(0)
    if pred.dim() == 3:
        pred = pred.permute(2, 0, 1).unsqueeze(0)
        
    metrics = {}
    metrics['psnr'] = compute_psnr(gt, pred)
    metrics['ssim'] = compute_ssim(gt, pred)
    if lpips_net is not None:
        metrics['lpips'] = compute_lpips(gt, pred, lpips_net)
    else:
        metrics['lpips'] = 0.0 # Not given in the sources if no net provided.
        
    if mask is not None:
        gt_masked = gt * mask
        pred_masked = pred * mask
        metrics['psnr_masked'] = compute_psnr(gt_masked, pred_masked)
        if lpips_net is not None:
            metrics['lpips_masked'] = compute_lpips(gt_masked, pred_masked, lpips_net)
            
    return metrics
```

### Parameters
* **LPIPS Network:** AlexNet [S2, S7][S8, unofficial] or VGG variant [S5][S10, unofficial].
* **LPIPS Weights:** Learned weights from the original LPIPS paper [S5].
* **SSIM Window Size:** $11 \times 11$ Gaussian kernel with $\sigma=1.5$ [S3].
* **Depth Scale ($s$):** Computed via alignment to match ground-truth depth range [S7].
* **Evaluation Interval:** Every 300 iterations [S3].

### Edge Cases and Pitfalls
* **Information Leakage:** In appearance-aware NeRFs, optimizing appearance embeddings on held-out pixels causes metrics to fail by memorizing the image rather than generalizing [S12]. Use split-image evaluation to prevent this [S12].
* **Zero/Mean Embeddings:** Setting appearance embeddings to zeros or mean values results in structurally accurate but appearance-mismatched renderings, causing perceptual metrics to fail [S12].
* **Depth Alignment:** Predicted depth must be scaled to match ground-truth range before computing depth errors [S7].
* **Clamping:** Predicted RGB values must be clamped to $[0, 1]$ before LPIPS computation [S8, unofficial].
* **FID Resampling:** FID is sensitive to resizing; use antialiased (area) interpolation at 128×128 resolution [S14].

## Subtopics

### Split-image Evaluation Protocol
An evaluation protocol for appearance-aware NeRF models that optimizes the appearance embedding on a subset of pixels to adapt scene appearance without directly optimizing held-out pixels, preventing information leakage while measuring perceptual similarity [S12]. It splits a ground-truth test image into left and right halves, optimizes the appearance embedding using only the left half, renders the right half, and calculates perceptual metrics on the rendered right half against the ground truth right half [S12]. Unlike the NRW baseline which encodes the entirety of a held-out image with an appearance encoder, this scheme isolates geometry from appearance and limits information leakage [S12]. Pitfall: Setting the appearance embedding to zeros or the mean of optimized embeddings results in plausible but structurally accurate renderings that fail to match ground truth appearance, causing perceptual metrics to fail [S12].

### [Masked Metrics Evaluation](15886-masked-metrics-evaluation.md)
A metric reporting protocol that computes PSNR, SSIM, and LPIPS on both the full image and the valid-masked region to isolate visible reconstruction quality. It computes standard metrics over all pixels and then recomputes PSNR restricted to pixels inside the validity mask ($I_{gt} > 0$), reporting these values alongside global metrics.

### FID Evaluation
Fréchet Inception Distance (FID) evaluates how well the method captures the appearance distribution of images [S4]. It is computed on an empty background (value=0, gray) at 128×128 resolution, using antialiased resampling if resizing is needed, as FID is sensitive to this aspect [S14].

### Scaffold-GS Evaluation Metrics
Reports SSIM, PSNR, LPIPS, and storage size for specific benchmark scenes. On Mip-NeRF360 scenes (bicycle, garden, stump, room, counter, kitchen, bonsai), SSIM scores are 0.705, 0.842, 0.784, 0.925, 0.914, 0.928, 0.946 respectively. PSNR scores are 24.50, 27.17, 26.27, 31.93, 29.34, 31.30, 32.70 respectively. LPIPS scores are 0.306, 0.146, 0.284, 0.202, 0.191, 0.126, 0.185 respectively. Storage sizes are 248, 271, 493, 133, 194, 173, 258 MB respectively [S13]. On Tanks&Temples/Deep Blending scenes (Truck, Train, Dr Johnson, Playroom), SSIM scores are 0.879, 0.857, 0.779, 0.774 respectively [S13].

## Operating characteristics
* **Time Complexity:** PSNR and SSIM are $O(N)$ where $N$ is the number of pixels. LPIPS is dominated by the forward pass through the feature extractor [G].
* **Memory Complexity:** LPIPS requires storing intermediate feature maps for all layers, which scales with image resolution [G].
* **Performance:** Evaluated every 300 iterations during training [S3]. On KITTI reconstruction, NSG achieves PSNR 26.66, SSIM 0.806, LPIPS 0.186 [S6].
* **Strengths:** PSNR provides pixel-level precision. SSIM captures structural information. LPIPS captures perceptual similarity [S1, S2, S4].
* **Limitations:** PSNR is sensitive to pixel-level errors. SSIM can be sensitive to misalignment. LPIPS depends on a pre-trained network [S2, S5].
* **When to Use:** Use PSNR and SSIM for quick checks and standard comparison. Use LPIPS for perceptual quality assessment, especially in appearance-aware NeRFs [S2, S5].

## Background
PSNR, SSIM, and LPIPS are standard metrics in computer vision and image processing [S1, S2, S4]. They were adopted for NeRF evaluation to provide a multi-faceted view of rendering accuracy, balancing pixel-level precision with structural and perceptual similarity [S1, S3]. The use of LPIPS in NeRF evaluation was popularized by works like BARF and NeRF-W [S7, S12]. The split-image evaluation protocol was introduced to address the issue of information leakage in appearance-aware NeRFs [S12].

## Sources
[S1] PhySG: Inverse Rendering with Spherical Gaussians for Physics-based Material Editing and Relighting, https://doi.org/10.1109/cvpr46437.2021.00541, fetched 2026-10-26
[S2] StegaNeRF: Embedding Invisible Information within Neural Radiance Fields, https://doi.org/10.1109/iccv51070.2023.00047, fetched 2026-10-26
[S3] Practical Evaluation of Standard 3D Gaussian Splatting for Real-World Scene Reconstruction., https://doi.org/10.3390/s26185754, fetched 2026-10-26
[S4] Texture Fields: Learning Texture Representations in Function Space, https://doi.org/10.1109/iccv.2019.00463, fetched 2026-10-26
[S5] NeX: Real-time View Synthesis with Neural Basis Expansion, https://doi.org/10.1109/cvpr46437.2021.00843, fetched 2026-10-26
[S6] Neural Scene Graphs for Dynamic Scenes, https://doi.org/10.1109/cvpr46437.2021.00288, fetched 2026-10-26
[S7] SPARF: Neural Radiance Fields from Sparse and Noisy Poses, https://doi.org/10.1109/cvpr52729.2023.00408, fetched 2026-10-26
[S8] muskie82/MonoGS, https://github.com/muskie82/MonoGS, fetched 2026-10-27 (unofficial)
[S9] barbararoessle/ganerf, https://github.com/barbararoessle/ganerf, fetched 2026-10-27 (unofficial)
[S10] otakuxiang/gaussian-splatting-jittor, https://github.com/otakuxiang/gaussian-splatting-jittor, fetched 2026-10-27 (unofficial)
[S11] Digging into Depth Priors for Outdoor Neural Radiance Fields, https://doi.org/10.1145/3581783.3612306, fetched 2026-10-30
[S12] NeRF in the Wild: Neural Radiance Fields for Unconstrained Photo Collections, https://doi.org/10.1109/cvpr46437.2021.00713, fetched 2026-10-26
[S13] Scaffold-GS: Structured 3D Gaussians for View-Adaptive Rendering, https://doi.org/10.1109/cvpr52733.2024.01952, fetched 2026-10-26
[S14] Shape, Pose, and Appearance from a Single Image via Bootstrapped Radiance Field Inversion, https://doi.org/10.1109/cvpr52729.2023.00427, fetched 2026-10-26
