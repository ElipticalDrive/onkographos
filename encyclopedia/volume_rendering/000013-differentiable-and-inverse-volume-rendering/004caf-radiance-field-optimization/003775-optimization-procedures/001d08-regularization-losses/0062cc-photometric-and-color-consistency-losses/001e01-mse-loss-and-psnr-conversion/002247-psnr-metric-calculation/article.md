# PSNR Metric Calculation

Also called: PSNR Calculation, PSNR Metric, Peak Signal-to-Noise Ratio

## Overview
Peak Signal-to-Noise Ratio (PSNR) is a standard benchmark metric used to evaluate the photometric fidelity of reconstructed images or volumes in novel view synthesis and 3D reconstruction [S1, unofficial][S2, unofficial][S4]. It quantifies the error between a predicted image and a ground-truth reference by converting the Mean Squared Error (MSE) into a logarithmic decibel (dB) scale, where higher values indicate better quality [S2, unofficial][S3, unofficial]. Unlike structural metrics such as SSIM, PSNR focuses strictly on pixel-wise intensity accuracy, making it sensitive to large smooth regions while largely ignoring small, high-frequency features like specular highlights [S1, unofficial].

## How it works
**Inputs and Outputs**
- **Input `x`**: Ground-truth image or volume [S5, unofficial].
- **Input `y`**: Predicted/reconstructed image or volume [S5, unofficial].
- **Parameter `MAX`**: The maximum possible representable pixel value (e.g., 1.0 for normalized float, 255 for 8-bit integer) [S1, unofficial][S4].
- **Parameter `eps`**: A small constant (typically $10^{-10}$) added to MSE to prevent $\log(0)$ [S3, unofficial].
- **Output**: A scalar float representing the PSNR in decibels (dB) [S2, unofficial].

**Data Structures**
The inputs are dense arrays or tensors containing pixel/voxel values [S5, unofficial]. The computation aggregates errors over all elements in the array [G].

**Steps**
1. **Validation**: Ensure `x` and `y` have identical dimensions; if not, return an error or maximum float value [S6, unofficial].
2. **Compute Squared Error**: Calculate the element-wise squared difference between `x` and `y` [S5, unofficial].
3. **Compute MSE**: Calculate the mean of the squared differences over all pixels/voxels [S6, unofficial].
4. **Compute PSNR**: Apply the logarithmic conversion using `MAX` and `eps` [S2, unofficial][S3, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def calculate_psnr(x: list, y: list, max_val: float = 1.0, eps: float = 1e-10) -> float:
    """Calculate Peak Signal-to-Noise Ratio between two images/volumes x and y."""
    if len(x) != len(y):
        raise ValueError("Input arrays must have identical dimensions")
    n = len(x)
    if n == 0:
        return 0.0
    error_sum = 0.0
    for i in range(n):
        diff = x[i] - y[i]
        error_sum += diff * diff
    mse = error_sum / n
    psnr = 10.0 * math.log10((max_val ** 2) / (mse + eps))
    return psnr
```

**Parameters and Typical Values**
- `max_val`: Default is 1.0 for normalized floating-point images [S2, unofficial][S8, unofficial].
- `eps`: Default is $10^{-10}$ [S3, unofficial].
- **Interpretation**: A difference of 3 dB corresponds to a halving of the squared error; a 6 dB difference corresponds to a quartering [S1, unofficial]. Target PSNR for NeRF Blender scenes is typically 31–33 dB [S2, unofficial].

**Edge Cases and Pitfalls**
- **Zero MSE**: If the prediction is perfect (MSE = 0), PSNR is infinite [S2, unofficial]. The `eps` term prevents this from causing a runtime error but results in a very large finite number [S3, unofficial].
- **Smooth Region Bias**: PSNR averages over every pixel, weighting large smooth regions heavily and ignoring small bright features like specular highlights or grass blades [S1, unofficial].
- **Noise Floor**: In scientific volumes, if the noise level is zero (signal-limited), MAD estimators may return exactly zero, bounding nothing [S7, unofficial].

## Subtopics

### [3D Evaluation Metrics](2903-3d-evaluation-metrics.md)
3D Evaluation Metrics quantify the fidelity of reconstructed volumetric or radiance-field representations against ground-truth references, addressing the problem of assessing both geometric accuracy and photometric consistency in 3D reconstruction. Unlike sibling methods that focus on specific aspects like mesh topology [Mesh Reconstruction Evaluation](2909-mesh-reconstruction-evaluation.md) or surface orientation [Normal Consistency](3050-normal-consistency.md), this category encompasses a suite of metrics including PSNR, SSIM, and mIoU to provide a comprehensive assessment of reconstruction quality. It distinguishes itself from [PSNR Metric Calculation](2742-psnr-metric-calculation.md) by extending the evaluation to 3D volumes and multi-view consistency, and from [Benchmark Performance Numbers](3471-benchmark-performance-numbers.md) by providing the specific algorithmic definitions of the scores rather than just reporting aggregated results.

### [Benchmark Performance Numbers](3471-benchmark-performance-numbers.md)
Benchmark Performance Numbers provide standardized quantitative baselines for evaluating radiance field optimization methods, primarily using the Mip-NeRF 360 dataset metrics (average PSNR and LPIPS). They serve as a reference point to compare the reconstruction quality of different variants (e.g., 3DGS, Mip-Splatting) against established baselines. Unlike raw metric calculation, these numbers aggregate results across a specific scene set to highlight trade-offs between visual fidelity, geometric accuracy, and resource efficiency.

### [PSNR Difference Metric for Ablation](5656-psnr-difference-metric-for-ablation.md)
The PSNR Difference Metric for Ablation quantifies the performance degradation caused by removing a specific component from a rendering pipeline by comparing the Peak Signal-to-Noise Ratio (PSNR) of the ablated configuration against the full method. It isolates the error ratio independent of the bit depth (MAX), as MAX cancels out in the logarithmic difference, providing a scale-invariant measure of quality loss.

### Noise Floor Estimation via MAD
Noise Floor Estimation via MAD estimates the noise standard deviation $\sigma$ in volumetric data to determine the theoretical maximum achievable PSNR (noise floor) [S7, unofficial]. It uses Median Absolute Deviation (MAD) estimators: (1) Laplacian MAD applies the D-dimensional discrete Laplacian and scales the MAD of the response by $1/(0.6745 \sqrt{2D(2D+1)})$ [S7, unofficial]; (2) Haar finite-difference MAD uses per-slice diagonal second differences $d = I(y,x) - I(y,x+1) - I(y+1,x) + I(y+1,x+1)$, scaling MAD by $1/(0.6745 \sqrt{2})$ [S7, unofficial]. The noise floor PSNR is then $10 \log_{10}(1/\sigma^2)$ [S7, unofficial]. A pitfall is that MAD estimators may return exactly zero for signal-limited volumes (e.g., Mouse heart nuclei), bounding nothing [S7, unofficial].

### PSNR Loss
PSNR Loss computes the Peak Signal-to-Noise Ratio between predicted and target images, used as a regression metric during training [S8, unofficial]. It is defined as $\text{PSNR} = 20 \log_{10}\left(\frac{\text{max\_val}}{\sqrt{\text{MSE}}}\right)$ [S8, unofficial]. Unlike the standard PSNR metric which is used for evaluation, this variant is often used as a differentiable loss function (or its negative) during optimization, with `max_val` defaulting to 1.0 [S8, unofficial].

### Color PSNR (YUV)
Color PSNR (YUV) computes color degradation by converting RGB to YUV and combining per-component PSNR values [S9]. Points in the distorted content are associated to reference points using a nearest neighbor algorithm, and RGB values are converted to YUV per ITU-R BT.709-3 [S9]. PSNR is computed for Y, U, and V components separately, then combined using a weighted average: $PSNRYUV = \frac{6 \cdot PSNRY + PSNRU + PSNRV}{8}$ [S9].

## Operating characteristics
- **Time Complexity**: Not given in the sources.
- **Memory Complexity**: Not given in the sources.
- **Strengths**: Simple to implement, computationally inexpensive, and widely understood as a standard benchmark for image quality [S2, unofficial][S4].
- **Limitations**: Does not correlate well with human perception for structural details or textures; heavily biased towards smooth regions; sensitive to global brightness shifts [S1, unofficial].
- **When to Use**: Standard reporting for novel view synthesis benchmarks (e.g., Mip-NeRF 360, Blender scenes) where photometric accuracy is the primary concern [S1, unofficial][S2, unofficial].
- **When Not to Use**: When structural similarity or perceptual quality is more important than pixel-wise accuracy; in such cases, SSIM or LPIPS should be used alongside or instead [S1, unofficial].

## Background
In the context of volume rendering and neural radiance fields (NeRF), PSNR became a standard metric for evaluating novel view synthesis quality, with early benchmarks targeting 31–33 dB on Blender scenes [S2, unofficial]. It is frequently reported alongside SSIM and LPIPS to provide a more complete picture of reconstruction fidelity, as PSNR alone fails to capture perceptual similarities in complex scenes [S1, unofficial].

## Sources
[S1] An Architectural Review of the 3D Gaussian Splatting Rendering Pipeline, https://doi.org/10.5281/zenodo.21739653, fetched 2026-10-26 (unofficial)
[S2] emmanueladutwum123/nerf-from-scratch, https://github.com/emmanueladutwum123/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S3] yashtmy/nerf-3d-vision, https://github.com/yashtmy/nerf-3d-vision, fetched 2026-10-26 (unofficial)
[S4] Discretized Gaussian Representation for Tomographic Reconstruction, https://doi.org/10.1109/ICCV51701.2025.02325, fetched 2026-10-27
[S5] ayush-chauhan-1202/physics-aware-ct-reconstruction, https://github.com/ayush-chauhan-1202/physics-aware-ct-reconstruction, fetched 2026-10-27 (unofficial)
[S6] andersblomqvist/unity-nanovdb-renderer, https://github.com/andersblomqvist/unity-nanovdb-renderer, fetched 2026-10-27 (unofficial)
[S7] Luxar: Gaussian splatting for microscopy and scalable interactive web visualisation of multidimensional scientific data, https://doi.org/10.64898/2026.09.22.753676, fetched 2026-10-26 (unofficial)
[S8] suvadityamuk/NeRF-Experiments, https://github.com/suvadityamuk/NeRF-Experiments, fetched 2026-10-27 (unofficial)
[S9] A novel methodology for quality assessment of voxelized point clouds, https://doi.org/10.1117/12.2322741, fetched 2026-10-30
