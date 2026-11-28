# Accelerated Training for 3DGS


## Overview
A strategy to improve training time efficiency by separating higher spherical harmonic (SH) bands from the 0th band within the rasterizer, reducing the number of updates for higher SH bands compared to diffused colors [S1]. It sets itself apart from standard 3D Gaussian Splatting by decoupling the update frequency of view-dependent color variations from the base color, and from [FastGS](9177-fastgs.md) by focusing on rasterizer-level optimization rather than gradient checkpointing or dynamic sparse rasterization.

## How it works
**Inputs and Outputs:**
*   **Input:** 3D Gaussian parameters (position, covariance, opacity, SH coefficients), camera parameters, scene images.
*   **Output:** Optimized 3D Gaussian parameters.

**Data Structures:**
*   **Gaussian Attributes:** Each Gaussian stores a 0th-order SH coefficient (diffused color, 3 values) and higher-order SH coefficients (view-dependent color, 45 values for degree 2) [S1].
*   **Rasterizer State:** Maintains separate accumulation buffers or update flags for diffused color and higher SH bands.

**Steps:**
1.  **Rasterization Split:** Modify the rasterizer to split the SH bands from the diffused color [S1]. The 0th band is treated as the diffused color, while bands 1–4 (45 dimensions) are treated separately [S1].
2.  **Update Frequency Control:**
    *   Update the diffused color (0th SH band) at every training step [S1].
    *   Update the higher SH bands (45 dimensions) every 16 iterations [S1]. This reduces the computational load associated with view-dependent color variations, which constitute a large proportion of the updates [S1].
3.  **SSIM Loss Optimization:** Replace the standard 11x11 2D Gaussian kernel used in SSIM calculation with two smaller 1D Gaussian kernels [S1].
4.  **Fused CUDA Kernel:** Calculate the optimized SSIM metric using a fused CUDA kernel that performs the 1D convolutions efficiently [S1].
5.  **Loss Calculation:** Compute the total loss $L$ using the formula:
    $$ L = (1 - \lambda_{SSIM})L_1 + \lambda_{SSIM} L_{D-SIM} + \lambda_m L_m + \lambda_{sh} L_{sh} $$
    where $L_m$ and $L_{sh}$ are defined in papers [11] and [13] respectively [S1].

**Parameters:**
*   **SH Update Interval:** 16 iterations [S1].
*   **SSIM Kernel:** 1D Gaussian kernels replacing the standard 11x11 2D kernel [S1].
*   **Loss Weights:** $\lambda_{SSIM}$, $\lambda_m$, $\lambda_{sh}$ are standard hyperparameters [S1].

**Edge Cases and Pitfalls:**
*   **Precision:** The separation of SH bands requires careful handling of the rasterizer's accumulation logic to ensure view-dependent effects are correctly applied only at the specified interval [G].
*   **Degenerate Input:** If the scene has no view-dependent effects, the higher SH bands may remain static, potentially leading to suboptimal representation if the update interval is too long [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_step(gaussians, cameras, images, step):
    # 1. Rasterization with split SH bands
    # Diffused color (0th band) is updated every step
    # Higher SH bands (45 dims) are updated every 16 steps
    if step % 16 == 0:
        # Update all SH bands
        sh_coeffs = update_sh_bands(gaussians.sh_coeffs)
    else:
        # Only update 0th band (diffused color)
        sh_coeffs = update_0th_band(gaussians.sh_coeffs)
    
    # Render image with current SH coefficients
    rendered_image = rasterize(gaussians, cameras, sh_coeffs)
    
    # 2. Optimized SSIM Loss
    # Replace 11x11 2D kernel with two 1D kernels
    ssim_loss = calculate_optimized_ssim(rendered_image, images[0])
    
    # 3. Total Loss
    l1_loss = calculate_l1(rendered_image, images[0])
    # L_m and L_sh are defined in [11] and [13]
    total_loss = (1 - lambda_ssim) * l1_loss + lambda_ssim * ssim_loss + lambda_m * L_m + lambda_sh * L_sh
    
    # 4. Backpropagation
    total_loss.backward()
```

## Subtopics

### Local-GS Integration
Local-GS is a variant that integrates tile-local warp coherence to accelerate 3D Gaussian Splatting [S2, unofficial]. It reduces over 60% of kernel latency by minimizing global memory access through warp-level Gaussian culling and cooperative tile-level hoisting [S2, unofficial]. On the Deep Blending dataset, Local-GS boosts throughput from 206.37 FPS (vanilla 3DGS) to 1168.88 FPS, a 5.66x speedup [S2, unofficial]. In the vanilla 3DGS cohort, it yields speedups from 2.58x to 3.05x across 13 scenes [S2, unofficial]. For example, on the bicycle scene, kernel time drops from 3.58 ms to 1.24 ms (2.89x speedup) [S2, unofficial]. On the drjohnson scene, kernel time reduces from 3.39 ms to 1.11 ms (3.05x speedup) [S2, unofficial]. On the playroom scene, kernel time reduces from 2.40 ms to 0.79 ms (3.05x speedup) [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Reduced by updating higher SH bands less frequently (every 16 iterations) [S1].
*   **Memory Complexity:** No significant change in memory usage, as the SH coefficients are still stored [G].
*   **Measured Performance:** Not given in the sources for the base method. Local-GS integration shows significant speedups [S2, unofficial].
*   **Strengths:** Improves training time efficiency by reducing the computational load of view-dependent color updates [S1].
*   **Limitations:** May not capture rapid view-dependent changes if the update interval is too long [G].
*   **When to use:** When training time is a bottleneck and view-dependent effects are not extremely high-frequency [G].
*   **When not to use:** When high-fidelity view-dependent effects are critical and cannot tolerate the 16-iteration delay [G].

## Background
Originated from Trick-GS, which proposes a balanced bag of tricks for efficient Gaussian Splatting [S1]. The method focuses on training time efficiency by controlling the low-pass filter in the rasterization stage [S1]. It builds on the observation that higher SH bands cover a large proportion of updates and are only used to represent view-dependent color variations [S1].

## Sources
[S1] Trick-GS: A Balanced Bag of Tricks for Efficient Gaussian Splatting, https://doi.org/10.1109/icassp49660.2025.10889395, fetched 2026-10-30
[S2] Local-GS: Accelerating 3D Gaussian Splatting via Tile-Local Warp Coherence, https://doi.org/10.48550/arXiv.2606.16566, fetched 2026-10-29 (unofficial)
