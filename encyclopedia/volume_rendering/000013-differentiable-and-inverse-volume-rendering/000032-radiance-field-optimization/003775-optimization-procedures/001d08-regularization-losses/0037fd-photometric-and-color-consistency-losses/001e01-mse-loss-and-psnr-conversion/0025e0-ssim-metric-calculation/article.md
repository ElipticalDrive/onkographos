# SSIM Metric Calculation

Also called: SSIM Metric, SSIM computation, Structural Similarity Index

## Overview
The Structural Similarity Index (SSIM) measures the perceived distortion between a rendered image and a ground truth image by comparing local patterns of pixel intensities, assuming the human visual system is highly adapted to structured natural scene information [S1, unofficial][S3]. It serves as a standard benchmark for image quality in radiance-field optimization, complementing pixel-wise metrics like MSE and PSNR [S2, unofficial][S3]. It provides a value between -1 and 1, where 1 indicates identical images [S2, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Two images, $x$ (prediction/rendered) and $y$ (target/ground truth), typically 4D tensors of shape $(N, C, H, W)$ or $(1, 3, H, W)$ [S2, unofficial][S4, unofficial].
- Output: A scalar SSIM score representing the mean structural similarity across the image [S2, unofficial][S4, unofficial].

**Data Structures:**
- **Gaussian Kernel:** A 2D separable window generated from a 1D Gaussian kernel [S4, unofficial].
- **Local Statistics:** Tensors for local means ($\mu$), squared means ($\mu^2$), variances ($\sigma^2$), and covariance ($\sigma_{xy}$) computed via convolution [S4, unofficial].

**Parameters:**
- `window_size`: Size of the Gaussian window. Default is 11 [S2, unofficial][S4, unofficial].
- `sigma`: Standard deviation for the Gaussian kernel. Default is 1.5 [S2, unofficial][S4, unofficial].
- `c1`: Stability constant for the mean difference denominator. Default is $0.01^2$ [S2, unofficial][S4, unofficial].
- `c2`: Stability constant for the variance/covariance denominator. Default is $0.03^2$ [S2, unofficial][S4, unofficial].

**Steps:**
1. **Kernel Generation:** Create a 1D Gaussian kernel of size `window_size` with standard deviation `sigma` [S4, unofficial]. Construct a 2D separable window by taking the outer product of the 1D kernel with itself [S4, unofficial]. Expand the window to match the number of image channels (e.g., if channels=3, the kernel is applied per-channel or grouped) [S4, unofficial].
2. **Local Means:** Compute local means $\mu_x$ and $\mu_y$ for the prediction and target images by performing 2D convolution with the Gaussian kernel [S4, unofficial]. Use padding equal to `window_size // 2` to preserve spatial dimensions [S4, unofficial]. To handle multi-channel images efficiently, use grouped convolution where the group size equals the number of channels [S4, unofficial].
   $$ \mu_x = \text{conv2d}(x, \text{window}, \text{padding}=\text{window\_size}//2, \text{groups}=C) $$
   $$ \mu_y = \text{conv2d}(y, \text{window}, \text{padding}=\text{window\_size}//2, \text{groups}=C) $$
3. **Squared Means and Cross-Mean:** Compute $\mu_x^2$, $\mu_y^2$, and $\mu_{xy}$ (the product of means) element-wise [S4, unofficial].
   $$ \mu_{x\_sq} = \mu_x^2 $$
   $$ \mu_{y\_sq} = \mu_y^2 $$
   $$ \mu_{xy} = \mu_x \cdot \mu_y $$
4. **Variances and Covariance:** Compute local variances $\sigma_x^2$, $\sigma_y^2$ and covariance $\sigma_{xy}$ by convolving the squared images and the product of images, then subtracting the squared means [S4, unofficial].
   $$ \sigma_{x\_sq} = \text{conv2d}(x^2, \text{window}, \text{padding}=\text{window\_size}//2, \text{groups}=C) - \mu_{x\_sq} $$
   $$ \sigma_{y\_sq} = \text{conv2d}(y^2, \text{window}, \text{padding}=\text{window\_size}//2, \text{groups}=C) - \mu_{y\_sq} $$
   $$ \sigma_{xy} = \text{conv2d}(x \cdot y, \text{window}, \text{padding}=\text{window\_size}//2, \text{groups}=C) - \mu_{xy} $$
5. **SSIM Map Calculation:** Calculate the per-pixel SSIM value using the formula [S1, unofficial][S2, unofficial][S3][S4, unofficial]:
   $$ \text{SSIM}(x, y) = \frac{(2\mu_{xy} + c_1)(2\sigma_{xy} + c_2)}{(\mu_{x\_sq} + \mu_{y\_sq} + c_1)(\sigma_{x\_sq} + \sigma_{y\_sq} + c_2)} $$
6. **Aggregation:** Return the mean of all pixel values in the SSIM map [S4, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def gaussian_kernel(window_size: int, sigma: float) -> torch.Tensor:
    """Create a 1D Gaussian kernel."""
    coords = torch.arange(window_size, dtype=torch.float32)
    coords -= window_size // 2
    g = torch.exp(-(coords ** 2) / (2 * sigma ** 2))
    g /= g.sum()
    return g

def ssim_metric(pred: torch.Tensor, target: torch.Tensor, window_size: int = 11, sigma: float = 1.5, c1: float = 0.01**2, c2: float = 0.03**2) -> torch.Tensor:
    """
    Compute SSIM between prediction and target.
    Inputs: pred, target of shape (N, C, H, W) or (1, C, H, W).
    """
    # Validate input shapes
    if pred.shape != target.shape:
        raise ValueError(f"Image shape mismatch: pred {pred.shape} vs target {target.shape}")
    
    # Ensure 4D input
    if pred.dim() == 3:
        pred = pred.unsqueeze(0)
        target = target.unsqueeze(0)
        
    N, C, H, W = pred.shape
    
    # 1. Generate 2D Gaussian Window
    # Create 1D kernel
    kernel_1d = gaussian_kernel(window_size, sigma)
    # Create 2D separable kernel via outer product
    kernel_2d = torch.outer(kernel_1d, kernel_1d)
    # Reshape for convolution: (1, 1, H_k, W_k)
    kernel_2d = kernel_2d.unsqueeze(0).unsqueeze(0)
    
    # Expand kernel to match channels for grouped convolution
    # We repeat the same kernel for each channel group
    kernel = kernel_2d.repeat(C, 1, 1, 1)
    
    padding = window_size // 2
    
    # 2. Compute Local Means
    mu_pred = F.conv2d(pred, kernel, padding=padding, groups=C)
    mu_target = F.conv2d(target, kernel, padding=padding, groups=C)
    
    # 3. Compute Squared Means and Cross Mean
    mu_pred_sq = mu_pred ** 2
    mu_target_sq = mu_target ** 2
    mu_pred_target = mu_pred * mu_target
    
    # 4. Compute Variances and Covariance
    # Convolve squared inputs
    pred_sq = pred ** 2
    target_sq = target ** 2
    pred_target = pred * target
    
    conv_pred_sq = F.conv2d(pred_sq, kernel, padding=padding, groups=C)
    conv_target_sq = F.conv2d(target_sq, kernel, padding=padding, groups=C)
    conv_pred_target = F.conv2d(pred_target, kernel, padding=padding, groups=C)
    
    sigma_pred_sq = conv_pred_sq - mu_pred_sq
    sigma_target_sq = conv_target_sq - mu_target_sq
    sigma_pred_target = conv_pred_target - mu_pred_target
    
    # 5. Calculate SSIM Map
    # Numerator: (2 * mu_pred_target + c1) * (2 * sigma_pred_target + c2)
    num = (2 * mu_pred_target + c1) * (2 * sigma_pred_target + c2)
    # Denominator: (mu_pred_sq + mu_target_sq + c1) * (sigma_pred_sq + sigma_target_sq + c2)
    den = (mu_pred_sq + mu_target_sq + c1) * (sigma_pred_sq + sigma_target_sq + c2)
    
    ssim_map = num / den
    
    # 6. Return Mean SSIM
    return ssim_map.mean()
```

## Subtopics

### SSIM Window Initialization
Creates and caches a 2D Gaussian window for SSIM loss computation to avoid recomputing the kernel in every iteration. The window size is fixed at 11x11 with a sigma of 1.5 [S5, unofficial].

### Accelerated SH and Differentiable Loss Computation
Optimizations for Spherical Harmonic (SH) coefficient updates and SSIM loss computation to reduce training time, specifically targeting the Adam optimizer bottleneck [S6].
- **SH Update Schedule:** Switches all SH bands beyond the first (0th band) to a batched update schedule, performing only one step of Adam optimization every 16 iterations. The SH learning rate is increased four times to 0.001, while the opacity learning rate is reduced by half to 0.025 [S6].
- **SH Data Layout:** Extends the differential rasterizer to load Gaussian SH coefficients from separate tensors, avoiding combining the 0th SH band and higher bands into a single tensor before rasterization [S6].
- **SSIM Loss Optimization:** Performs differentiable 2D convolution via two consecutive 1D convolutions using optimized CUDA kernels, leveraging the separability of Gaussian kernels. Uses a fused kernel for the evaluation of the SSIM metric from the convolved results [S6].

## Operating characteristics
- **Time Complexity:** The standard implementation involves multiple 2D convolutions [S4, unofficial]. Optimized implementations use separable 1D convolutions to reduce computation [S6].
- **Memory Complexity:** Requires storing intermediate tensors for means, variances, and covariance, proportional to the image size $H \times W$ [S4, unofficial].
- **Performance:** Target SSIM scores for high-quality reconstructions are typically 0.95+ [S2, unofficial].
- **Strengths:** Captures structural information and perceived quality better than pixel-wise metrics [S1, unofficial][S3].
- **Limitations:** The main downside is its complexity and the nonlinearity of the human visual system [S1, unofficial].
- **When to use:** Used as a loss term in differentiable rendering frameworks like 3D Gaussian Splatting to encourage structural fidelity [S6].

## Background
The Structural Similarity Index was proposed by Wang et al. in 2004 [S1, unofficial][S2, unofficial]. It was designed to address the limitations of traditional metrics like MSE and PSNR by modeling image distortion as a change in structural information, based on the assumption that the human visual system is highly adapted to structured natural scene information [S1, unofficial]. It has since become a standard benchmark for image quality assessment in computer vision and graphics [S2, unofficial][S3].

## Sources
[S1] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
[S2] emmanueladutwum123/nerf-from-scratch, https://github.com/emmanueladutwum123/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S3] Discretized Gaussian Representation for Tomographic Reconstruction, https://doi.org/10.1109/ICCV51701.2025.02325, fetched 2026-10-27
[S4] 3DAgentWorld/Advanced3DGS, https://github.com/3DAgentWorld/Advanced3DGS, fetched 2026-10-07 (unofficial)
[S5] Show HN: Msplat – 3D Gaussian Splatting training in ~90s on M4 Max, native Metal, https://github.com/rayanht/msplat, fetched 2026-10-26 (unofficial)
[S6] Taming 3DGS: High-Quality Radiance Fields with Limited Resources, https://doi.org/10.1145/3680528.3687694, fetched 2026-10-30
