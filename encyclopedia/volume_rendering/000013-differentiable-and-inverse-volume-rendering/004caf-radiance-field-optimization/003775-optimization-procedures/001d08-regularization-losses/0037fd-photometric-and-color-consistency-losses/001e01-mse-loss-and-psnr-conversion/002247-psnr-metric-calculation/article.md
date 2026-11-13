# PSNR Metric Calculation

Also called: PSNR Calculation, PSNR Metric, Peak Signal-to-Noise Ratio

## Overview
Peak Signal-to-Noise Ratio (PSNR) is a standard benchmark metric used to evaluate the photometric quality of reconstructed images or volumes in novel view synthesis and tomographic reconstruction [S1, unofficial][S2, unofficial][S4]. It quantifies the fidelity of a predicted image $y$ against a ground-truth reference $x$ by converting the Mean Squared Error (MSE) into a logarithmic decibel (dB) scale, where higher values indicate better quality [S2, unofficial][S3, unofficial]. Unlike structural metrics such as SSIM, PSNR focuses strictly on pixel-wise intensity error, making it sensitive to large smooth regions but less responsive to small, high-frequency features like specular highlights [S1, unofficial].

## How it works
**Inputs and Outputs**
- **Inputs**: 
  - Two images or volumes $x$ (ground truth) and $y$ (prediction/reconstruction).
  - `max_val`: The maximum possible pixel value (e.g., 1.0 for normalized float, 255 for 8-bit integer) [S2, unofficial][S4].
  - `eps`: A small constant to prevent division by zero or log(0) [S3, unofficial].
- **Output**: A single floating-point number representing the PSNR in dB.

**Data Structures**
- The inputs are typically treated as arrays of shape $(H, W, C)$ for images or $(D, H, W, C)$ for volumes, where $C$ is the number of channels (usually 3 for RGB) [S5, unofficial][S6, unofficial].
- The MSE is computed over all pixels/voxels and all channels.

**Steps**
1. **Validation**: Ensure $x$ and $y$ have identical dimensions. If not, the metric is undefined or returns an error/maximum value [S6, unofficial].
2. **Compute MSE**: Calculate the Mean Squared Error between $x$ and $y$.
   - For each pixel/voxel $i$, compute the squared difference across all channels.
   - Average these squared differences over the total number of elements $N$ (where $N = H \times W \times C$ or $D \times H \times W \times C$).
3. **Compute PSNR**: Apply the logarithmic conversion using `max_val` and the computed MSE.
4. **Handle Edge Cases**: If MSE is 0, PSNR is infinite. In practice, a small epsilon is added to MSE to avoid numerical instability [S3, unofficial][S2, unofficial].

**Formulas**
The standard definition of PSNR is [S1, unofficial][S4]:
$$ \text{PSNR} = 10 \log_{10} \left( \frac{\text{MAX}^2}{\text{MSE}} \right) $$

This is algebraically equivalent to [S2, unofficial][S5, unofficial]:
$$ \text{PSNR} = 20 \log_{10} \left( \frac{\text{MAX}}{\sqrt{\text{MSE}}} \right) $$

Where:
- $\text{MAX}$ is the maximum possible pixel value (`max_val`) [S4].
- $\text{MSE}$ is the mean squared error:
  $$ \text{MSE} = \frac{1}{N} \sum_{i=1}^{N} (x_i - y_i)^2 $$
  Here, $x_i$ and $y_i$ represent the vector of channel values at the $i$-th pixel/voxel [S6, unofficial].

**Parameters**
- `max_val`: Default is 1.0 for normalized images [S2, unofficial]. If images are in [0, 255], this should be 255.
- `eps`: Default is $10^{-10}$ to prevent $\log(0)$ [S3, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def calculate_psnr(x, y, max_val=1.0, eps=1e-10):
    """
    Calculate Peak Signal-to-Noise Ratio between two images/volumes x and y.
    
    Args:
        x: Ground truth array (numpy/torch tensor).
        y: Predicted array (same shape as x).
        max_val: Maximum possible pixel value (default 1.0).
        eps: Small constant to prevent log(0) (default 1e-10).
        
    Returns:
        psnr: Float value in dB.
    """
    # Validate dimensions
    if x.shape != y.shape:
        raise ValueError("Input arrays must have identical dimensions.")
    
    # Compute squared differences
    # x and y are assumed to be floating point
    diff = x - y
    squared_diff = diff ** 2
    
    # Compute MSE over all elements
    # N is the total number of elements (pixels * channels)
    mse = numpy.mean(squared_diff)
    
    # Calculate PSNR
    # Using the 10 * log10(MAX^2 / MSE) formulation
    # Add eps to MSE to avoid log(0) if prediction is perfect
    psnr = 10 * math.log10((max_val ** 2) / (mse + eps))
    
    return psnr
```

**Edge Cases and Pitfalls**
- **Perfect Prediction**: If $x = y$, MSE is 0, leading to $\log(\infty)$. The epsilon term handles this, returning a very large finite number [S2, unofficial].
- **Dimension Mismatch**: If input textures/images have different widths or heights, the metric is invalid. Some implementations return `float.MaxValue` and log an error [S6, unofficial].
- **Weighting Bias**: PSNR averages over every pixel, weighting large smooth regions heavily. It ignores small bright features like specular highlights or grass blades, which occupy too few pixels to significantly affect the score [S1, unofficial].
- **Interpretation**: The scale is logarithmic. A difference of 3 dB corresponds to a halving of the squared error, and 6 dB corresponds to a quartering [S1, unofficial].

## Subtopics

### [3D Evaluation Metrics](2903-3d-evaluation-metrics.md)
3D Evaluation Metrics quantify the fidelity of reconstructed volumetric or surface data against ground-truth references, addressing the problem of assessing geometric and photometric accuracy in 3D reconstruction pipelines. Unlike PSNR which focuses on pixel-level photometric error in 2D projections, this method encompasses both volumetric image quality (PSNR, SSIM) and surface geometric accuracy (Normal Consistency, Chamfer Distance). It is distinguished from Benchmark Performance Numbers by providing the specific mathematical definitions and implementation details for these quality scores rather than just reporting aggregate results.

### [Benchmark Performance Numbers](3471-benchmark-performance-numbers.md)
Benchmark Performance Numbers provide standardized quantitative baselines for evaluating radiance field optimization methods, primarily using the Mip-NeRF 360 dataset metrics (average PSNR and LPIPS). They serve as a reference point to compare the reconstruction quality of different variants (e.g., 3DGS, Mip-Splatting) against established baselines. Unlike raw metric calculation, these numbers aggregate results across a specific scene set to highlight trade-offs between visual fidelity, geometric accuracy, and resource efficiency.

### [PSNR Difference Metric for Ablation](5656-psnr-difference-metric-for-ablation.md)
The PSNR Difference Metric for Ablation quantifies the performance degradation caused by removing a specific component from a rendering pipeline by comparing the Peak Signal-to-Noise Ratio (PSNR) of the ablated configuration against the full method. It isolates the error ratio independent of the bit depth (MAX), as MAX cancels out in the logarithmic difference, providing a scale-invariant measure of quality loss.

### Noise Floor Estimation via MAD
Noise Floor Estimation via MAD estimates the noise standard deviation $\sigma$ using Median Absolute Deviation (MAD) estimators to determine the theoretical maximum PSNR (noise floor) for a volume [S7, unofficial].
- **Laplacian MAD**: Applies the D-dimensional discrete Laplacian and scales the MAD of the response by $1/(0.6745 \sqrt{2D(2D+1)})$ [S7, unofficial].
- **Haar Finite-Difference MAD**: Computes diagonal second differences $d = I(y,x) - I(y,x+1) - I(y+1,x) + I(y+1,x+1)$ and scales the median of $|d|$ by $1/(0.6745 \sqrt{2})$ [S7, unofficial].
- **Formula**: $\text{Noise Floor PSNR} = 10 \log_{10}(1/\sigma^2)$ [S7, unofficial].
- **Pitfall**: MAD estimators may return exactly zero for some volumes (e.g., Mouse heart nuclei), indicating the model is signal-limited, not noise-limited, and thus bounding nothing [S7, unofficial].

### PSNR Loss
PSNR Loss computes the Peak Signal-to-Noise Ratio between predicted and target images, used as a regression metric during training [S8, unofficial].
- **Formula**: $\text{PSNR} = 20 \log_{10}\left(\frac{\text{max\_val}}{\sqrt{\text{MSE}}}\right)$ [S8, unofficial].
- **Parameter**: `max_val` (default 1.0) is the maximum possible pixel value in the target image [S8, unofficial].

## Operating characteristics
- **Time Complexity**: $O(N)$, where $N$ is the total number of pixels/voxels. The calculation involves a single pass to compute squared differences and a mean operation.
- **Memory Complexity**: $O(N)$ if intermediate difference arrays are stored, or $O(1)$ if computed in a streaming fashion.
- **Measured Performance**: For NeRF paper Blender scenes, a target PSNR is typically 31–33 dB [S2, unofficial].
- **Strengths**: Simple to compute, widely understood, and provides a single scalar value for comparison.
- **Limitations**: Does not correlate well with human perception for high-frequency details or structural similarities. It is heavily biased towards large, smooth regions of the image [S1, unofficial].
- **When to Use**: Quick baseline evaluation of photometric accuracy, especially when structural fidelity is less critical than intensity accuracy.
- **When Not to Use**: When perceptual quality is the primary concern (use SSIM or LPIPS instead) or when evaluating geometric accuracy.

## Background
PSNR originated as a standard metric in image and video processing to evaluate compression algorithms and reconstruction quality [S4]. It has been widely adopted in the field of novel view synthesis and radiance field optimization (e.g., NeRF, 3D Gaussian Splatting) as a primary benchmark for photometric fidelity [S1, unofficial][S2, unofficial]. The metric is often reported alongside SSIM and LPIPS to provide a more comprehensive evaluation of reconstruction quality [S1, unofficial].

## Sources
[S1] An Architectural Review of the 3D Gaussian Splatting Rendering Pipeline, https://doi.org/10.5281/zenodo.21739653, fetched 2026-10-26 (unofficial)
[S2] emmanueladutwum123/nerf-from-scratch, https://github.com/emmanueladutwum123/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S3] yashtmy/nerf-3d-vision, https://github.com/yashtmy/nerf-3d-vision, fetched 2026-10-26 (unofficial)
[S4] Discretized Gaussian Representation for Tomographic Reconstruction, https://doi.org/10.1109/ICCV51701.2025.02325, fetched 2026-10-27
[S5] ayush-chauhan-1202/physics-aware-ct-reconstruction, https://github.com/ayush-chauhan-1202/physics-aware-ct-reconstruction, fetched 2026-10-27 (unofficial)
[S6] andersblomqvist/unity-nanovdb-renderer, https://github.com/andersblomqvist/unity-nanovdb-renderer, fetched 2026-10-27 (unofficial)
[S7] Luxar: Gaussian splatting for microscopy and scalable interactive web visualisation of multidimensional scientific data, https://doi.org/10.64898/2026.09.22.753676, fetched 2026-10-26 (unofficial)
[S8] suvadityamuk/NeRF-Experiments, https://github.com/suvadityamuk/NeRF-Experiments, fetched 2026-10-27 (unofficial)
