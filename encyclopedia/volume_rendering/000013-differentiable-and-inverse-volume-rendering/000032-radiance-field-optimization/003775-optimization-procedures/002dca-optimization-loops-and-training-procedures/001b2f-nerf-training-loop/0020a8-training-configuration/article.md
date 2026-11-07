# Training Configuration

Also called: Hyperparameter Configuration, Instant-NGP Training Setup, NGP Training Configuration, Optimization Configuration, Optimizer Settings, Training Configuration & Optimization Setup, Training Hyperparameters, VMesh Training Configuration

## Overview
Training Configuration defines the static hyperparameters, optimizer settings, and loss function weights required to stabilize and accelerate the optimization of differentiable volume rendering models. It solves the problem of translating theoretical radiance field formulations into convergent training procedures by specifying exact network structures, sampling strategies, and optimization schedules. Unlike [Training Loop Setup](1522-training-loop-setup.md), which defines the dynamic runtime sequence of data loading and gradient updates, Training Configuration provides the immutable parameters that govern the optimization landscape. It differs from [Regularization and Loss Functions](8586-regularization-and-loss-functions.md) by focusing on the *values* of the weights and the *optimizer* mechanics rather than the mathematical form of the loss terms themselves.

## How it works
### Inputs and Outputs
- **Inputs**: Scene data (images, camera poses), model architecture definition, and a configuration object containing hyperparameters.
- **Outputs**: Optimizer instance, learning rate scheduler, loss function weights, and sampling parameters initialized for the training loop.

### Data Structures
The configuration is typically stored in a structured object (e.g., a dictionary or class instance) containing:
- `optimizer`: String identifier (e.g., "Adam") and parameters (`betas`, `eps`, `weight_decay`).
- `learning_rate`: Base learning rates for different parameter groups (e.g., embeddings vs. MLP weights).
- `scheduler`: Scheduler type (e.g., "LambdaLR", "CosineAnnealing") and decay parameters.
- `loss_weights`: Dictionary mapping loss terms (e.g., "L2", "SSIM", "Eikonal") to scalar weights.
- `sampling`: Parameters for ray marching (e.g., `num_rays`, `num_steps`, `delta`).
- `training`: Total iterations, batch size, and device settings.

### Steps
1. **Initialize Optimizer**: Create the optimizer instance with the specified learning rates for each parameter group. For example, in Instant-NGP [S7, unofficial], separate learning rates are assigned to embeddings (`lr0`) and the network (`lr1`).
2. **Initialize Scheduler**: Create the learning rate scheduler. Common strategies include linear decay, exponential decay, or cosine annealing [S7, unofficial][S3].
3. **Configure Loss Weights**: Set the scalar multipliers for each component of the total loss function. These may vary between coarse and fine stages [S2].
4. **Set Sampling Parameters**: Define the number of rays per batch, steps per ray, and voxel/grid resolutions [S2][S7, unofficial].
5. **Handle Dataset-Specific Overrides**: Apply specific configurations for standard datasets (e.g., Blender vs. LLFF) to handle background colors or loss functions [S8].

### Formulas
- **Total Loss**: $L = \sum_i w_i L_i$, where $w_i$ is the weight for loss term $L_i$ [S4].
- **Instant-NGP Learning Rate Decay**: $lr(iter) = lr_{init} \cdot 0.1^{\min(iter / iters, 1)}$ [S7, unofficial].
- **Luxar Adaptive Scaling**: For dimension $D \le 3$, the effective learning rate is scaled by $P_D / P^2$, where $P_D = D + D(D+1)/2$ [S9, unofficial].
- **NeuralVDB Subdomain Size**: Must be a multiple of 512 to balance compute waste and dispatch overhead [S12, unofficial].

### Parameters
- **Optimizer**: Adam is the standard optimizer. Typical settings: $\beta_1=0.9, \beta_2=0.99, \epsilon=10^{-15}$ [S7, unofficial][S3] or $\epsilon=10^{-8}$ [S9, unofficial].
- **Learning Rates**:
    - Instant-NGP: Embeddings $lr0=2 \times 10^{-2}$, Network $lr1=10^{-3}$ [S7, unofficial].
    - Direct Voxel Grid: Voxel grids $0.1$, Shallow MLP $10^{-3}$ [S2].
    - Luxar: Base rate $0.01$ [S9, unofficial].
- **Batch Size**:
    - TurboSquid16: 512 points [S1].
    - Direct Voxel Grid: 8,192 rays [S2].
    - NeuralVDB: $2^{16}$ random samples or $2^{12}$ resampled batches [S12, unofficial].
- **Iterations**:
    - Direct Voxel Grid: 10k (coarse) + 20k (fine) [S2].
    - Instant-NGP: 30,000 default [S7, unofficial].
    - Luxar: 20,000 cap [S9, unofficial].
    - NeuralVDB: 2,500 epochs (non-resampled) [S12, unofficial].
- **Sampling**:
    - Instant-NGP: `num_rays`=4096, `num_steps`=512 (CPU) or 1024 (CUDA) [S7, unofficial].
    - Direct Voxel Grid: Step size $\delta = 0.5 \times s$ (voxel size) [S2].
    - NeuralVDB: Subdomain size multiple of 512 [S12, unofficial].

### Edge Cases and Pitfalls
- **Precision**: Use `fp16` (mixed precision) with caution; ensure loss scaling is handled to prevent gradient underflow [S7, 2766-mixed-precision-training.md, unofficial].
- **Boundary Conditions**: In Instant-NGP, the `bound` parameter defines the scene box $[-bound, bound]^3$. If $>1$, adaptive ray marching is invoked [S7, unofficial].
- **Degenerate Input**: In NeuralVDB, if the subdomain size is too small, overlapping halo regions cause redundant computation; if too large, empty subdomains waste compute [S12, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def initialize_training_config(config):
    """
    Initializes the optimizer, scheduler, and loss weights based on configuration.
    """
    # 1. Initialize Optimizer
    # Example: Instant-NGP style with separate LR for embeddings and network
    if config.optimizer == "Adam":
        optimizer = torch.optim.Adam(
            model.get_params(config.lr0, config.lr1),
            betas=(config.beta1, config.beta2),
            eps=config.eps
        )
    else:
        raise ValueError(f"Unsupported optimizer: {config.optimizer}")

    # 2. Initialize Scheduler
    # Example: Instant-NGP LambdaLR decay
    if config.scheduler == "LambdaLR":
        scheduler = torch.optim.lr_scheduler.LambdaLR(
            optimizer,
            lr_lambda=lambda iter: config.lr_decay_factor ** min(iter / config.total_iters, 1)
        )
    elif config.scheduler == "ReduceOnPlateau":
        # Example: Luxar style
        scheduler = torch.optim.lr_scheduler.ReduceLROnPlateau(
            optimizer,
            mode='min',
            factor=config.lr_reduce_factor,
            patience=config.lr_patience,
            threshold=config.lr_threshold,
            min_lr=config.lr_floor
        )
    else:
        scheduler = None

    # 3. Configure Loss Weights
    # Example: Direct Voxel Grid style weights
    loss_weights = {
        "photo": config.w_photo,
        "pt_rgb": config.w_pt_rgb,
        "bg": config.w_bg
    }
    
    # 4. Set Sampling Parameters
    sampling_params = {
        "num_rays": config.num_rays,
        "num_steps": config.num_steps,
        "voxel_size": config.voxel_size
    }

    return optimizer, scheduler, loss_weights, sampling_params
```

## Subtopics

### [Training Loop Setup](1522-training-loop-setup.md)
Defines the procedural steps of a single training iteration (data loading, forward pass, loss, update). It bridges static configuration and dynamic gradient computation.

### [Training Loop with Adaptive Regularization & LR Decay](2408-training-loop-with-adaptive-regularization-lr-decay.md)
Integrates adaptive regularization (orthogonality, L1 density) and exponential LR decay within the loop to stabilize convergence.

### [Learning Rate Scheduling](2701-learning-rate-scheduling.md)
Focuses specifically on adjusting the optimizer's step size to balance convergence speed and accuracy, often coupled with structural changes.

### [Mixed Precision Training](2766-mixed-precision-training.md)
Accelerates optimization on CUDA devices using Automatic Mixed Precision (AMP) to maintain numerical stability with lower precision formats.

### [Training Schedule](3154-training-schedule.md)
Defines the temporal progression of hyperparameters and structural updates (e.g., warm-up, densification), dynamically altering active loss terms.

### [Training Objective for Fine Representation](4686-training-objective-for-fine-representation.md)
Specifies the loss and optimization strategy for the fine-stage of a two-stage pipeline, reducing regularization weights to prioritize photometric accuracy.

### [Method-Specific Training Configurations](6267-method-specific-training-configurations.md)
Defines hyperparameters for specific methods (e.g., Mip-NeRF, DONeRF), addressing unique failure modes like negative density gradients.

### [Regularization and Priors](6281-regularization-and-priors.md)
Defines the specific prior terms (e.g., Eikonal, TV) added to the loss function.

### [PixelNeRF](7471-pixelnerf.md)
Extends vanilla NeRF by leveraging CNN features from images, requiring specific configuration for feature extraction and fusion.

### [Warm-start Encoding](8751-warm-start-encoding.md)
Encodes time-sequences by warm-starting training with weights from the previous frame, ensuring temporal coherency.

### Randomized Background Colors
During training, a random RGB background color is drawn from $[0,1]^3$ when compositing pixel color to encourage reconstruction of a fully opaque background, avoiding semi-transparent artifacts [S8]. Test-time background is set to $(0.5, 0.5, 0.5)$ [S8]. Used for 360 and LLFF datasets; Blender uses a fixed white background [S8].

### Blender Dataset Simplified Configuration
For Blender dataset: one round of sampling, 128 samples for proposal MLP, 32 samples for NeRF MLP, proposal MLP with 4 layers and 256 hidden units, NeRF MLP with 8 layers and 256 (or 512) hidden units, axis-aligned IPE, MSE loss, no distortion regularizer [S8].

### Adam Optimisation with Adaptive Scaling
Standard PyTorch Adam with fused Adam on CUDA. Adaptive model capacity is handled via fixed-pool relocation [S9, unofficial]. Effective rate scaling for $D \le 3$: scaled by $P_D / P^2$, where $P_D = D + D(D+1)/2$ [S9, unofficial]. For $D \ge 4$, additional factor $D^{0.8}$ [S9, unofficial]. Amplitudes get 3× rate, positions 1.5× [S9, unofficial]. Base LR 0.01, reduce-on-plateau (patience 15, factor 0.9, threshold $10^{-3}$, floor $10^{-8}$) [S9, unofficial]. Iteration cap 20,000, early stopping patience 500 [S9, unofficial].

### Checkpoint Loading and Resume Training
Load model and optimizer states from a checkpoint file to resume training or evaluation [S10, unofficial]. Steps: Load model state dict, load optimizer state dict, load gradient scaler state if used, extract start iteration [S10, unofficial].

### LLFF Training Configuration
Distortion regularizer is not disabled. Charbonnier loss is used instead of MSE [S8]. On LLFF with 256 hidden units: mip-NeRF PSNR 26.93, SSIM 0.830, LPIPS 0.177, Time 2.48 hrs, Params 0.61M; Our Model PSNR 26.68, SSIM 0.847, LPIPS 0.150, Time 2.39 hrs, Params 0.84M [S8]. With 512 hidden units: mip-NeRF PSNR 27.01, SSIM 0.845, LPIPS 0.148, Time 6.15 hrs, Params 2.27M; Our Model PSNR 26.86, SSIM 0.858, LPIPS 0.128, Time 3.84 hrs, Params 3.23M [S8]. Our model yields improved SSIM and LPIPS and significant speedup when MLP is large [S8].

### View Sampler
Abstract base class for sampling context and target views [S11, unofficial]. Parameters: `stage`, `is_overfitting`, `cameras_are_circular`, `step_tracker` [S11, unofficial]. Layout: Context view indices `Int64[Tensor, "context_view"]`, Target view indices `Int64[Tensor, "target_view"]` [S11, unofficial].

### NeuralVDB Hyperparameter Configuration
Batch sampling: $2^{16}$ random samples per epoch, or resample subset of input voxels at fixed intervals (100 or 500) and draw $2^{12}$ batches [S12, unofficial]. Subdomain size: Multiple of 512 [S12, unofficial]. Network width: Level-1 half of level-0. Default level-0 width 128, depth 3 (4 layers for more detail) [S12, unofficial]. Optimizer: Adam, LR 0.01. Decay: 0.975 every 100 epochs (no resampling) or 0.75 every 1,000 epochs (with resampling) [S12, unofficial]. Max epochs: 2,500 (non-resampled) [S12, unofficial]. Sparse VDB with active voxel masks yields best visual quality and low noise [S12, unofficial].

## Operating characteristics
- **Time Complexity**: Training time is heavily dependent on the number of iterations and batch size. Instant-NGP achieves convergence in ~30k iterations [S7, unofficial], while Direct Voxel Grid converges in 30k iterations (10k coarse + 20k fine) [S2]. Luxar caps at 20k iterations [S9, unofficial].
- **Memory Complexity**: Depends on the representation. Direct Voxel Grid uses $100^3$ (coarse) and $160^3$ (fine) voxels [S2]. NeuralVDB uses sparse VDB structures, reducing memory compared to dense grids [S12, unofficial]. Instant-NGP uses hierarchical grids up to resolution 300 [S7, unofficial].
- **Measured Performance**: On LLFF dataset with 512 hidden units, Our Model (with specific config) achieves 3.84 hrs training time vs. mip-NeRF's 6.15 hrs [S8]. On Blender dataset, Our Model is significantly faster than mip-NeRF [S8].
- **Strengths**: Proper configuration (e.g., randomized backgrounds for unbounded scenes [S8], adaptive scaling for splats [S9, unofficial]) leads to faster convergence and better visual quality (lower LPIPS, higher SSIM) [S8].
- **Limitations**: Poorly chosen subdomain sizes in NeuralVDB lead to noise or redundant computation [S12, unofficial]. Fixed batch sizes may not scale well to very large scenes without adaptive strategies [S12, unofficial].

## Background
Training configuration has evolved from simple fixed learning rates in early NeRF implementations [37] to complex, multi-stage, and method-specific configurations. The introduction of Instant-NGP [S7, unofficial] popularized hierarchical grid optimization with separate learning rates for embeddings and networks. Direct Voxel Grid [S2] simplified this by optimizing voxel grids directly, requiring specific initialization and sampling strategies. Mip-NeRF 360 [S8] introduced dataset-specific configurations like randomized backgrounds to handle unbounded scenes. Luxar [S9, unofficial] and NeuralVDB [S12, unofficial] further refined configurations for specific domains (microscopy, sparse volumes) with adaptive scaling and hierarchical sampling.

## Sources
[S1] Neural Geometric Level of Detail: Real-time Rendering with Implicit 3D Shapes, https://doi.org/10.1109/cvpr46437.2021.01120, fetched 2026-10-26
[S2] Direct Voxel Grid Optimization: Super-fast Convergence for Radiance Fields Reconstruction, https://doi.org/10.1109/cvpr52688.2022.00538, fetched 2026-10-26
[S3] Zip-NeRF: Anti-Aliased Grid-Based Neural Radiance Fields, https://doi.org/10.1109/iccv51070.2023.01804, fetched 2026-10-26
[S4] 3DGUT: Enabling Distorted Cameras and Secondary Rays in Gaussian Splatting, https://doi.org/10.1109/cvpr52734.2025.02425, fetched 2026-10-26
[S5] Neural Deformable Voxel Grid for Fast Optimization of Dynamic View Synthesis, https://doi.org/10.1007/978-3-031-26319-4_27, fetched 2026-10-26
[S6] VMesh: Hybrid Volume-Mesh Representation for Efficient View Synthesis, https://core.ac.uk/download/560406650.pdf, fetched 2026-10-27
[S7] NeRF-Factory/IDDR-NGP, https://github.com/NeRF-Factory/IDDR-NGP, fetched 2026-10-27 (unofficial)
[S8] Mip-NeRF 360: Unbounded Anti-Aliased Neural Radiance Fields, https://doi.org/10.1109/cvpr52688.2022.00539, fetched 2026-10-26
[S9] Luxar: Gaussian splatting for microscopy and scalable interactive web visualisation of multidimensional scientific data, https://doi.org/10.64898/2026.09.22.753676, fetched 2026-10-26 (unofficial)
[S10] rigved-sanku/NeRF-Implementation-for-Efficient-3D-Scene-Reconstruction, https://github.com/rigved-sanku/NeRF-Implementation-for-Efficient-3D-Scene-Reconstruction, fetched 2026-10-27 (unofficial)
[S11] dcharatan/pixelsplat, https://github.com/dcharatan/pixelsplat, fetched 2026-10-27 (unofficial)
[S12] NeuralVDB: High-resolution Sparse Volume Representation using Hierarchical Neural Networks | NVIDIA, https://www.youtube.com/watch?v=IfgeduMv_ns, fetched 2026-11-04 (unofficial)
