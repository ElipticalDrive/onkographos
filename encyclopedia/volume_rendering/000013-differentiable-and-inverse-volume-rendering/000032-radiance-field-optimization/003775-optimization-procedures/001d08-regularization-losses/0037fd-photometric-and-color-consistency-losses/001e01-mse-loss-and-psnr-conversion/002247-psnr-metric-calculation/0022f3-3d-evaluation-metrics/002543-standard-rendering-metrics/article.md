# Standard Rendering Metrics

Also called: PSNR (Peak Signal-to-Noise Ratio), PSNR Calculation, PSNR computation, Peak Signal-to-Noise Ratio

## Overview
Standard Rendering Metrics provide an objective, numerical assessment of image fidelity for 3D reconstructions and novel view synthesis [S1, unofficial]. The primary metric is the Peak Signal-to-Noise Ratio (PSNR), which quantifies pixel-wise reconstruction error in decibels by comparing rendered images against ground-truth references [S5, unofficial]. While PSNR is the standard baseline due to its computational cheapness, it is widely recognized as insensitive to perceptual quality, often necessitating companion metrics like SSIM or LPIPS for a complete evaluation [S6, unofficial].

## How it works
The calculation of PSNR involves computing the Mean Squared Error (MSE) between the predicted (rendered) image and the target (ground-truth) image, followed by a logarithmic transformation.

**Inputs and Outputs:**
*   **Input:** Rendered image tensor $\hat{I}$ and Ground-truth image tensor $I$. Both must be of identical dimensions $H \times W \times C$ (typically $C=3$ for RGB).
*   **Output:** A scalar float representing the PSNR in decibels (dB).

**Data Structures:**
*   Images are typically stored as floating-point tensors. If the source data is integer-based (e.g., 8-bit), it must be normalized to a range where the peak signal is defined (commonly $[0, 1]$ or $[0, 255]$).

**Steps:**
1.  **Normalization:** Ensure both $\hat{I}$ and $I$ are normalized to the same range. If using the formula with $MAX_I=1.0$, values should be in $[0, 1]$. If using $MAX_I=255$, values should be in $[0, 255]$ [S4, S7, unofficial].
2.  **MSE Calculation:** Compute the Mean Squared Error between the two images.
    $$ MSE = \frac{1}{N} \sum_{k=1}^{N} (\hat{I}_k - I_k)^2 $$
    Where $N$ is the total number of pixels ($H \times W \times C$) and $\hat{I}_k, I_k$ are the pixel values at index $k$ [S2, S5, unofficial].
3.  **Clamping:** Clamp the MSE to a minimum value (e.g., $10^{-12}$) to prevent numerical instability (logarithm of zero or negative numbers) [S5, unofficial].
4.  **PSNR Calculation:** Compute the PSNR using the clamped MSE.
    $$ PSNR = -10 \log_{10}(\max(MSE, \epsilon)) $$
    Alternatively, using the peak intensity $MAX_I$:
    $$ PSNR = 10 \log_{10} \left( \frac{(MAX_I)^2}{MSE} \right) $$
    Where $\epsilon$ is the stability floor (e.g., $10^{-12}$) [S2, S3, S5, unofficial].

**Parameters:**
*   **$\epsilon$ (MSE clamp min):** $10^{-12}$ is a common value for numerical stability [S5, unofficial].
*   **$MAX_I$:** The maximum possible intensity value. For normalized $[0,1]$ data, this is $1.0$. For 8-bit integer data, this is $255$ [S4, S7, unofficial].

**Edge Cases and Pitfalls:**
*   **Non-finite Values:** If the MSE is zero (identical images) or negative (due to numerical error), the logarithm is undefined. Clamping prevents $\log(0)$, resulting in a very high but finite PSNR. If the result is NaN or Inf, it should trigger an error or be handled specifically [S5, unofficial].
*   **Range Mismatch:** Calculating MSE between images with different normalization ranges (e.g., one in $[0,1]$ and one in $[0,255]$) will yield incorrect results.
*   **Perceptual Irrelevance:** High PSNR does not guarantee high perceptual quality. Artifacts like blurring may yield high PSNR while being visually inferior to sharper, noisier images with lower PSNR [S6, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def calculate_psnr(rendered: list, ground_truth: list, max_val: float = 1.0, mse_clamp_min: float = 1e-12) -> float:
    """
    Calculate PSNR between rendered and ground truth images.
    
    Args:
        rendered: Flattened list or array of pixel values (predicted).
        ground_truth: Flattened list or array of pixel values (target).
        max_val: The maximum possible pixel value (e.g., 1.0 for [0,1], 255 for [0,255]).
        mse_clamp_min: Minimum value for MSE to prevent log(0).
        
    Returns:
        PSNR value in dB.
    """
    if len(rendered) != len(ground_truth):
        raise ValueError("Rendered and ground truth images must have the same number of elements.")
    
    n = len(rendered)
    mse_sum = 0.0
    
    for i in range(n):
        diff = rendered[i] - ground_truth[i]
        mse_sum += diff * diff
        
    mse = mse_sum / n
    
    # Clamp MSE to prevent log(0)
    mse = max(mse, mse_clamp_min)
    
    # Calculate PSNR
    # Formula: 10 * log10((max_val^2) / MSE)
    # Equivalent to: 20 * log10(max_val / sqrt(MSE))
    # Or: -10 * log10(MSE) if max_val is 1.0
    
    if max_val == 1.0:
        psnr = -10.0 * math.log10(mse)
    else:
        psnr = 10.0 * math.log10((max_val ** 2) / mse)
        
    return psnr
```

## Subtopics

### [PSNR-based Quality Estimation for Ray Casting](9927-psnr-based-quality-estimation-for-ray-casting.md)
This method provides an objective, numerical metric for estimating sampling artifacts in Direct Volume Rendering (DVR) by treating rendered images as noisy signals. Unlike traditional assessment which relies on subjective human visual judgment, it quantifies the dispersion of pixel colors across multiple renders with different stochastic jitter seeds to produce a Peak Signal-to-Noise Ratio (PSNR) value. It is distinct from standard PSNR calculations for image compression because it measures the variance of the rendering algorithm itself rather than the error between a rendered image and a ground-truth reference.

## Operating characteristics
*   **Time Complexity:** $O(N)$, where $N$ is the number of pixels. It is extremely cheap to compute compared to perceptual metrics like LPIPS which require neural network inference [S6, unofficial].
*   **Memory Complexity:** $O(N)$ to store the images during comparison, or $O(1)$ if computed in a streaming fashion.
*   **Strengths:** Simple to implement, fast, and provides a single scalar value for easy comparison across experiments [S6, unofficial].
*   **Limitations:** Insensitive to perceptual quality; correlates poorly with human judgment compared to SSIM or LPIPS [S6, unofficial]. It penalizes high-frequency errors heavily and may favor over-smoothed results [S6, unofficial].
*   **When to use:** As a baseline metric for rapid iteration and comparison of reconstruction accuracy.
*   **When not to use:** When perceptual fidelity is the primary goal; in such cases, LPIPS or SSIM should be used alongside or instead of PSNR [S6, unofficial].

## Background
PSNR originated in signal processing and image compression as a measure of reconstruction quality [S7, unofficial]. In the context of 3D rendering and NeRFs, it has become a standard metric for evaluating novel view synthesis [S1, unofficial]. It is often reported alongside SSIM and LPIPS to provide a more comprehensive evaluation [S6, unofficial]. The metric is widely used in benchmarks such as Mip-NeRF 360, Tanks & Temples, and NeRF Synthetic [S6, unofficial].

## Sources
[S1] Differentiable Voxel-based X-ray Rendering Improves Sparse-View 3D CBCT Reconstruction, https://doi.org/10.48550/arxiv.2411.19224, fetched 2026-10-26 (unofficial)
[S2] damn8daniel/nerf-from-scratch, https://github.com/damn8daniel/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S3] Compressed volume rendering using deep learning, https://www.cs.umd.edu/sites/default/files/scholarly_papers/Jain.pdf, fetched 2026-10-26 (unofficial)
[S4] hbb1/2d-gaussian-splatting, https://github.com/hbb1/2d-gaussian-splatting, fetched 2026-10-27 (unofficial)
[S5] 3DAgentWorld/Advanced3DGS, https://github.com/3DAgentWorld/Advanced3DGS, fetched 2026-10-27 (unofficial)
[S6] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S7] Bournemouth University Research Online [BURO] - A Novel Implicit Neural Representation for Volume Data., https://eprints.bournemouth.ac.uk/38355/, fetched 2026-10-29 (unofficial)
