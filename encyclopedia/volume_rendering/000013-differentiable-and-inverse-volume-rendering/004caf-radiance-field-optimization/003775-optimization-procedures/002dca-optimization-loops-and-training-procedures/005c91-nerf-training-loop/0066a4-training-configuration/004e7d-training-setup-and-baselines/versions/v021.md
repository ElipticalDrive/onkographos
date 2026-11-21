# Training Setup and Baselines


## Overview
This family defines the reference training protocols for differentiable volume rendering, establishing the hyperparameters, optimization strategies, and architectural configurations required to train neural radiance fields. It serves as the foundational comparison point for novel methods, ensuring that performance gains are attributable to architectural innovations rather than training advantages. The family encompasses standard NeRF baselines, HDR-aware training, GAN-based volumetric generation, and sparse volume representations.

## Shared principles
All methods in this family share the core differentiable volume rendering pipeline defined in [NeRF Training Loop](1522-training-loop-setup.md). This involves:
1.  **Ray Sampling:** Generating rays from camera poses and sampling points along them.
2.  **Network Query:** Evaluating a neural network (MLP or grid-based) at sample points to obtain volume density $\sigma$ and color $\mathbf{c}$.
3.  **Volume Rendering:** Computing the predicted pixel color $\hat{\mathbf{C}}(\mathbf{r})$ using the discretized integral:
    $$ \hat{\mathbf{C}}(\mathbf{r}) = \sum_{i=1}^{N} T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i $$
    where $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j)$ is the accumulated transmittance and $\delta_i$ is the distance between samples [S1, unofficial][S2].
4.  **Loss Computation:** Minimizing the difference between rendered and ground truth images, typically using Mean Squared Error (MSE) [S1, unofficial][S2].
5.  **Optimization:** Updating network parameters using gradient-based optimization (e.g., Adam, RMSprop) [S1, unofficial][S2].

## Subtopics

### [NeRF-GT Baseline Training Setup](11349-nerf-gt-baseline-training-setup.md)
Establishes a reference protocol for NeRF using HDR inputs, serving as an upper-bound for methods synthesizing HDR from LDR. It differs from standard NeRF by explicitly handling dynamic range mismatch via tone-mapping during loss computation. Choose this when evaluating HDR reconstruction capabilities.

### [StyleGAN2 Training Configuration](12972-stylegan2-training-configuration.md)
Defines the training protocol for StyleGAN2-based radiance fields (e.g., StyleNeRF, VoxGRAF). It solves GAN instability in 3D by enforcing R1 regularization, specific learning rate scaling, and removing 2D noise injection to preserve 3D consistency. Choose this for generative 3D-aware models.

### [NeRF Training Setup](13271-nerf-training-setup.md)
Describes the configuration for NeRF-based models using Nerfstudio/Instant-NGP, focusing on fast convergence for static scenes. It details Control-NeRF's use of feature volumes instead of raw coordinates for the fine network. Choose this for efficient static scene reconstruction.

### [NLR Baseline](15254-nlr-baseline.md)
Implements the Neural Lumigraph Representation (NLR) as a foundational reference for differentiable volume rendering. It differs from standard NeRF by using an architecture optimized for light fields rather than a static MLP. Choose this for meta-learning or light field baselines.

### NeuralVDB Hyperparameter Configuration
Configures training for sparse volume representations using hierarchical neural networks. It addresses the trade-off between noise reduction and computational efficiency in VDB-based rendering. Choose this when high-resolution sparse volume reconstruction is required.
*   **Step:** Batch sampling strategy is either drawing $2^{16}$ random samples per epoch, or resampling a subset of input voxels at fixed intervals (100 or 500) and drawing $2^{12}$ batches per epoch. The resampled subset size equals the resampling intervals multiplied by the batch size. Resampling is used when fine details with thin structures are critical [S1, unofficial].
*   **Parameter:** Subdomain size must be a multiple of 512. Too large wastes compute on empty subdomains; too small increases dispatch overhead and causes dominant overlapping halo regions leading to redundant computation [S1, unofficial]. Network width: Level-1 networks use half the width of level-0 networks. Default starting point: level-0 width 128, depth 3. Use 4 layers for volumes requiring more detail capture [S1, unofficial]. Optimizer: Adam with default learning rate 0.001. Decay: 0.975 every 100 epochs (no resampling) or 0.75 every 1,000 epochs (with resampling). Max epochs: 2,500 for non-resampled; more for resampled cases [S1, unofficial].
*   **Comparison:** Dense grid training produces the highest noise. Block grid (dense leaf nodes without active masks) reduces noise but still shows visible fog artifacts. Sparse VDB with active voxel masks yields the best visual quality and low noise reconstruction [S1, unofficial].

### Neural Codebook Training Configuration
Configures training for compact real-time radiance fields using a Neural Codebook (NCB). It focuses on efficient storage and fast inference through codebook quantization. Choose this for real-time applications requiring compact representation.
*   **Step:** Multiply learning rate by 0.3 for every 5,000 iterations. Cast the trained Neural Codebook (NCB) to float16 after training [S2].
*   **Parameter:** Optimizer: Adam with $\beta_1 = 0.9$ and $\beta_2 = 0.999$. Weight decay: $1e-5$. Initial learning rate: $5e-3$. Total training iterations: 20,000. Batch size: 100,000 voxels [S2].

## Background
The concept of training neural radiance fields was introduced by Mildenhall et al. in 2020 with the original NeRF paper [S1, unofficial]. Subsequent work expanded the baseline configurations to include HDR handling [S1, unofficial], GAN-based generation , and sparse volume representations like NeuralVDB [S1, unofficial] and Neural Codebooks [S2]. These baselines are critical for benchmarking new methods in the field of differentiable volume rendering.

## Sources
[S1] NeuralVDB: High-resolution Sparse Volume Representation using Hierarchical Neural Networks | NVIDIA, https://www.youtube.com/watch?v=IfgeduMv_ns, fetched 2026-11-04 (unofficial)
[S2] Compact Real-Time Radiance Fields with Neural Codebook, https://doi.org/10.1109/icme55011.2023.00374, fetched 2026-10-26
[S3] Unofficial notes on StyleGAN2 and 3D consistency, fetched 2026-11-04 [S3, unofficial]
