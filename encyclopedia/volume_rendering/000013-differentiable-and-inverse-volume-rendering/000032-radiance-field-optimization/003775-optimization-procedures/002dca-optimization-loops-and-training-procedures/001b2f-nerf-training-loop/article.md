# NeRF Training Loop

Also called: IPE (Interatomic Positional Encoding), Interatomic Positional Encoding, NeRF Optimization Loop, NeRF Training Procedure, Neural Radiance Field (NeRF) Training

## Overview

The NeRF Training Loop is the iterative optimization procedure that minimizes the photometric reconstruction error between rendered images and ground-truth observations by updating the parameters of a neural radiance field [S5, S6, S14, unofficial]. It solves the inverse problem of recovering a continuous 3D scene representation from a set of 2D images by treating the scene as a learnable function $F_\Theta$ mapping 3D coordinates and viewing directions to volume density and color [S3, S5, unofficial]. It sets itself apart from evaluation by using stochastic ray sampling, adaptive perturbations, and chunked processing to manage memory and enable gradient descent, whereas evaluation disables these for deterministic results [S2, S4, S9, unofficial].

## How it works

**Inputs and Outputs:**
*   **Inputs:** A dataset of $N$ training images with known camera poses (extrinsics $P$ and intrinsics $K$) [S5, S11, unofficial]. The neural network parameters $\Theta$ (weights and biases of the MLPs) [S5, S11, unofficial].
*   **Outputs:** Updated network parameters $\Theta$ that minimize the loss function [S5, unofficial].

**Data Structures:**
*   **Rays:** A ray is defined by its origin $\mathbf{o}$ and direction $\mathbf{d}$. In implementation, rays are often pre-computed and stored in a flat array `rays_rgb` of shape $[(N-1) \cdot H \cdot W, 3, 3]$ where axis 0 is the flattened pixel index, axis 1 is the channel type (0: ray origin, 1: ray direction, 2: observed RGB), and axis 2 contains the XYZ/RGB components [S5, S14, unofficial]. Data type is typically `float32` [S5, S14, unofficial].
*   **Batch:** A subset of rays sampled for each optimization step. Transposed batch tensor of shape $[3, B, 3]$ where axis 0 is channel type (0: origin, 1: direction, 2: target), axis 1 is batch index, and axis 2 is component [S14, unofficial].

**Numbered Steps:**

1.  **Initialization:**
    *   Load training and validation datasets [S9, S11, unofficial].
    *   Initialize the neural network model(s) (e.g., coarse and fine MLPs) [S11, unofficial]. If using progressive growth, initialize alpha linear weights of grown sub-networks to match the parent to avoid instability [S1].
    *   Initialize the optimizer (typically Adam or AdamW) with a learning rate $lr$ [S5, S13, unofficial].
    *   Initialize a learning rate scheduler (e.g., Exponential Decay or Cosine Annealing) [S5, S11, unofficial].
    *   Precompute all rays (origins, directions) and target pixel colors from the training images [S11, unofficial]. If batching is enabled, shuffle this array [S5, S11, unofficial].

2.  **Iteration Loop:** For iteration $i$ from 1 to $N_{iters}$:
    *   **Ray Sampling:**
        *   Select a batch of $N_{rand}$ rays from the pre-computed array [S5, S14, unofficial].
        *   If using precropping (for early iterations), restrict sampling to a central crop of the image [S14, unofficial].
        *   Perturb the sampling positions along the rays if `perturb > 0` to break symmetry and improve reconstruction quality [S5, S2, unofficial]. This involves stratified sampling within intervals defined by the sample depths [S5, unofficial].
    *   **Forward Pass (Rendering):**
        *   For each ray in the batch, sample $K$ points along the ray between near and far bounds [S5, S13, unofficial].
        *   Query the neural network $F_\Theta$ at each point to obtain volume density $\sigma$ and color $\mathbf{c}$ [S5, unofficial].
        *   Compute the rendered color $\hat{C}(\mathbf{r})$ using the emission-absorption model (Differentiable Volume Rendering) [S1][S5, S18, unofficial]:
            $$
            \hat{C}(\mathbf{r}) = \sum_{i=1}^K T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i
            $$
            where $\delta_i$ is the distance between samples, and $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$ is the transmittance [S1][S5, S18, unofficial].
        *   If a coarse network is used, compute the coarse loss [S11, unofficial].
    *   **Loss Computation:**
        *   Compute the Mean Squared Error (MSE) between the rendered color $\hat{C}(\mathbf{r})$ and the ground-truth color $\mathbf{C}(\mathbf{r})$ [S5, S14, unofficial]:
            $$
            \mathcal{L}_{MSE} = \frac{1}{M} \sum_{m=1}^M (\hat{C}_m - \mathbf{C}_m)^2
            $$
            where $M$ is the number of pixels in the batch [S14, S17, unofficial].
        *   Add any auxiliary losses (e.g., coarse loss, regularization terms) [S11, S18, unofficial].
    *   **Backpropagation and Update:**
        *   Zero the gradients of the optimizer [S18, unofficial].
        *   Compute gradients via backpropagation: $\nabla_\Theta \mathcal{L}$ [S18, unofficial].
        *   Step the optimizer to update $\Theta$ [S18, unofficial].
        *   Update the learning rate according to the scheduler [S18, unofficial].
    *   **Logging and Checkpointing:**
        *   Log loss and PSNR metrics at regular intervals [S11, S14, unofficial].
        *   Render validation images at regular intervals [S11, unofficial].
        *   Save model checkpoints at regular intervals [S11, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim
from torch.utils.data import DataLoader

def train_nerf_loop(model, train_dataset, val_dataset, config):
    """
    Main training loop for NeRF.
    
    Args:
        model: Neural Radiance Field model (MLP).
        train_dataset: Dataset containing images, poses, and intrinsics.
        val_dataset: Validation dataset.
        config: Configuration object containing hyperparameters.
    """
    # 1. Initialization
    optimizer = optim.Adam(model.parameters(), lr=config.lr)
    # Example scheduler: Exponential Decay
    scheduler = optim.lr_scheduler.ExponentialLR(optimizer, gamma=config.decay_rate)
    
    # Precompute rays if batching is used
    if config.use_batching:
        rays_rgb = precompute_rays(train_dataset) # Shape: [N_pixels, 3, 3]
        rays_rgb = shuffle(rays_rgb)
        i_batch = 0
        
    # 2. Iteration Loop
    for epoch in range(config.num_epochs):
        for iteration, batch_data in enumerate(train_dataset):
            # 2.1 Ray Sampling
            if config.use_batching:
                # Slice batch from precomputed array
                batch_rays = rays_rgb[i_batch : i_batch + config.batch_size]
                i_batch += config.batch_size
                
                # Shuffle and reset if exhausted
                if i_batch >= len(rays_rgb):
                    rays_rgb = shuffle(rays_rgb)
                    i_batch = 0
                    
                # Split into origins/directions and targets
                origins_dirs = batch_rays[:, :2, :] # [B, 2, 3]
                targets = batch_rays[:, 2, :]       # [B, 3]
            else:
                # Random sampling from image grid
                origins_dirs, targets = sample_random_rays(train_dataset, config)
            
            # Perturb sampling if enabled
            if config.perturb > 0:
                origins_dirs, targets = perturb_rays(origins_dirs, targets, config.perturb)

            # 2.2 Forward Pass
            optimizer.zero_grad()
            
            # Render rays
            # model.render_rays returns rendered RGB, depth, etc.
            rendered_rgb, extras = model.render_rays(origins_dirs, config)
            
            # 2.3 Loss Computation
            # Standard MSE loss
            loss = nn.functional.mse_loss(rendered_rgb, targets)
            
            # Add coarse loss if available (for two-stage training)
            if 'rgb0' in extras:
                loss_coarse = nn.functional.mse_loss(extras['rgb0'], targets)
                loss = loss + loss_coarse
                
            # 2.4 Backpropagation and Update
            loss.backward()
            optimizer.step()
            scheduler.step()
            
            # 2.5 Logging
            if iteration % config.log_interval == 0:
                psnr = compute_psnr(rendered_rgb, targets)
                print(f"Epoch {epoch}, Iter {iteration}, Loss: {loss.item():.4f}, PSNR: {psnr:.2f}")
                
            # 2.6 Checkpointing
            if iteration % config.checkpoint_interval == 0:
                save_checkpoint(model, optimizer, epoch, iteration, config.checkpoint_path)

    return model
```

**Parameters:**
*   `lr` (Learning Rate): Typical value $5 \times 10^{-4}$ [S11, unofficial].
*   `batch_size` ($N_{rand}$): Number of rays sampled per iteration. Typical value 1024 or 2048 [S5, S11, unofficial].
*   `num_samples` ($K$): Number of samples per ray. Typical value 128 for coarse, 192 for fine [S5, S11, unofficial].
*   `near` / `far`: Near and far clipping planes. For Blender synthetic data, near=2.0, far=6.0 [S13, S2, unofficial]. For LLFF, computed from bounding depths [S2, unofficial].
*   `perturb`: Jitter amount for sampling. 0.0 for no jitter, 1.0 for max jitter [S2, unofficial].
*   `decay_rate`: Factor for exponential decay. Default 0.1 [S6, S18, unofficial].
*   `decay_steps`: Number of steps for decay. Often `args.lrate_decay * 1000` [S5, S18, unofficial].

**Edge Cases and Pitfalls:**
*   **VRAM Overflow:** Rendering all rays at once can exceed GPU memory. Mitigated by chunking rays into smaller batches during the forward pass [S5, S15, unofficial].
*   **Gradient Freeing:** Higher-order derivatives (e.g., for logging gradients or Laplacians) must be computed *before* `loss.backward()` because the backward pass frees the autograd graph [S3, unofficial].
*   **Checkpoint Resumption:** When resuming, the scheduler's last epoch must be set to `start_epoch - 1` to ensure correct scheduling [S9, unofficial]. If a checkpoint is loaded but the step is 0, it is considered empty and should trigger an error [S15, unofficial].
*   **Instability in Growth:** Directly growing a randomly initialized network can cause density to drop to zero. Sub-network weights should be initialized to match the parent [S1].

## Subtopics

### [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md)
The differentiable forward pass that synthesizes RGB images from a neural radiance field (NeRF) during optimization. It solves the problem of computing gradients for scene parameters by sampling points along camera rays, querying a neural network for volume density and color, and integrating them via alpha compositing. It sets itself apart from evaluation by using stochastic perturbations, adaptive sampling strategies (e.g., occupancy grid pruning), and chunked processing to manage memory, whereas evaluation disables these for deterministic results.

### [Training Configuration](2332-training-configuration.md)
Training Configuration defines the hyperparameters, optimization routines, and architectural constraints required to stabilize and accelerate the differentiable volume rendering optimization process. It addresses the sensitivity of radiance field training to initialization, sampling density, and activation function choices. Unlike general optimization loops, it specifies implementation-specific details such as learning rate schedules, loss weights, and batch sampling strategies that are critical for convergence in methods like mip-NeRF, Instant-NGP, and NeuralVDB.

### [Neural Volume Scene Setup](4168-neural-volume-scene-setup.md)
Constructs a Mitsuba scene containing a volumetric cube and a custom neural radiance field integrator, initializing the density grid and spherical harmonics coefficients. It differs from generic training setups by directly binding the neural network parameters to the scene graph's heterogeneous volume and integrator components.

### [Camera Pose and Parameter Optimization](7611-camera-pose-and-parameter-optimization.md)
Camera Pose and Parameter Optimization refers to the family of methods that jointly optimize the parameters of a Neural Radiance Field (NeRF) and the camera poses (extrinsics) and/or intrinsics (e.g., focal length) from a set of input images. It solves the problem of reconstructing 3D scenes when camera parameters are unknown, inaccurate, or when the system requires online pose estimation for navigation. Unlike baseline NeRF, which relies on pre-estimated poses from Structure-from-Motion (SfM) pipelines like COLMAP, these methods treat camera parameters as learnable variables updated via gradient descent alongside the NeRF weights. This family encompasses static scene reconstruction with unknown poses, hybrid 2D/3D registration, and online robot navigation filters.

### [Regularization and Loss Functions](8586-regularization-and-loss-functions.md)
Regularization and Loss Functions define the objective landscape for optimizing Neural Radiance Fields (NeRFs), addressing the ill-posed nature of inverse volume rendering where infinite density-color configurations can produce identical images. This family supplements the standard photometric reconstruction loss with auxiliary terms that enforce geometric plausibility, structural consistency, or alignment with external priors, thereby preventing degenerate solutions such as "floaters," image copying, or over-saturation. It distinguishes itself from sibling methods like [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) by focusing on the *what* to optimize (the loss landscape) rather than the *how* of sampling, and from [Training Configuration](2332-training-configuration.md) by providing the specific mathematical formulations rather than hyperparameter schedules.

### [Progressive Inpainting-and-Erasing](8658-progressive-inpainting-and-erasing.md)
Progressive Inpainting-and-Erasing (PERF) is a training strategy for single-view NeRF that avoids geometry conflicts between visible and invisible regions by computing a mask of conflicted regions and eliminating them from NeRF training. It sets itself apart from standard single-view NeRF methods by integrating a collaborative RGBD inpainting step with a progressive erasing mechanism to handle the ambiguity of unseen geometry, building on the [NeRF Training Loop](970-nerf-training-loop.md) by dynamically expanding the reference view set.

### [KNN-NeRF](8672-knn-nerf.md)
KNN-NeRF is a baseline method for training Neural Radiance Fields (NeRFs) using sparse 3D point clouds as supervision instead of multi-view images. It solves the problem of generating ground-truth color labels for rays that do not pass through existing points by assigning the color of the nearest point in the cloud to the ray. Unlike standard NeRF which relies on pixel colors from images, KNN-NeRF constructs a dense training signal from sparse geometry, though it struggles with sharp object borders due to the low-frequency bias of neural networks and the sparsity of the input.

### [Progressive and Dynamic Scene Growth](8869-progressive-and-dynamic-scene-growth.md)
Progressive and Dynamic Scene Growth addresses the challenge of reconstructing large-scale or complex 3D scenes from limited input data by incrementally expanding the neural representation's capacity or the training dataset. Unlike static NeRF architectures that require pre-defining the entire hierarchy or fixed network size, these methods adaptively grow the model structure or add synthetic reference views during training to resolve occlusions and capture high-frequency details. This family distinguishes itself from sibling methods like [Adaptive Ray Sampling](8870-adaptive-ray-sampling.md) (which focuses on ray selection) and [Sparse NeRF Reconstruction with Geometric and Semantic Initialization](8779-sparse-nerf-reconstruction-with-geometric-and-semantic-initi.md) (which relies on external priors) by actively modifying the model's topology or the set of supervision signals to improve reconstruction quality in complex regions.

### [Adaptive Ray Sampling](8870-adaptive-ray-sampling.md)
Adaptive ray sampling refers to a family of training procedures in neural radiance fields (NeRF) that replace uniform ray sampling with strategies that concentrate computational resources on regions of high geometric or photometric complexity. It solves the inefficiency of uniform sampling, which wastes evaluations on empty space or smooth regions, by dynamically adjusting sample locations or ray selection based on density, error, depth priors, or spatial partitions. This family is distinguished from [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) by focusing on *where* to sample rather than *how* to integrate, and from [Coarse-to-Fine NeRF](3406-coarse-to-fine-nerf.md) by encompassing broader strategies including error-based mining, depth-guided selection, and superpixel partitioning, not just hierarchical point clustering.

### [Sparse View and Initialization Methods](9394-sparse-view-and-initialization-methods.md)
This family addresses the degradation of NeRF quality—such as blurred geometry, slow convergence, and floaters—when trained on sparse input views (typically 15–20 images or fewer than 50–100 views). It distinguishes itself from standard NeRF training by decoupling the initialization and regularization processes from the default random or empty-space assumptions, instead leveraging external geometric scaffolds, semantic masks, or visibility priors to anchor the optimization.

### [NerfingMVS](9518-nerfingmvs.md)
NerfingMVS is a multi-view depth estimation method that optimizes Neural Radiance Fields (NeRF) directly over implicit volumes, guided by learning-based depth priors derived from sparse Structure-from-Motion and Multi-View Stereo (SfM+MVS) reconstructions. It addresses the shape-radiance ambiguity inherent in NeRF for indoor scenes by eliminating the need for accurate pixel matching across views, instead using external depth estimates to guide the sampling and optimization process. Unlike standard NeRF which samples uniformly or stratified along rays, NerfingMVS incorporates depth priors and their associated confidences into the rendering pipeline to bias sampling towards geometrically plausible regions.

### [NeRV Training](10253-nerv-training.md)
NeRV training optimizes a continuous volumetric representation that decouples scene geometry (shape) from material appearance (reflectance) and lighting interactions (visibility) to enable efficient relighting and view synthesis. It differs from standard NeRF training by introducing a separate "visibility MLP" that predicts ray visibility and termination depth, supervised by consistency terms with the primary shape MLP rather than ground-truth labels, allowing the simulation of direct and one-bounce indirect illumination within the training loop.

### [NeRV Rendering](10264-nerv-rendering.md)
NeRV rendering computes the pixel color for a camera ray by integrating reflected radiance along the ray, where the radiance at each sample point is derived from a physically-based shading model that accounts for direct and indirect illumination. It differs from standard NeRF volume rendering by explicitly modeling surface normals and microfacet BRDF parameters via separate MLPs, and by estimating indirect lighting through secondary visibility queries rather than simple volumetric absorption.

### [Aug-NeRF](10294-aug-nerf.md)
Aug-NeRF is a training framework for Neural Radiance Fields (NeRF) that injects worst-case adversarial perturbations into three distinct levels of the pipeline—input coordinates, intermediate features, and pre-rendering output—to implicitly regularize the model with physical grounds. It addresses the generalization gap in NeRF where interpolation between seen views yields inconsistent and non-smooth geometries. Unlike explicit regularizers like Laplacian or Total Variation which constrain MLP flexibility aggressively, Aug-NeRF uses data-adaptive smoothness priors to enhance generalization to unseen views without sacrificing representation flexibility.

### [Cross-Guided Optimization of Radiance Fields (CROP)](10371-cross-guided-optimization-of-radiance-fields-crop.md)
Cross-Guided Optimization of Radiance Fields (CROP) is a framework that performs multi-view image super-resolution (MVSR) on training-view images during the radiance field optimization process to improve high-resolution novel view synthesis (HRNVS). It fuses single-image super-resolution (SISR) features with voxel-based uncertainty fields to ensure multi-view consistency and high-frequency details. Unlike methods that rely solely on the spectral characteristics of coordinate-based networks for upscaling, CROP explicitly bridges 2D SISR and 5D radiance fields by iteratively refining input images based on rendering errors.

### [Application-Specific Pipelines](11200-application-specific-pipelines.md)
Application-Specific Pipelines are specialized training and optimization procedures designed to address distinct downstream tasks—such as rapid reconstruction, pose estimation, or scene editing—rather than generic radiance field optimization. They diverge from standard NeRF training by integrating external priors (e.g., multi-view stereo data, diffusion models, or dynamics models) and modifying the loss landscape or rendering loop to suit specific constraints like speed, geometric accuracy, or semantic control.

### [NeRF-GT Baseline Training Setup](11349-nerf-gt-baseline-training-setup.md)
NeRF-GT is a baseline training procedure for synthesizing High Dynamic Range (HDR) novel views using Neural Radiance Fields. It addresses the problem of encoding high-dynamic-range scenes by training the radiance field directly on HDR input views, rather than Low Dynamic Range (LDR) inputs. It differs from standard NeRF training [NeRF Training Loop](970-nerf-training-loop.md) by operating in the HDR domain and differs from LDR-based variants [Training Configuration](2332-training-configuration.md) by requiring tone-mapping of predictions for loss computation to handle the high dynamic range.

### [Scene Representation and Architecture](11368-scene-representation-and-architecture.md)
Scene Representation and Architecture defines the parametric function $F_\Theta$ that maps 3D position $\mathbf{x}$ and viewing direction $\mathbf{d}$ to volume density $\sigma$ and color $\mathbf{c}$, serving as the core learnable prior in radiance-field optimization [Parent]. It differs from sibling methods by focusing exclusively on the network topology, input encoding, and spatial decomposition strategies that define *what* is being optimized, whereas siblings like [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) define *how* it is evaluated, and [Training Configuration](2332-training-configuration.md) defines the hyperparameters.

### [Acceleration and Sparse Structures](11405-acceleration-and-sparse-structures.md)
This family of methods accelerates NeRF training and rendering by reducing the computational cost of evaluating the neural radiance field. It addresses the bottleneck of querying high-dimensional MLPs for every sample point along every ray. Unlike standard uniform or adaptive sampling over the entire scene bounding box, these methods restrict evaluation to sparse structures (visual hulls) or use lightweight approximations (caches) to predict radiance, thereby reducing the number of expensive neural network queries.

### Supersampling Baseline
A baseline method for evaluating NeRF on multiscale data by training on full-resolution images and using brute-force supersampling during evaluation, or training on multiscale data with supersampling during both training and evaluation [S21]. For every pixel, it casts multiple jittered rays sampled uniformly at random through the spatial footprint of the pixel, renders each ray with NeRF, and uses the mean of those rendered values as the predicted color in the loss function [S21]. The full-resolution supersampling baseline relies on 'oracle' scale information that does not exist for most datasets [S21]. Performance-wise, the 16x supersampling baseline performs worse than mip-NeRF and is significantly more expensive during both training and evaluation [S21].

### Two-stage training for proposers
A training schedule split into two equal-duration phases to stabilize learning of a trainable proposer network in NeRF-ID [S22, unofficial]. Phase 1 trains the learnt proposer to mimic the heuristic proposer by minimizing the L2 matching loss between their proposal sets [S22, unofficial]. Phase 2 trains all network components end-to-end, resetting the optimizer state and performing a linear warmup for 1,000 SGD steps to reduce shock from switching regimes [S22, unofficial]. Training from scratch sometimes works but often fails; two-stage training is consistently effective [S22, unofficial].

### Surface-aware density formulation
A method to extract high-quality surfaces from NeRF by modeling geometry as a distance function or occupancy field and converting its output to density using a probability density function [S23]. It incorporates an eikonal term to encourage geometric consistency, enforcing that the gradient norm of the learned SDF is equal to one [S23]. The density distribution is modeled as $\Phi_S(x) = \frac{se^{-sx}}{1+e^{-sx}}$, where $s$ is a scaling parameter and $x$ is the SDF or occupancy value [S23].

### Incremental Registration
A training strategy that registers frames incrementally following temporal order, leveraging the intuition that camera motions in videos are smooth and adjacent poses are close [S24]. At the start of training, it jointly optimizes NeRF parameters $\Theta$ and poses $\{P_1, P_2\}$ from the first two frames $\{I_1, I_2\}$ [S24]. After every $k$ iterations, it adds a new frame $I_i$ and initializes its pose $P_i$ by the previous pose $P_{i-1}$ [S24]. It freezes the learning rate on poses $\{P_i\}$ and NeRF $\Theta$ until all frames are registered, applying a learning rate decay schedule after all $N$ images are added [S24]. Typical values for $k$ are 1k iterations for CO3D/HO3D datasets and 500 iterations for LLFF [S24]. Freeze duration is 100k iterations for HO3D and CO3D, and 30k for LLFF [S24].

## Operating characteristics

*   **Time Complexity:** The training loop is iterative, typically running for 100,000 to 1,000,000 iterations [S11, S14, unofficial]. Each iteration involves sampling $N_{rand}$ rays and querying the neural network $K$ times per ray. The total number of network queries is proportional to $N_{iters} \cdot N_{rand} \cdot K$.
*   **Memory Complexity:** Memory usage is dominated by the storage of intermediate states for backpropagation. Standard reverse-mode AD requires storing $O(kn)$ intermediate states (where $n$ is pixels and $k$ is samples), leading to high memory usage [S5, unofficial]. Chunking rays mitigates this by processing subsets of rays sequentially [S5, S15, unofficial].
*   **Measured Performance:** Training on 128x128 images for 250 epochs can take significant time depending on hardware [S4, unofficial]. For example, training on 199 down-sampled images with a batch size of 1024 results in ~7,800 iterations per epoch [S17, unofficial].
*   **Scaling:** Performance scales with the number of training images and the resolution of the input images. Larger batch sizes can improve throughput but may hinder data loading performance and prevent full GPU occupancy [S15, unofficial]. A batch size of $2^{10}$ is often chosen as a trade-off [S15, unofficial].
*   **Strengths:** Stochastic optimization allows for efficient training on large datasets by sampling subsets of rays. The differentiable nature of the rendering operator allows for end-to-end learning of the scene representation.
*   **Limitations:** High computational cost due to the large number of neural network queries required. Sensitive to hyperparameters such as learning rate, batch size, and sampling strategies. Prone to overfitting if not regularized [S1][S17, unofficial].
*   **When to Use:** Suitable for reconstructing static scenes from multi-view images when high-quality novel view synthesis is required.
*   **When Not to Use:** Not suitable for real-time applications without acceleration techniques (e.g., [Acceleration and Sparse Structures](11405-acceleration-and-sparse-structures.md)). Not suitable for dynamic scenes without extensions.

## Background

The NeRF Training Loop was introduced in the seminal paper "NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis" by Mildenhall et al. in 2020 [S5, S18, unofficial]. It builds upon the concept of coordinate-based neural networks for implicit scene representation [S3, S18, unofficial]. The method leverages the differentiability of volume rendering to optimize the neural network parameters via gradient descent [S1][S5, S18, unofficial]. Since its introduction, it has become a standard baseline for novel view synthesis and has inspired numerous variants and extensions, including mip-NeRF, Instant-NGP, and various acceleration techniques [S1][S11, S15, unofficial].

## Sources

[S1] Recursive-NeRF: An Efficient and Dynamically Growing NeRF, https://doi.org/10.1109/tvcg.2022.3204608, fetched 2026-10-26
[S2] GreatDrake/reparameterized-volume-sampling, https://github.com/GreatDrake/reparameterized-volume-sampling, fetched 2026-10-26 (unofficial)
[S3] Yonghao-Lee/differentiable-rendering, https://github.com/Yonghao-Lee/differentiable-rendering, fetched 2026-10-26 (unofficial)
[S4] omkarchittar/Neural_Radiance_Fields, https://github.com/omkarchittar/Neural_Radiance_Fields, fetched 2026-10-26 (unofficial)
[S5] bmild/nerf, https://github.com/bmild/nerf, fetched 2026-10-26 (unofficial)
[S6] yenchenlin/nerf-pytorch, https://github.com/yenchenlin/nerf-pytorch, fetched 2026-10-26 (unofficial)
[S7] yashbhalgat/HashNeRF-pytorch, https://github.com/yashbhalgat/HashNeRF-pytorch, fetched 2026-10-26 (unofficial)
[S8] learning3d/assignment3, https://github.com/learning3d/assignment3, fetched 2026-10-26
[S9] samirangode/Volume-Rendering-and-Neural-Radiance-Fields, https://github.com/samirangode/Volume-Rendering-and-Neural-Radiance-Fields, fetched 2026-10-26 (unofficial)
[S10] Shyam-pi/Volume-Rendering-and-Neural-Radiance-Fields, https://github.com/Shyam-pi/Volume-Rendering-and-Neural-Radiance-Fields, fetched 2026-10-26 (unofficial)
[S11] yashtmy/nerf-3d-vision, https://github.com/yashtmy/nerf-3d-vision, fetched 2026-10-26 (unofficial)
[S12] 3D volumetric rendering with NeRF - Keras, https://keras.io/examples/vision/nerf/, fetched 2026-10-26
[S13] suvadityamuk/NeRF-Experiments, https://github.com/suvadityamuk/NeRF-Experiments, fetched 2026-10-27 (unofficial)
[S14] dgsmith7/nerf-mlp, https://github.com/dgsmith7/nerf-mlp, fetched 2026-10-27 (unofficial)
[S15] blurgyy/jaxngp, https://github.com/blurgyy/jaxngp, fetched 2026-10-27 (unofficial)
[S16] DESY-FS-PETRA/HoloINR, https://github.com/DESY-FS-PETRA/HoloINR, fetched 2026-10-27 (unofficial)
[S17] 7.6. Neural Radiance Fields for Drones, https://www.roboticsbook.org/S76_drone_learning.html, fetched 2026-10-29 (unofficial)
[S18] NeRF: A Volume Rendering Perspective | Yue Yu - GitHub Pages, https://yconquesty.github.io/blog/ml/nerf/nerf_rendering.html, fetched 2026-10-29 (unofficial)
[S19] TöRF: Time-of-Flight Radiance Fields for Dynamic Scene View Synthesis, https://doi.org/10.48550/arxiv.2109.15271, fetched 2026-10-29 (unofficial)
[S20] Overview of NeRF technology and applications, https://www.jcad.cn/en/article/doi/10.3724/SP.J.1089.2023-00759, fetched 2026-10-31
[S21] Mip-NeRF: A Multiscale Representation for Anti-Aliasing Neural Radiance Fields, https://doi.org/10.1109/iccv48922.2021.00580, fetched 2026-10-26
[S22] NeRF in detail: Learning to sample for view synthesis, https://doi.org/10.48550/arxiv.2106.05264, fetched 2026-10-27 (unofficial)
[S23] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S24] ICON: Incremental CONfidence for Joint Pose and Radiance Field Optimization, https://doi.org/10.1109/cvpr52733.2024.00517, fetched 2026-10-26
