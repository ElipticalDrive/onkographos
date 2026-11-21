# 3DGS Model Compression Pipeline

Also called: Hardware-Friendly Compression, Iterative Pruning & VQ Compression

## Overview
A compression scheme tailored for hardware deployment that reduces model size and bandwidth by iteratively pruning Gaussians, reducing spherical harmonics (SH) degree, and quantizing attributes [S1]. It modifies LightGaussian to achieve higher compression ratios (51.6× vs 15×) by adding iterative pruning, progressive SH reduction, and full VQ on all coefficients/colors [S1].

## How it works
**Inputs:** Pre-trained 3D Gaussian Splatting model (Gaussians with positions, covariances, opacities, colors, and SH coefficients).
**Outputs:** Compressed model with reduced Gaussian count, lower SH degree, and quantized attributes.

**Data Structures:**
*   **Gaussian Set:** List of $N$ Gaussians. Each Gaussian $i$ has:
    *   Mean position $\mu_i \in \mathbb{R}^3$
    *   Covariance $\Sigma_i \in \mathbb{R}^{3 \times 3}$ (or rotation/scale representation)
    *   Opacity $\alpha_i \in [0, 1]$
    *   Color $c_i$ (encoded as SH coefficients)
*   **SH Coefficients:** Stored as vectors of coefficients for each RGB channel.

**Steps:**
1.  **Iterative Gaussian Pruning:**
    *   Remove Gaussians that contribute least to the visual fidelity. This is done iteratively to reach a target compression ratio or sparsity level [S1].
    *   *Note:* The specific metric for "least contribution" (e.g., gradient magnitude, opacity, or rendering error) is not explicitly detailed in the provided sources, but it is implied to be a refinement of the pruning strategy in LightGaussian [S1].
2.  **Progressive Spherical Harmonics (SH) Degree Reduction:**
    *   Reduce the degree of the spherical harmonics used to encode color and directional information. This reduces the number of coefficients per Gaussian [S1].
    *   *Note:* The specific schedule for degree reduction (e.g., from degree 3 to degree 1) is not explicitly detailed, but it is a key step in the pipeline [S1].
3.  **Vector Quantization (VQ):**
    *   Apply VQ to **all** SH coefficients and colors [S1].
    *   This involves clustering the coefficients/colors into a codebook and replacing each coefficient/color with an index to the codebook entry [S1].
    *   *Note:* The size of the codebook and the specific VQ algorithm (e.g., k-means, LBG) are not explicitly detailed in the provided sources [S1].

**Parameters:**
*   **Target Compression Ratio:** 51.6× model-size reduction [S1].
*   **PSNR Loss:** 0.743 dB [S1].

**Edge Cases and Pitfalls:**
*   **View-Dependent Culling:** Approximately 42% of Gaussian points are considered unnecessary via near-plane culling, but this percentage is view-dependent. When zooming out, nearly all Gaussians can become visible, decreasing the culling benefit [S1].
*   **Alpha-Blending Contribution:** Only about 50% of Gaussian points contribute to the pixel color during $\alpha$-blending [S1].

## Subtopics

## Operating characteristics
*   **Compression Ratio:** 51.6× model-size reduction [S1].
*   **Quality Loss:** 0.743 dB PSNR loss [S1].
*   **Comparison:** Modifies LightGaussian (which achieves 15× reduction) to reach 51.6× by adding iterative pruning, progressive SH reduction, and full VQ on all coefficients/colors [S1].
*   **Hardware Efficiency:** The compressed model enables a hardware accelerator to deliver 1219 Mpixels/J at 267.5 Mpixels/s, enabling 1080p at 129 FPS [S1].
*   **Memory Usage:** Unbounded 360 degree scenes typically require millions of Gaussian points, leading to storage requirements in the gigabyte range. This pipeline mitigates this [S1].

## Background
The 3DGS algorithm is not hardware-friendly and faces four main design challenges, including substantial memory usage [S1]. To overcome these limitations, a dedicated hardware accelerator for real-time rendering of 3DGS models on portable AR/VR devices was introduced, targeting full-HD resolution at 120 FPS per eye [S1]. The compression pipeline is a key component of this accelerator, modifying the lightweight model, LightGaussian, to higher compression ratios [S1].

## Sources
[S1] A 129 FPS Full HD Real-Time Accelerator for 3D Gaussian Splatting., https://doi.org/10.1109/tvcg.2026.3683714, fetched 2026-10-27
