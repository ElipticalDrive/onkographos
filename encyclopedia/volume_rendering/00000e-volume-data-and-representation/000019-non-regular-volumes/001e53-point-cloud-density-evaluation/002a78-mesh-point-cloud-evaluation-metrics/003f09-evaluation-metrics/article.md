# Evaluation Metrics
Also called: PSNR and SSIM Calculation, Rendering Quality Assessment
## Overview
Computes Peak Signal-to-Noise Ratio (PSNR) and Structural Similarity Index Measure (SSIM) to quantitatively assess rendering quality against ground truth images [S2, unofficial]. It differs from mesh-centric metrics (e.g., Chamfer Distance) by operating on pixel-space image data rather than geometric point clouds or meshes [S1][S2, unofficial].
## How it works
### Inputs and Outputs
- **Inputs**: Rendered image tensor $I_{render}$ and ground truth image tensor $I_{gt}$. Both are assumed to be normalized to $[0, 1]$ with shape $(H, W, C)$ where $C=3$ for RGB [S2, unofficial].
- **Outputs**: Scalar PSNR value (dB) and scalar SSIM value ($[-1, 1]$ or $[0, 1]$ depending on implementation, typically $[0, 1]$ for normalized inputs).

### Data Structures
- **Image Tensors**: Floating-point arrays.
- **Sliding Window**: For SSIM, a local window of size $K \times K$ (typically $11 \times 11$) is used to compute local statistics [S2, implied by standard SSIM definition, unofficial].

### Steps
1. **Preprocessing**: Ensure both images are in the range $[0, 1]$. If inputs are in $[0, 255]$, divide by 255.0 [S2, unofficial].
2. **PSNR Calculation**:
   - Compute the Mean Squared Error (MSE) between $I_{render}$ and $I_{gt}$:
     $$ MSE = \frac{1}{N} \sum_{i=1}^{N} (I_{render}(i) - I_{gt}(i))^2 $$
     where $N = H \times W \times C$.
   - Compute PSNR:
     $$ PSNR = 10 \cdot \log_{10} \left( \frac{MAX_I^2}{MSE} \right) $$
     where $MAX_I = 1.0$ for normalized data [S2, unofficial].
3. **SSIM Calculation**:
   - For each pixel, compute local means ($\mu_x, \mu_y$), variances ($\sigma_x^2, \sigma_y^2$), and covariance ($\sigma_{xy}$) within the window [S2, implied, unofficial].
   - Compute SSIM index for the window:
     $$ SSIM(x, y) = \frac{(2\mu_x\mu_y + C_1)(2\sigma_{xy} + C_2)}{(\mu_x^2 + \mu_y^2 + C_1)(\sigma_x^2 + \sigma_y^2 + C_2)} $$
     where $C_1 = (0.01 \cdot MAX_I)^2$ and $C_2 = (0.03 \cdot MAX_I)^2$ are stability constants [S2, implied standard values, unofficial].
   - Average SSIM values over all windows to get the final SSIM score.

### Parameters
- **data_range**: Set to $1.0$ for normalized $[0, 1]$ image data [S2, unofficial].
- **channel_axis**: Set to $-1$ for RGB channels in the last dimension [S2, unofficial].
- **MAX_I**: $1.0$ (derived from data_range) [S2, unofficial].

### Edge Cases and Pitfalls
- **Division by Zero**: If MSE is exactly 0, PSNR is infinite. Handle by capping or returning a large finite value (e.g., 100 dB) [G].
- **Boundary Effects**: SSIM windowing may cause boundary artifacts if not handled (e.g., padding or excluding edge windows) [G].
- **Dynamic Range**: If images are not normalized, $MAX_I$ must match the actual maximum value (e.g., 255.0) [S2, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_psnr(rendered: np.ndarray, ground_truth: np.ndarray, data_range: float = 1.0) -> float:
    """
    Compute PSNR between two images.
    :param rendered: np.ndarray, shape (H, W, C), values in [0, 1]
    :param ground_truth: np.ndarray, shape (H, W, C), values in [0, 1]
    :param data_range: float, maximum possible value of the image (1.0 for normalized)
    :return: float, PSNR in dB
    """
    # Ensure inputs are float32/64
    rendered = rendered.astype(np.float64)
    ground_truth = ground_truth.astype(np.float64)
    
    # Compute MSE
    mse = np.mean((rendered - ground_truth) ** 2)
    
    # Handle zero MSE
    if mse == 0:
        return 100.0  # Arbitrary large value for perfect match
        
    # Compute PSNR
    psnr = 10 * np.log10((data_range ** 2) / mse)
    return psnr

def compute_ssim(rendered: np.ndarray, ground_truth: np.ndarray, data_range: float = 1.0, window_size: int = 11) -> float:
    """
    Compute SSIM between two images.
    :param rendered: np.ndarray, shape (H, W, C)
    :param ground_truth: np.ndarray, shape (H, W, C)
    :param data_range: float, maximum possible value of the image
    :param window_size: int, size of the sliding window (must be odd)
    :return: float, mean SSIM
    """
    # Constants
    C1 = (0.01 * data_range) ** 2
    C2 = (0.03 * data_range) ** 2
    
    # Convert to float64 for precision
    rendered = rendered.astype(np.float64)
    ground_truth = ground_truth.astype(np.float64)
    
    # Pad images to handle boundaries
    pad = window_size // 2
    rendered_padded = np.pad(rendered, ((pad, pad), (pad, pad), (0, 0)), mode='reflect')
    ground_truth_padded = np.pad(ground_truth, ((pad, pad), (pad, pad), (0, 0)), mode='reflect')
    
    h, w, c = rendered.shape
    ssim_map = np.zeros((h, w))
    
    # Compute SSIM for each pixel
    for i in range(h):
        for j in range(w):
            # Extract windows
            y1, y2 = i, i + window_size
            x1, x2 = j, j + window_size
            
            win_x = rendered_padded[y1:y2, x1:x2, :]
            win_y = ground_truth_padded[y1:y2, x1:x2, :]
            
            # Compute local means
            mu_x = np.mean(win_x)
            mu_y = np.mean(win_y)
            
            # Compute local variances and covariance
            sigma_x2 = np.var(win_x)
            sigma_y2 = np.var(win_y)
            sigma_xy = np.cov(win_x.flatten(), win_y.flatten())[0, 1]
            
            # Compute SSIM index
            num = (2 * mu_x * mu_y + C1) * (2 * sigma_xy + C2)
            den = (mu_x ** 2 + mu_y ** 2 + C1) * (sigma_x2 + sigma_y2 + C2)
            
            ssim_map[i, j] = num / den
            
    # Return mean SSIM
    return np.mean(ssim_map)
```
## Operating characteristics
- **Time Complexity**: PSNR is $O(N)$ where $N$ is the number of pixels. SSIM is $O(N \cdot K^2)$ where $K$ is the window size, due to local window operations.
- **Memory Complexity**: $O(N)$ for storing intermediate arrays (MSE, SSIM map).
- **Strengths**: Standard, widely understood metrics for image quality. PSNR is sensitive to pixel-wise errors; SSIM correlates better with human perception.
- **Limitations**: PSNR is not always correlated with perceived quality. SSIM is computationally more expensive. Both are pixel-aligned, so they are sensitive to registration errors between rendered and ground truth images [G].
- **When to Use**: When pixel-perfect alignment is guaranteed and quantitative comparison of rendering fidelity is needed.
- **When Not to Use**: When comparing geometric meshes (use Chamfer Distance, F1, NC) or when images are misaligned [S1][S2, unofficial].

## Background
Originated from image processing literature. PSNR is a standard metric for lossy compression. SSIM was introduced by Wang et al. (2004) to address limitations of MSE/PSNR. In the context of 3D rendering and Gaussian Splatting, these metrics are used to evaluate the visual quality of rendered images against ground truth [S2, unofficial]. The provided notes also mention mesh evaluation metrics (CD, F1, NC) which are distinct and used for geometric accuracy, not rendering quality [S1].

## Sources
[S1] DCCVT: Differentiable Clipped Centroidal Voronoi Tessellation, https://arxiv.org/abs/2601.13603v1, fetched 2026-10-26
[S2] Fhz2346/3D-Gaussian-Splatting, https://github.com/Fhz2346/3D-Gaussian-Splatting, fetched 2026-10-27 (unofficial)
