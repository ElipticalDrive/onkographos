# 3DGS Model Compression Pipeline

Also called: Hardware-Friendly Compression, Iterative Pruning & VQ Compression

## Overview
A compression scheme tailored for hardware deployment that reduces model size and bandwidth by iteratively pruning Gaussians, reducing spherical harmonics (SH) degree, and quantizing attributes [S1]. It modifies LightGaussian to achieve significantly higher compression ratios (51.6× vs 15×) by adding iterative pruning, progressive SH reduction, and full vector quantization (VQ) on all coefficients and colors [S1].

## How it works
The pipeline consists of three sequential stages designed to minimize memory footprint and computational complexity for real-time hardware rendering [S1].

**Inputs:** A trained 3D Gaussian Splatting model (positions, covariances, opacities, SH coefficients, colors) [S1].
**Outputs:** A compressed model with reduced Gaussian count, lower SH degree, and quantized attributes [S1].

**Data Structures:**
*   **Gaussian Set:** A list of $N$ Gaussians. Each Gaussian $i$ retains position $\mathbf{p}_i$, covariance $\mathbf{\Sigma}_i$, opacity $\alpha_i$, and reduced SH coefficients $\mathbf{c}_i$ [S1].
*   **Quantization Map:** A codebook for VQ mapping quantized indices to representative values [S1].

**Steps:**

1.  **Iterative Gaussian Pruning:**
    *   Evaluate the contribution of each Gaussian to the rendered image.
    *   Remove Gaussians with negligible contribution or high redundancy.
    *   Fine-tune the remaining Gaussians to recover rendering quality [S1].
    *   *Note:* The specific metric for "negligible contribution" (e.g., gradient magnitude, opacity threshold) is not given in the sources [G].

2.  **Progressive Spherical Harmonics (SH) Degree Reduction:**
    *   Reduce the SH degree of the remaining Gaussians.
    *   This step reduces the dimensionality of the color representation per Gaussian [S1].
    *   *Note:* The specific schedule for degree reduction (e.g., from degree 3 to degree 1) is not given in the sources [G].

3.  **Vector Quantization (VQ):**
    *   Apply VQ to **all** SH coefficients and colors.
    *   Replace continuous coefficient values with discrete indices from a codebook.
    *   This step drastically reduces the storage size of the attribute data [S1].

**Parameters:**
*   **Target Compression Ratio:** 51.6× model-size reduction [S1].
*   **Acceptable PSNR Loss:** 0.743 dB [S1].

**Edge Cases and Pitfalls:**
*   **View-Dependent Culling:** Approximately 42% of Gaussian points are unnecessary via near-plane culling, but this percentage is view-dependent. When zooming out, nearly all Gaussians can become visible, decreasing culling benefits [S1].
*   **Alpha-Blending Contribution:** Only about 50% of Gaussian points contribute to the final pixel color during $\alpha$-blending [S1].

## Subtopics

## Operating characteristics
*   **Compression Ratio:** 51.6× model-size reduction [S1].
*   **Quality Loss:** 0.743 dB PSNR loss [S1].
*   **Comparison:** Modifies LightGaussian (15× reduction) to reach 51.6× by adding iterative pruning, progressive SH reduction, and full VQ on all coefficients/colors [S1].
*   **Hardware Context:** Designed for a TSMC 28-nm process accelerator at 800 MHz, occupying 0.66 mm² with 1.1438 M gates and 120 kB SRAM, consuming 0.219 W, and delivering 1219 Mpixels/J at 267.5 Mpixels/s, enabling 1080p at 129 FPS [S1].
*   **Computational Savings:** Skips zero-Jacobian matrix multiplications in the projection path, reducing processing elements by 63% and computation by 53% [S1].

## Background
The method was introduced to overcome the limitations of standard 3DGS, which is not hardware-friendly due to substantial memory usage (gigabyte range for unbounded scenes) and inefficient computation (accessing unnecessary points) [S1]. It builds on LightGaussian [S1].

## Sources
[S1] A 129 FPS Full HD Real-Time Accelerator for 3D Gaussian Splatting., https://doi.org/10.1109/tvcg.2026.3683714, fetched 2026-10-27
