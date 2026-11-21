# Decoupled Neural Fields

Also called: Decoupled geometry-appearance network, WildSplat Network Architecture

## Overview
Decoupled Neural Fields is a feedforward neural network architecture that predicts pixel-aligned 3D Gaussians from unposed images by separating the representation of geometry and appearance into distinct branches [S1]. Unlike single-network baselines that predict all Gaussian parameters simultaneously, this decoupled approach prevents the interference between geometric and appearance features, thereby avoiding rendering artifacts such as gaps and holes [S2]. It builds on [Neural Network and Feed-Forward Approaches](9526-neural-network-and-feed-forward-approaches.md) by providing a specialized initialization and parameter prediction mechanism for Gaussian Splatting.

## How it works
**Inputs and Outputs:**
*   **Input:** Unposed images of a scene, partitioned into context views (for encoding) and target views (for supervision).
*   **Output:** A set of pixel-aligned 3D Gaussians, each defined by position, covariance, opacity, and view-dependent color.

**Data Structures:**
The architecture consists of two main branches:
1.  **Geometry Branch:**
    *   **Encoder:** DINOv2-large [S1].
        *   Embedding dimension: $D = 1024$ [S1].
    *   **Transformer:** $L = 24$ alternating-attention layers [S1].
2.  **Appearance Branch:**
    *   **Extractor:** Pretrained DINOv2-base model [S1].
    *   **Injector:** Four identical blocks. Each block contains:
        *   AdaLN-Zero modulation layer [S1].
        *   Cross-attention layer [S1].
        *   Self-attention layer [S1].
        *   MLP [S1].

**Parameters:**
*   Spherical Harmonics degree for view-dependent colors: $k = 4$ [S1].
*   Optimizer: Adam with $\beta_1 = 0.9, \beta_2 = 0.95$ [S1].
*   Learning Rate: $2 \times 10^{-5}$ for network backbones, $2 \times 10^{-4}$ for DPT heads [S1].
*   Loss weights: $\lambda_1 = 10.0$ (pose distillation), $\lambda_2 = 10.0$ (MSE), $\lambda_3 = 1.0$ (SSIM), $\lambda_4 = 0.05$ (LPIPS) [S1].
*   Hardware: Optimized across four NVIDIA RTX H20 GPUs [S1].

**Steps:**
1.  **Geometry Encoding:** Pass context images through the DINOv2-large encoder to obtain features with dimension $D=1024$ [S1].
2.  **Geometry Refinement:** Process the encoded features through $L=24$ alternating-attention layers to refine geometric representations [S1].
3.  **Appearance Extraction:** Pass context images through the pretrained DINOv2-base model to extract appearance features [S1].
4.  **Appearance Injection:** Combine geometry and appearance features using the Appearance Injector. For each of the four blocks:
    a. Apply AdaLN-Zero modulation to condition the features on the geometry [S1].
    b. Apply cross-attention to align appearance features with geometry [S1].
    c. Apply self-attention to refine appearance features [S1].
    d. Pass through an MLP [S1].
5.  **Gaussian Prediction:** Predict the parameters for pixel-aligned 3D Gaussians. View-dependent colors are parameterized using spherical harmonics of degree $k=4$ [S1].
6.  **Optimization:** Optimize all network parameters except those in the geometry encoder (DINOv2-large) [S1].
7.  **Merging:** Employ a voxel-based merging strategy to aggregate spatially redundant Gaussians during training and inference to optimize memory consumption [S1].

**Pitfalls:**
*   Using a single neural field to concurrently represent geometric and appearance properties causes consistent degradation across all image quality metrics, resulting in compromised rendering fidelity and inaccurate geometry such as gaps and holes [S2].
*   The single neural field baseline requires a maximum hash size of $2^{18}$ to maintain experimental fairness, which leads to a slightly larger parameter count than the decoupled architecture [S2].

## Operating characteristics
*   **Memory:** Employs a voxel-based merging strategy to aggregate spatially redundant Gaussians, optimizing memory consumption during training and inference [S1].
*   **Performance:** The decoupled approach avoids the degradation seen in single-field baselines, maintaining superior visual fidelity [S2].
*   **Hardware:** Implemented and optimized on four NVIDIA RTX H20 GPUs [S1].
*   **Strengths:** Prevents interference between geometry and appearance, leading to accurate geometry without gaps or holes [S2].
*   **Limitations:** Requires significant computational resources for the dual-branch architecture and multiple GPUs for training [S1].

## Background
Decoupled Neural Fields were introduced in the WildSplat paper as a method for feedforward Gaussian Splatting from unposed in-the-wild images [S1]. The architecture was designed to address the limitations of single-network approaches in predicting 3D Gaussians, specifically the trade-off between geometric accuracy and appearance quality [S2].

## Sources
[S1] WildSplat: Feedforward Gaussian Splatting from Unposed In-the-Wild Images, https://doi.org/10.1007/978-3-032-37324-3_2, fetched 2026-10-29
[S2] HyRF: Hybrid Radiance Fields for Memory-efficient and High-quality Novel View Synthesis, https://doi.org/10.52202/085713-0617, fetched 2026-10-30
