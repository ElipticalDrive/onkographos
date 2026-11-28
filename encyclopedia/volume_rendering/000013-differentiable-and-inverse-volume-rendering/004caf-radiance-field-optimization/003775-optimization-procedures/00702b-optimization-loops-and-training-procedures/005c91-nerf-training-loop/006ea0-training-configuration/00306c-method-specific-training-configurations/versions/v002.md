# Method-Specific Training Configurations


## Overview
This family defines the hyperparameters, optimization routines, and architectural constraints required to train specific differentiable volume rendering methods. It solves the problem of translating theoretical radiance field formulations into stable, convergent training procedures by specifying exact network structures, sampling strategies, and loss functions. Unlike generic [Training Configuration](2332-training-configuration.md) guidelines, these configurations are method-specific, addressing unique failure modes such as negative density gradients in [Mip-NeRF](3409-mip-nerf-optimization-and-activation-details.md) or the computational cost of dense sampling in [DONeRF](4461-donerf.md).

## Shared principles
All methods in this family optimize a neural representation of a scene by minimizing a photometric loss between rendered and ground-truth images. They share the following core mechanisms:
1.  **Ray Sampling**: Rays are generated from camera poses and sampled at discrete points along their paths. The density $\sigma$ and color $c$ are queried at these points.
2.  **Volume Rendering**: The final pixel color is computed via differentiable volume rendering, integrating density and color along the ray using transmittance $T$ and alpha $\alpha$ [S1, S2, S3].
3.  **Optimization**: Parameters are updated using gradient descent, typically Adam, minimizing the Mean Squared Error (MSE) between rendered and target colors [S1, S2, S3].

## Subtopics

### [NeRF Hyperparameters](3065-nerf-hyperparameters.md)
Defines the baseline configuration for the original NeRF method. It uses a single-pass stratified sampling approach (or simplified two-stage) with fixed sample counts, optimized for synthetic scenes like Blender. It relies on standard positional encoding and a single MLP architecture. Choose this for standard differentiable volume rendering baselines.

### [Mip-NeRF Optimization and Activation Details](3409-mip-nerf-optimization-and-activation-details.md)
Specifies training details for mip-NeRF and JaxNeRF, focusing on activation functions and learning rate schedules to improve stability. It differs from standard NeRF by removing identity concatenation, using shifted softplus for density, and employing a specific warmup/annealing schedule. Choose this to prevent optimization failure modes like zero gradients from negative density or color saturation.

### [DONeRF](4461-donerf.md)
A training configuration for compact neural radiance fields that replaces dense raymarching with a two-network pipeline: a depth oracle and a shading network. It uses a non-end-to-end, two-stage training procedure and relies on a classification-based depth oracle. Choose this when computational cost is prohibitive and high-quality rendering with few samples (2–4) is required.

### NeUDF Training Configuration
*   **Optimizer**: ADAM [S1].
*   **Learning Rate**: Warms up to $2 \times 10^{-4}$ in the first 5k iterations, then decreases to $1 \times 10^{-5}$ by the end [S1].
*   **Batch Size**: 512 random rays sampled from 8 randomly selected camera poses per iteration [S1].
*   **Training Duration**: 400k iterations total. Takes 9 hours with mask, 11 hours without mask, on a single Nvidia 3090 GPU [S1].

### NeRF-SDP Training Configuration
*   **Optimizer**: Adam [S2].
*   **Learning Rate**: $1 \times 10^{-3}$ for Res-UNet, $5 \times 10^{-4}$ for other parts [S2].
*   **Batch Size**: 500 [S2].
*   **Hardware**: Nvidia GeForce RTX 3090 [S2].
*   **Total Steps**: 250,000. First 170,000 steps optimize auxiliary and rendering losses. Last 80,000 steps optimize only rendering loss [S2].
*   **Feature Dimensions**: 2D features 24, 3D features 8 [S2].
*   **Datasets**: LLFF, IBRNet, Google Scanned [S2].

## Background
The original NeRF hyperparameters were established in the seminal NeRF paper [S3, S4, unofficial]. Mip-NeRF details were introduced in the Mip-NeRF paper [S1]. DONeRF was proposed to address computational efficiency in neural radiance fields [S2]. NeUDF and NeRF-SDP configurations are derived from their respective publications [S1, S2].

## Sources
[S1] NeUDF: Leaning Neural Unsigned Distance Fields with Volume Rendering, https://doi.org/10.1109/cvpr52729.2023.00031, fetched 2026-10-26
[S2] NeRF-SDP: Efficient Generalizable Neural Radiance Field with Scene Depth Perception, https://doi.org/10.1145/3595916.3626380, fetched 2026-10-30
[S3] Neural Radiance Fields for Real-Time View Synthesis in Non-Line-of-Sight Imaging, https://arxiv.org/abs/2003.08934, fetched 2026-10-26
[S4] Official NeRF Implementation, https://github.com/bmild/nerf, fetched 2026-10-26
