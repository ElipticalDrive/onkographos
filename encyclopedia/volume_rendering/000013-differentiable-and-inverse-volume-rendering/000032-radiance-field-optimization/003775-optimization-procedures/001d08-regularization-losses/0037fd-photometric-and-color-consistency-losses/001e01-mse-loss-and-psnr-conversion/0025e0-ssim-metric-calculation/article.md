# SSIM Metric Calculation

Also called: SSIM, Structural Similarity Index Measure, Structural Similarity Index

## Overview
SSIM is an image fidelity score used to evaluate the quality of 3D reconstructions and novel 2D views by measuring local structural similarity between a rendered image and a ground truth image [S1, unofficial][S2, S5]. Unlike pixel-wise metrics such as MSE, SSIM captures local patterns of pixel intensities and is correlated with human visual system perception, assuming the human visual system is highly adapted to structured natural scene information [S2][S3, unofficial]. It sets itself apart from its siblings by explicitly modeling luminance, contrast, and structure, whereas PSNR relies solely on global error magnitude and Normal-L2 focuses on geometric normals [S2][S4, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Two images (or image patches) $x$ and $y$ (e.g., rendered RGB and ground truth RGB).
- Output: A scalar SSIM value in the range $[-1, 1]$, where 1 indicates identical images [S4, S6, unofficial].

**Data Structures:**
- **Images:** 4D tensors with shape $(N, C, H, W)$, where $N$ is batch size, $C$ is channels (typically 3 for RGB), $H$ is height, and $W$ is width [S6, unofficial].
- **Kernel:** A 2D Gaussian window of size $K \times K$ (typically $11 \times 11$) with standard deviation $\sigma = 1.5$, normalized to unit sum [S2][S4, S6, unofficial].

**Parameters:**
- Window size ($K$): 11 [S2][S4, S6, unofficial].
- Stride: 1 [S2].
- Gaussian standard deviation ($\sigma$): 1.5 [S2][S4, S6, unofficial].
- Stability constants: $C_1 = (0.01)^2$, $C_2 = (0.03)^2$ [S4, S6, unofficial]. Note: Some formulations use $C_3 = C_2/2$ for the structure term, but the combined formula below is standard [S2][S4, unofficial].

**Steps:**
1. **Kernel Generation:** Create a 1D Gaussian kernel of size $K$ with $\sigma=1.5$. Create a 2D separable window via outer product. Expand to match image channels if necessary [S6, unofficial].
2. **Local Means:** Compute local means $\mu_x$ and $\mu_y$ for the prediction and target images using 2D convolution with the Gaussian kernel and padding $K//2$ [S4, S6, unofficial].
3. **Local Variances and Covariance:**
   - Compute squared means: $\mu_{x}^2$ and $\mu_{y}^2$.
   - Compute cross-mean: $\mu_{xy} = \mu_x \cdot \mu_y$.
   - Compute local variances $\sigma_x^2$ and $\sigma_y^2$ by convolving squared images ($x^2, y^2$) with the kernel and subtracting squared means [S6, unofficial].
   - Compute local covariance $\sigma_{xy}$ by convolving the product of images ($x \cdot y$) with the kernel and subtracting the cross-mean [S6, unofficial].
4. **SSIM Calculation:** Calculate the SSIM index for each local window using the formula:
   $$ \text{SSIM}(x, y) = \frac{(2\mu_x\mu_y + C_1)(2\sigma_{xy} + C_2)}{(\mu_x^2 + \mu_y^2 + C_1)(\sigma_x^2 + \sigma_y^2 + C_2)} $$
   [S4, unofficial][S5][S6, unofficial].
5. **Mean SSIM (MSSIM):** Return the mean of all pixel-wise SSIM values [S2][S6, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def gaussian_kernel(window_size: int, sigma: float) -> torch.Tensor:
    """Generate a 1D Gaussian kernel."""
    coords = torch.arange(window_size, dtype=torch.float32)
    coords -= window_size // 2
    g = torch.exp(-(coords ** 2) / (2 * sigma ** 2))
    g /= g.sum()
    return g

def ssim_metric(image: torch.Tensor, target: torch.Tensor, window_size: int = 11, sigma: float = 1.5) -> torch.Tensor:
    """
    Compute SSIM between image and target.
    Assumes inputs are 4D tensors (N, C, H, W) with values in [0, 1] or similar range.
    """
    # 1. Generate 2D Gaussian window
    kernel_1d = gaussian_kernel(window_size, sigma)
    kernel_2d = torch.outer(kernel_1d, kernel_1d)
    # Expand to match channels: (C, 1, K, K) for groups convolution
    window = kernel_2d.unsqueeze(0).unsqueeze(0).repeat(image.shape[1], 1, 1, 1)
    
    # Constants for stability
    c1 = (0.01 ** 2)
    c2 = (0.03 ** 2)
    
    # 2. Compute local means
    # padding ensures output size matches input size
    mu1 = F.conv2d(image, window, padding=window_size // 2, groups=image.shape[1])
    mu2 = F.conv2d(target, window, padding=window_size // 2, groups=target.shape[1])
    
    # 3. Compute local variances and covariance
    mu1_sq = mu1 ** 2
    mu2_sq = mu2 ** 2
    mu12 = mu1 * mu2
    
    sigma1_sq = F.conv2d(image ** 2, window, padding=window_size // 2, groups=image.shape[1]) - mu1_sq
    sigma2_sq = F.conv2d(target ** 2, window, padding=window_size // 2, groups=target.shape[1]) - mu2_sq
    sigma12 = F.conv2d(image * target, window, padding=window_size // 2, groups=image.shape[1]) - mu12
    
    # 4. Calculate SSIM
    numerator = (2 * mu12 + c1) * (2 * sigma12 + c2)
    denominator = (mu1_sq + mu2_sq + c1) * (sigma1_sq + sigma2_sq + c2)
    
    ssim_map = numerator / denominator
    
    # 5. Return mean SSIM
    return ssim_map.mean()
```

**Edge Cases and Pitfalls:**
- Input tensors must be 4D with shape $(N, C, H, W)$ [S4, S6, unofficial].
- Image shape mismatch between render and GT tensors raises an error [S6, unofficial].
- The denominator can become unstable if variances are near zero; constants $C_1$ and $C_2$ prevent division by zero [S5].

## Subtopics

### SSIM Window Initialization
Creates and caches a 2D Gaussian window for SSIM loss computation to avoid recomputing it per batch. Uses a window size of $11 \times 11$ and sigma of 1.5 [S7, unofficial].

### GMSD Metric Calculation
Gradient Magnitude Similarity Deviation (GMSD) evaluates the deviation in gradient magnitude between the original and reconstructed images. It is effective in evaluating the preservation of edges and fine details, particularly in regions with high spatial variability [S8].

## Operating characteristics
- **Time Complexity:** $O(N \cdot H \cdot W \cdot K^2)$ per image, where $N$ is batch size, $H, W$ are dimensions, and $K$ is window size. Using separable convolution reduces this to $O(N \cdot H \cdot W \cdot K)$.
- **Memory Complexity:** $O(N \cdot H \cdot W)$ for storing intermediate maps (means, variances, SSIM map).
- **Performance:** Standard benchmark for image quality in NeRF and 3D reconstruction [S4, unofficial]. Target SSIM is often 0.95+ for high-quality reconstructions [S4, unofficial].
- **Strengths:** Correlates well with human perception; captures structural information [S2][S3, unofficial].
- **Limitations:** Computationally more expensive than MSE; sensitive to misalignment; assumes local stationarity [S3, unofficial].

## Background
SSIM was proposed by Wang et al. in 2004 [S3, S4, unofficial]. It is widely used for evaluating neural rendering methods like NeRF [S2]. It measures how structural information changes to provide an approximation of perceived image distortion [S3, unofficial].

## Sources
[S1] Differentiable Voxel-based X-ray Rendering Improves Sparse-View 3D CBCT Reconstruction, https://doi.org/10.48550/arxiv.2411.19224, fetched 2026-10-26 (unofficial)
[S2] S3IM: Stochastic Structural SIMilarity and Its Unreasonable Effectiveness for Neural Fields, https://doi.org/10.1109/iccv51070.2023.01652, fetched 2026-10-26
[S3] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
[S4] emmanueladutwum123/nerf-from-scratch, https://github.com/emmanueladutwum123/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S5] Discretized Gaussian Representation for Tomographic Reconstruction, https://doi.org/10.1109/ICCV51701.2025.02325, fetched 2026-10-27
[S6] 3DAgentWorld/Advanced3DGS, https://github.com/3DAgentWorld/Advanced3DGS, fetched 2026-10-27 (unofficial)
[S7] Show HN: Msplat – 3D Gaussian Splatting training in ~90s on M4 Max, native Metal, https://github.com/rayanht/msplat, fetched 2026-10-26 (unofficial)
[S8] The Adaption of Recent New Concepts in Neural Radiance Fields and Their Role for High-Fidelity Volume Reconstruction in Medical Images., https://doi.org/10.3390/s24185923, fetched 2026-10-26
