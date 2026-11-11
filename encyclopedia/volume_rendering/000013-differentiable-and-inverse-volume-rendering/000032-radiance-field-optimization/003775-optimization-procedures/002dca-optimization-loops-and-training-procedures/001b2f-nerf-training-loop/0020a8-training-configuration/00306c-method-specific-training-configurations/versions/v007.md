# Method-Specific Training Configurations


## Overview
Method-Specific Training Configurations define the precise hyperparameters, architectural constraints, and optimization schedules required to train specific Neural Radiance Field (NeRF) variants and related implicit surface representations. This family addresses the problem that standard NeRF configurations (e.g., basic MLPs with ReLU and MSE loss) often fail to converge, produce artifacts, or lack the fidelity required for specific datasets (synthetic vs. real-world) or architectures (hash grids, SDFs, depth-oracles). It sets itself apart from generic [Training Configuration](2332-training-configuration.md) by providing concrete, implementation-ready values for distinct methods such as Mip-NeRF, DONeRF, and Neuralangelo, rather than abstract principles.

## Shared principles
The articles in this family share the following mechanisms and terms:
*   **Optimizer:** All cited methods use variants of the Adam optimizer [S1, S2, S3, S7, S8, S9].
*   **Loss Function:** The primary loss is photometric, typically Mean Squared Error (MSE) or Charbonnier loss, comparing rendered pixel colors against ground truth [S1, S3, S7, S8].
*   **Sampling:** Training involves sampling rays from camera poses and querying the implicit field at sample points along these rays [S1, S2, S3, S4, S5, S6, S7, S8].
*   **Batching:** Training is performed in batches of rays or pixels [S1, S2, S3, S4, S5, S6, S7, S8].
*   **Learning Rate Scheduling:** All methods employ a learning rate schedule, often involving warmup and decay steps [S1, S2, S4, S6, S8].

## Subtopics

### [NeRF Hyperparameters](3065-nerf-hyperparameters.md)
Defines the baseline configuration for standard NeRF. It uses an 8-layer position MLP (256/512 units) and 4-layer direction MLP (128 units) with axis-aligned IPE. It employs hierarchical sampling (64 coarse, 128 fine) and Adam with an initial LR of $5 \times 10^{-4}$ decaying to $5 \times 10^{-6}$. Choose this for standard synthetic datasets like Blender when baseline performance is required.

### [Mip-NeRF Optimization and Activation Details](3409-mip-nerf-optimization-and-activation-details.md)
Modifies NeRF to handle anti-aliasing via integrated positional encoding (IPE) and robust activations. It replaces ReLU with shifted softplus for density and uses a widened sigmoid for color. It removes identity encoding from inputs and uses a specific warmup LR schedule. Choose this for unbounded scenes requiring anti-aliasing and stability.

### [DONeRF](4461-donerf.md)
Replaces dense raymarching with a two-network architecture: a depth oracle and a shading network. The oracle predicts high-importance sample locations using classification-based depth prediction, allowing the shading network to render with only 2–16 samples per ray. Choose this when computational cost is prohibitive and depth supervision (or initialization) is available.

### Blender Dataset Simplified Configuration
A simplified variant of Mip-NeRF for synthetic Blender datasets. It uses one round of sampling, 128 samples for the proposal MLP, and 32 samples for the NeRF MLP. The proposal MLP has 4 layers/256 units, and the NeRF MLP has 8 layers/256 (or 512) units. It uses axis-aligned IPE, MSE loss, and no distortion regularizer [S1].

### LLFF Training Configuration
A variant for the LLFF dataset. It differs from the Blender simplified config by keeping the distortion regularizer enabled and using Charbonnier loss instead of MSE. Performance metrics on LLFF with 256 hidden units show PSNR 26.68, SSIM 0.847, LPIPS 0.150, Time 2.39 hrs, Params 0.84M for the proposed model, compared to Mip-NeRF's PSNR 26.93, SSIM 0.830, LPIPS 0.177, Time 2.48 hrs, Params 0.61M [S1].

### NeUDF Training Configuration
Configures NeUDF for unsigned distance fields. It uses Adam with LR warming up to $2 \times 10^{-4}$ in the first 5k iterations and decreasing to $1 \times 10^{-5}$. Batch size is 512 rays from 8 random poses. Total training is 400k iterations (9-11 hours on RTX 3090) [S2].

### NeRF-SDP Training Configuration
Configures NeRF-SDP for generalizable NeRF with depth perception. It uses Adam with LR $1 \times 10^{-3}$ for Res-UNet and $5 \times 10^{-4}$ for other parts. Batch size is 500. Total steps are 250,000: first 170,000 optimize auxiliary and rendering losses, last 80,000 optimize only rendering loss. Feature dimensions are 24 (2D) and 8 (3D) [S3].

### Neuralangelo Training Configuration
Configures Neuralangelo for high-fidelity surface reconstruction using hash encoding. It assumes ROI inside a unit sphere. Total iterations are 500k. LR is $1 \times 10^{-3}$ with 5k warmup, decaying by 10x at 300k and 400k. Optimizer is AdamW with weight decay $10^{-2}$. SDF MLP has 1 layer, color MLP has 4 layers. Batch size is 1 for DTU, 16 for Tanks and Temples [S4].

### GO-NeRF Training Configuration
Configures GO-NeRF for object generation in NeRF. It optimizes a hash grid representation. $\lambda_S$ and $\lambda_O$ use a cosine scheduler (30 to 300), and $\lambda_R$ is 500. Training is 20,000 iterations on a single RTX 3090, with 30% iterations using background augmentation [S5].

### Tri-MipRF Training Configuration
Configures Tri-MipRF for anti-aliasing using tri-mip representations. Base mipmap shape is $H=512, W=512, C=16$. Optimizer is AdamW with weight decay $10^{-5}$ and LR $2 \times 10^{-3}$ (MultiStepLR). Tri-Mip encoding LR is scaled 10x relative to MLP. Loss is area loss [S6].

### NeRF† Implementation Configuration
A variant of nerfacto using a standard MLP density field. For Blender: $t_n=2.0, t_f=6.0$, no camera optimizer/appearance/distortion/contraction, white background, batch 2048, 16.5k epochs. For LLFF: no camera optimizer/appearance/distortion, $\ell_\infty$ contraction, batch 1024 (train)/2048 (eval), 30k epochs. On Mip-NeRF 360, it achieves PSNR 24.20, SSIM 0.726, LPIPS 0.247 [S7].

### AR-NeRF Training Configuration
Configures AR-NeRF for depth/defocus learning. R1 regularization weight is 0.2, identity regularizer weight is 15. LR is 0.00006. Training is 200,000 iterations. Mapping network: 3 hidden layers, 256 units. Foreground synthesis: 8 FiLM-SIREN layers, 128 units. Background synthesis: 8 FiLM-SIREN layers, 64 units. Applied to Oxford Flowers, CUB-200-2011, and FFHQ [S8].

### DVGO Reference Architecture
Reference architecture for DVGO (Deep Voxel Grids). Uses Re:NeRF for compression. Hardware is NVIDIA A40 GPU. Software is PyTorch 1.12 [S9].

## Background
The concept of method-specific training configurations emerged as NeRF variants diverged significantly from the original NeRF architecture [S1, S3, S4, S7]. The original NeRF paper [S3] established baseline hyperparameters (8-layer MLP, Adam, MSE), but subsequent works like Mip-NeRF [S1] and Neuralangelo [S4] demonstrated that activation functions, encoding strategies, and optimization schedules were critical for stability and fidelity. The proliferation of specialized architectures (hash grids, SDFs, depth oracles) necessitated distinct training protocols, leading to the categorization of these configurations as a family of methods [S1, S2, S3, S4, S5, S6, S7, S8, S9].

## Sources
[S1] Mip-NeRF 360: Unbounded Anti-Aliased Neural Radiance Fields, https://doi.org/10.1109/cvpr52688.2022.00539, fetched 2026-10-26
[S2] NeUDF: Leaning Neural Unsigned Distance Fields with Volume Rendering, https://doi.org/10.1109/cvpr52729.2023.00031, fetched 2026-10-26
[S3] NeRF-SDP: Efficient Generalizable Neural Radiance Field with Scene Depth Perception, https://doi.org/10.1145/3595916.3626380, fetched 2026-10-30
[S4] Neuralangelo: High-Fidelity Neural Surface Reconstruction, https://doi.org/10.1109/cvpr52729.2023.00817, fetched 2026-10-26
[S5] GO-NeRF: Generating Objects in Neural Radiance Fields for Virtual Reality Content Creation, https://doi.org/10.1109/tvcg.2025.3549558, fetched 2026-10-29
[S6] Tri-MipRF: Tri-Mip Representation for Efficient Anti-Aliasing Neural Radiance Fields, https://doi.org/10.1109/iccv51070.2023.01811, fetched 2026-10-26
[S7] Analyzing the Internals of Neural Radiance Fields, https://doi.org/10.1109/cvprw63382.2024.00288, fetched 2026-10-26
[S8] AR-NeRF: Unsupervised Learning of Depth and Defocus Effects from Natural Images with Aperture Rendering Neural Radiance Fields, https://arxiv.org/abs/2206.06100v1, fetched 2026-10-26
[S9] Two is Better than One: Achieving High-Quality 3D Scene Modeling with a NeRF Ensemble, https://doi.org/10.1007/978-3-031-43153-1_27, fetched 2026-10-26
