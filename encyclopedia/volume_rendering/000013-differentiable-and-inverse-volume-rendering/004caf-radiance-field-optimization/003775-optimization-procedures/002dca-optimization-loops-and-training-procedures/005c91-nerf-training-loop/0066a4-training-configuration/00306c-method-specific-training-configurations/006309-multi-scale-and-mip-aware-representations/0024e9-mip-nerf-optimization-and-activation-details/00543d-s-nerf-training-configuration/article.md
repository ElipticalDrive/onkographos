# S-NeRF Training Configuration

Also called: S-NeRF Optimization Setup, S-NeRF Hyperparameters

## Overview
S-NeRF defines the specific hyperparameters, network architectures, and training schedules required to optimize neural radiance fields for dynamic street scenes using LiDAR and RGB data [S1, unofficial]. It distinguishes itself from standard NeRF variants by employing a coarse-fine training strategy with separate configurations for foreground vehicles and background street scenes, and by integrating confidence maps derived from multi-view geometric consistency to mitigate depth outliers [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Sparse LiDAR points (consecutive frames $t-1, t, t+1$), RGB images, camera poses, and intrinsics.
*   **Outputs:** Optimized MLP weights for density and color fields, and confidence maps for filtering outliers [S1, unofficial].

**Data Structures:**
The method uses two distinct MLP architectures depending on the scene component [S1, unofficial]:
1.  **Foreground Vehicle MLP:** 4 layers, 256 hidden units per layer.
2.  **Background Density MLP:** 8 layers, 1024 hidden units per layer.
3.  **Background Color MLP:** 3 layers, 128 hidden units per layer.
*Note: The background coarse and fine networks share parameters [S1, unofficial].*

**Training Procedure:**
The training is split into foreground and background phases [S1, unofficial].

1.  **Foreground Vehicle Training:**
    *   **Sampling:** 64 samples per ray [S1, unofficial].
    *   **Iterations:** 30,000 [S1, unofficial].
    *   **Optimizer:** Adam [S1, unofficial].
    *   **Learning Rate:** $5 \times 10^{-4}$ [S1, unofficial].
    *   **Batch Size:** 1024 [S1, unofficial].
    *   **Loss Weights:** Depth loss weight $\lambda_1 = 1$, Smooth loss weight $\lambda_2 = 0.15$ [S1, unofficial].
    *   **Hardware:** Single RTX 3090 GPU [S1, unofficial].

2.  **Background Street Scene Training:**
    *   **Scene Parameterization:** Radius $r = 3$ [S1, unofficial].
    *   **Sampling:** 128 samples per ray in log scale [S1, unofficial].
    *   **Iterations:** 100,000 [S1, unofficial].
    *   **Optimizer:** Adam [S1, unofficial].
    *   **Learning Rate Schedule:** Log-linear decay from $5 \times 10^{-4}$ to $5 \times 10^{-6}$ with a warm-up phase of 2,500 iterations [S1, unofficial].
    *   **Batch Size:** 2048 [S1, unofficial].
    *   **Loss Weights:** Depth loss weight $\lambda_1 = 0.2$, Smooth loss weight $\lambda_2 = 0.01$ [S1, unofficial].
    *   **Confidence Threshold:** $\tau = 20\%$ for confidence measurement [S1, unofficial].
    *   **Hardware:** Two RTX 3090 GPUs [S1, unofficial].

**Confidence Map Computation (Minor Variant):**
To reduce the influence of depth outliers, S-NeRF computes geometry and projection confidence maps [S1, unofficial].
1.  **Reprojection:** Project 3D LiDAR points from time $t-1$ and $t+1$ to time $t$ using the displacement $\Delta P$ between frames [S1, unofficial].
2.  **Consistency Check:** Remove moving object outliers using optical flow and depth projection consistency [S1, unofficial].
3.  **Feature Consistency:** Compute RGB, SSIM, and VGG confidence maps to measure photometric and feature consistency [S1, unofficial].
4.  **Aggregation:** Combine depth, flow, RGB, SSIM, and VGG components via learnable aggregation to produce final confidence maps [S1, unofficial].

**Parameters Summary:**
*   Foreground samples: 64 [S1, unofficial]
*   Background samples: 128 [S1, unofficial]
*   Foreground iterations: 30,000 [S1, unofficial]
*   Background iterations: 100,000 [S1, unofficial]
*   Foreground LR: $5 \times 10^{-4}$ [S1, unofficial]
*   Background LR: $5 \times 10^{-4}$ to $5 \times 10^{-6}$ [S1, unofficial]
*   Background Batch Size: 2048 [S1, unofficial]
*   Foreground Loss $\lambda_1$: 1 [S1, unofficial]
*   Foreground Loss $\lambda_2$: 0.15 [S1, unofficial]
*   Background Loss $\lambda_1$: 0.2 [S1, unofficial]
*   Background Loss $\lambda_2$: 0.01 [S1, unofficial]
*   Confidence Threshold $\tau$: 20% [S1, unofficial]
*   Scene Radius $r$: 3 [S1, unofficial]

## Subtopics

### [Confidence Updating with Detached Color Network](16065-confidence-updating-with-detached-color-network.md)
This method distinguishes inlier and outlier camera poses by using a detached color network that omits the viewing direction as input, minimizing the impact of shape-radiance ambiguities that cause high PSNR for wrong poses. It solves the issue where outliers might have higher PSNR than inliers due to shape-radiance ambiguity, which traditional rendering metrics fail to detect. Unlike previous works using rendering metrics directly, this approach uses a detached network to provide reliable confidence scores for pose filtering.

## Operating characteristics
*   **Training Time:** ~2 hours per vehicle on a single RTX 3090 GPU; ~17 hours per background scene (250 images, 1280×1920) on two RTX 3090 GPUs [S1, unofficial].
*   **Parameter Count:** 8.76M learnable parameters, similar to other SOTA methods (8.7–9.9M) [S1, unofficial].
*   **Performance:** On Waymo dataset, S-NeRF achieves 23.60 PSNR and 0.743 SSIM, outperforming Mip-NeRF 360 (22.10 PSNR, 0.724 SSIM) [S1, unofficial].
*   **Scaling:** Training time scales with scene complexity and number of images; background scenes require more iterations and larger batch sizes than foreground vehicles [S1, unofficial].
*   **Limitations:** Training is slower than Mip-NeRF 360 due to the requirement of coarse rendering for supervision, though this can be accelerated by removing coarse supervision [S1, unofficial].

## Background
S-NeRF was introduced in the paper "S-NeRF: Neural Radiance Fields for Street Views" published at ICLR 2023 [S1, unofficial]. It builds on the Mip-NeRF optimization and activation details [3409-mip-nerf-optimization-and-activation-details.md] to handle dynamic street scenes with LiDAR data. The method was developed to address the challenges of reconstructing large-scale urban environments with moving objects and depth outliers.

## Sources
[S1] S-NeRF: Neural Radiance Fields for Street Views, https://doi.org/10.48550/arxiv.2303.00749, fetched 2026-10-29 (unofficial)
