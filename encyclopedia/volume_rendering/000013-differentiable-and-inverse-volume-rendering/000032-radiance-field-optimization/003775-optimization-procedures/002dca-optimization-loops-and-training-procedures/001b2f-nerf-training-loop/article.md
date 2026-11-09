# NeRF Training Loop

Also called: IPE (Interatomic Positional Encoding), Interatomic Positional Encoding, NeRF Optimization Loop, NeRF Training Procedure, Neural Radiance Field (NeRF) Training

## Overview
The NeRF Training Loop is the iterative optimization procedure that minimizes the photometric error between rendered images and ground-truth observations by updating the parameters of a neural radiance field [S5, S6, S14, unofficial]. It solves the inverse problem of reconstructing a 3D scene's geometry and appearance from 2D images by treating the scene as a continuous volumetric function parameterized by a neural network [S5, S14, unofficial]. It sets itself apart from sibling methods like [Stage-based Differentiable Volume/Radiance Field Training](4241-stage-based-differentiable-volume-radiance-field-training.md) by typically employing a unified end-to-end optimization of coarse and fine networks (or a single network) using stochastic gradient descent on random ray batches, whereas other methods may decouple geometry and appearance or use multi-stage hierarchical optimization [S11, S14, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ training images with known camera poses (extrinsics $P_i$ and intrinsics $K_i$), near/far bounds ($near, far$), and hyperparameters (learning rate, batch size, number of iterations) [S5, S11, unofficial].
*   **Outputs:** Trained neural network weights (density $\sigma$ and color $\mathbf{c}$) and optionally camera poses if optimized jointly [S2, S11, unofficial].

**Data Structures:**
*   **Rays Array (`rays_rgb`):** A pre-computed or on-the-fly generated array containing ray origins, directions, and target RGB colors.
    *   Layout: Shape $[(N-1) \times H \times W, 3, 3]$ or $[3, B, 3]$ for a batch [S5, S14, unofficial].
    *   Axis 0 (or 2 in batch): Channel type (0: Ray Origin, 1: Ray Direction, 2: Target RGB) [S5, S14, unofficial].
    *   Axis 1 (or 1 in batch): Spatial components (x, y, z) or batch index [S5, S14, unofficial].
    *   Data Type: `float32` [S5, S14, unofficial].
*   **Checkpoint:** A dictionary storing model state, optimizer state, and iteration count.
    *   Keys: `'step'` (int), `'coarse'` (state_dict), `'fine'` (state_dict), `'optim'` (optimizer state_dict) [S11, unofficial].
    *   Alternative keys: `'model'`, `'optimizer'`, `'epoch'` [S10, unofficial].

**Numbered Steps:**

1.  **Initialization:**
    *   Load training and validation datasets [S8][S11, unofficial].
    *   Initialize the NeRF model(s). For coarse-fine training, initialize both `NeRFMLP_coarse` and `NeRFMLP_fine` [S11, unofficial].
    *   Initialize the optimizer (typically Adam or AdamW) with a learning rate $lr$ (default $5 \times 10^{-4}$) [S11, S13, unofficial].
    *   Initialize a learning rate scheduler (e.g., CosineAnnealingLR or LambdaLR with decay) [S8][S11, unofficial].
    *   Precompute ray bundles if using batching: Generate all possible rays from training images, stack origins, directions, and target RGBs into `rays_rgb`, and shuffle [S5, S14, unofficial].

2.  **Iteration Loop:** Iterate from step $i = 1$ to $N_{iters}$ (default 1,000,000) [S11, S14, unofficial].
    *   **Ray Sampling:**
        *   If batching is enabled, slice `rays_rgb` for the current batch size $N_{rand}$ (default 1024 or 2048) [S5, S14, unofficial].
        *   If not batching, randomly select $N_{rand}$ pixel coordinates from a random training image (optionally using a central crop for the first $precrop\_iters$ steps) [S14, unofficial].
        *   Construct the batch tensor `batch_rays` of shape $[3, N_{rand}, 3]$ containing origins, directions, and target colors [S14, unofficial].
        *   *Optional:* Apply perturbations to sample positions if `perturb > 0` to break symmetry and improve reconstruction [S2, S5, unofficial].
    *   **Forward Pass (Rendering):**
        *   Pass `batch_rays` through the NeRF model(s) [S6, S11, unofficial].
        *   The model samples points along rays, queries density $\sigma$ and color $\mathbf{c}$, and integrates them using the volume rendering equation [S5, S14, unofficial].
        *   If using coarse-fine training, render coarse predictions $\hat{C}_{coarse}$ and fine predictions $\hat{C}_{fine}$ [S11, unofficial].
    *   **Loss Computation:**
        *   Compute the Mean Squared Error (MSE) between predicted RGB $\hat{C}$ and target RGB $C_{gt}$ [S14, unofficial]:
            $$ \mathcal{L}_{rgb} = \frac{1}{N_{rand}} \sum_{j=1}^{N_{rand}} (\hat{C}_j - C_{gt,j})^2 $$
        *   If coarse output is available, compute coarse loss $\mathcal{L}_{coarse}$ and add to total loss [S6, S18, unofficial]:
            $$ \mathcal{L}_{total} = \mathcal{L}_{coarse} + \mathcal{L}_{fine} $$
        *   *Optional:* Add regularization terms (e.g., Eikonal loss, sparsity loss) if configured [S8][S13, unofficial].
    *   **Backpropagation and Update:**
        *   Zero gradients: `optimizer.zero_grad()` [S18, unofficial].
        *   Compute gradients: `loss.backward()` [S18, unofficial].
        *   Update weights: `optimizer.step()` [S18, unofficial].
        *   Update learning rate scheduler [S8][S11, unofficial].
    *   **Logging and Checkpointing:**
        *   Log loss and PSNR every $i_{print}$ steps (default 100) [S2, S11, unofficial].
        *   Render validation images every $i_{img}$ steps (default 500) [S2, S11, unofficial].
        *   Save checkpoint every $i_{weights}$ steps (default 10,000) [S2, S11, unofficial].

**Formulas:**
*   **MSE Loss:**
    $$ \text{MSE}(x, y) = \text{mean}((x - y)^2) $$
    [S14, S17, unofficial]
*   **PSNR from MSE:**
    $$ \text{PSNR} = -10 \log_{10}(\text{MSE}) $$
    [S14, unofficial]
*   **Learning Rate Decay (Exponential):**
    $$ lr_{new} = lr_{initial} \times \text{decay\_rate}^{(\text{global\_step} / \text{decay\_steps})} $$
    Where $\text{decay\_steps} = \text{args.lrate\_decay} \times 1000$ and $\text{decay\_rate} = 0.1$ [S5, S18, unofficial].
*   **Learning Rate Decay (Lambda):**
    $$ lr(\text{epoch}) = lr_{initial} \times \gamma^{(\text{epoch} / \text{step\_size})} $$
    [S8][S10, unofficial]

**Parameters:**
*   `N_rand`: Number of rays per batch. Typical: 1024, 2048 [S5, S14, unofficial].
*   `lr`: Learning rate. Typical: $5 \times 10^{-4}$ [S11, unofficial].
*   `n_iters`: Total iterations. Typical: 100,000 to 1,000,000 [S11, S14, unofficial].
*   `chunk`: Max rays per forward pass to limit VRAM. Value: Not given in the sources (implementation dependent) [S5, unofficial].
*   `perturb`: Jitter amount for sampling. Typical: 0.0 to 1.0 [S2, unofficial].
*   `raw_noise_std`: Noise added to density output. Typical: 0.0 or $10^{-5}$ [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **VRAM Overflow:** Rendering all rays at once can exceed GPU memory. Mitigated by chunking rays (`chunk` parameter) [S5, S14, unofficial].
*   **Gradient Graph Freeing:** Higher-order derivatives (e.g., for logging gradients) must be computed *before* `loss.backward()` [S3, unofficial].
*   **Resuming Training:** When loading a checkpoint, the scheduler's last epoch must be set to `start_epoch - 1` to maintain correct decay [S9, unofficial].
*   **Empty Checkpoints:** If a checkpoint has step 0, it is considered empty and should trigger an error or re-initialization [S15, unofficial].
*   **Instability in Growing Networks:** Directly growing sub-networks with random initialization causes density to drop to zero. Initialize grown sub-network weights to match the parent network [S1].

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
        model: Neural Radiance Field model (coarse/fine or single).
        train_dataset: Dataset yielding (rays_o, rays_d, target_rgb, near, far).
        val_dataset: Validation dataset.
        config: Dictionary containing hyperparameters.
    """
    # 1. Initialization
    optimizer = optim.Adam(model.parameters(), lr=config['lr'])
    # Optional: Scheduler
    scheduler = optim.lr_scheduler.CosineAnnealingLR(optimizer, T_max=config['n_iters'])
    
    # Precompute rays if batching is used (Simplified for brevity)
    # rays_rgb = precompute_rays(train_dataset) 
    # rays_rgb = shuffle(rays_rgb)
    
    i = 0
    while i < config['n_iters']:
        start_time = time.time()
        
        # 2. Ray Sampling
        # Option A: Random batch from precomputed rays
        if config['use_batching']:
            batch_indices = slice(i % len(rays_rgb), (i + config['batch_size']) % len(rays_rgb))
            batch_rays = rays_rgb[batch_indices] # Shape: [B, 3, 3]
        else:
            # Option B: Random pixels from random image
            img_idx = random.randint(0, len(train_dataset) - 1)
            rays_o, rays_d, target_rgb, near, far = train_dataset[img_idx]
            # Sample N_rand pixels
            H, W = target_rgb.shape[:2]
            px = torch.randint(0, W, (config['batch_size'],))
            py = torch.randint(0, H, (config['batch_size'],))
            rays_o = rays_o[py, px]
            rays_d = rays_d[py, px]
            target_rgb = target_rgb[py, px]
            batch_rays = torch.stack([rays_o, rays_d, target_rgb], dim=0) # [3, B, 3]
            
        # 3. Forward Pass
        optimizer.zero_grad()
        
        # Render coarse and fine if applicable
        if hasattr(model, 'render_coarse') and hasattr(model, 'render_fine'):
            rgb_coarse, extras_coarse = model.render_coarse(batch_rays[:2], near, far)
            rgb_fine, _ = model.render_fine(batch_rays[:2], near, far)
            target = batch_rays[2]
            
            # 4. Loss Computation
            loss_coarse = nn.functional.mse_loss(rgb_coarse, target)
            loss_fine = nn.functional.mse_loss(rgb_fine, target)
            loss = loss_coarse + loss_fine
        else:
            rgb, _ = model.render(batch_rays[:2], near, far)
            target = batch_rays[2]
            loss = nn.functional.mse_loss(rgb, target)
            
        # Optional: Add regularization (e.g., Eikonal)
        # if config.get('eikonal_weight', 0) > 0:
        #     loss += config['eikonal_weight'] * compute_eikonal_loss(model, batch_rays[:2])

        # 5. Backpropagation
        loss.backward()
        optimizer.step()
        
        # 6. Logging and Checkpointing
        if i % config['i_print'] == 0:
            mse = loss.item()
            psnr = -10.0 * math.log10(mse)
            print(f"Iter {i}: Loss {mse:.6f}, PSNR {psnr:.2f}")
            
        if i % config['i_img'] == 0:
            render_validation(model, val_dataset, i)
            
        if i % config['i_weights'] == 0:
            save_checkpoint(model, optimizer, i, config['checkpoint_path'])
            
        i += 1
        if scheduler:
            scheduler.step()
```

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

### [Scene Representation and Architecture](11368-scene-representation-and-architecture.md)
Scene Representation and Architecture defines the mathematical mapping and neural network structures that convert 3D spatial coordinates and viewing directions into volumetric properties (density and color). It solves the problem of how to compactly and accurately represent a 3D scene's geometry and appearance for differentiable rendering. It sets itself apart from its siblings by focusing on the *definition* of the scene function itself, whereas [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) handles the numerical evaluation of that function, and [Training Configuration](2332-training-configuration.md) handles the hyperparameters of the optimization process.

### [Acceleration and Sparse Structures](11405-acceleration-and-sparse-structures.md)
Acceleration and Sparse Structures optimize the computational bottleneck of NeRF training and rendering by replacing dense, uniform ray sampling with sparse data structures or constrained sampling volumes. This family solves the problem of excessive MLP invocations and memory bandwidth limitations inherent in standard volumetric rendering. It sets itself apart from sibling methods like [Adaptive Ray Sampling](8870-adaptive-ray-sampling.md) by explicitly discretizing the scene into grids or hulls to enable memory-bound lookups rather than iterative density-based decisions, and from [Scene Representation and Architecture](11368-scene-representation-and-architecture.md) by focusing on inference-time speedups via caching rather than training-time representation efficiency.

### Supersampling Baseline
A baseline method for evaluating NeRF on multiscale data by training on full-resolution images and using brute-force supersampling during evaluation, or training on multiscale data with supersampling during both training and evaluation [S21]. For every pixel, it casts multiple jittered rays sampled uniformly at random through the spatial footprint of the pixel, renders each ray with NeRF, and uses the mean of those rendered values as the predicted color in the loss function [S21]. This baseline relies on 'oracle' scale information that does not exist for most datasets [S21]. The 16x supersampling baseline performs worse than mip-NeRF and is significantly more expensive during both training and evaluation [S21].

### Two-stage training for proposers
A training schedule split into two equal-duration phases to stabilize learning of a trainable proposer network in NeRF-ID [S22, unofficial]. Phase 1 trains the learnt proposer to mimic the heuristic proposer by minimizing the L2 matching loss between their proposal sets [S22, unofficial]. Phase 2 trains all network components end-to-end, resetting the optimizer state and performing a linear warmup for 1,000 SGD steps to reduce shock from switching regimes [S22, unofficial]. Training from scratch sometimes works but often fails; two-stage training is consistently effective [S22, unofficial].

### Surface-aware density formulation
A method to extract high-quality surfaces from NeRF by modeling geometry as a distance function or occupancy field and converting its output to density using a probability density function [S23]. It incorporates an eikonal term to encourage geometric consistency, enforcing that the gradient norm of the learned SDF is equal to one [S23]. The density distribution is defined as $\Phi_S(x) = \frac{se^{-sx}}{1+e^{-sx}}$, where $s$ is a scaling parameter and $x$ is the SDF or occupancy value [S23].

### Incremental Registration
A training strategy that registers frames incrementally following temporal order, leveraging the intuition that camera motions in videos are smooth and adjacent poses are close [S24]. At the start of training, it jointly optimizes NeRF parameters $\Theta$ and poses $\{P_1, P_2\}$ from the first two frames $\{I_1, I_2\}$ [S24]. After every $k$ iterations, it adds a new frame $I_i$ and initializes its pose $P_i$ by the previous pose $P_{i-1}$ [S24]. It freezes the learning rate on poses $\{P_i\}$ and NeRF $\Theta$ until all frames are registered, applying a learning rate decay schedule after all $N$ images are added [S24]. Typical values for $k$ are 1k iterations for CO3D/HO3D datasets and 500 iterations for LLFF dataset [S24]. Freeze duration is 100k iterations for HO3D and CO3D, and 30k for LLFF [S24].

### NeRF-GT Baseline Training Setup
1. Train the NeRF model using HDR views (or consistently exposed LDR views) as input [S25]. 2. During training, tone-map the predicted HDR pixel values into LDR pixel values [S25]. 3. Compute the loss by comparing the tone-mapped predictions to the tone-mapped ground truth LDR views [S25]. Training directly on HDR views can fail to encode all scene areas well due to the high dynamic range, even when using tone-mapped colors for loss calculation [S25].

## Operating characteristics
*   **Time Complexity:** $O(N_{iters} \times N_{rand} \times N_{samples} \times D)$, where $N_{iters}$ is the number of iterations, $N_{rand}$ is the batch size (rays per step), $N_{samples}$ is the number of samples per ray, and $D$ is the depth of the MLP [S5, S14, unofficial].
*   **Memory Complexity:** $O(N_{rand} \times N_{samples} \times D_{model})$, where $D_{model}$ is the number of model parameters. VRAM usage is dominated by the intermediate activations during the forward pass, mitigated by chunking [S5, S14, unofficial].
*   **Measured Performance:** Training on 128x128 images typically takes 250 epochs or ~100,000 iterations [S4, S11, unofficial]. With a batch size of 1024, ~8M rays are processed per epoch on a 200x200 scene [S17, unofficial].
*   **Scaling:** Performance scales with batch size, but larger batches hinder data loading performance and prevent full GPU occupancy, while smaller batches cap iterations per second [S15, unofficial]. A batch size of $2^{10}$ (1024) is a common tradeoff [S15, unofficial].
*   **Strengths:** Simple to implement; stochastic optimization allows for efficient use of large datasets; differentiable nature allows for joint optimization of geometry and appearance [S5, S14, unofficial].
*   **Limitations:** High memory usage for large scenes or high-resolution images; slow convergence due to the need for many iterations; sensitive to hyperparameters like learning rate and sampling strategy [S5, S14, S17, unofficial].
*   **When to use:** Standard multi-view reconstruction with known camera poses [S5, S14, unofficial].
*   **When not to use:** When real-time performance is required (use [Acceleration and Sparse Structures](11405-acceleration-and-sparse-structures.md)); when camera poses are unknown (use [Camera Pose and Parameter Optimization](7611-camera-pose-and-parameter-optimization.md)); when sparse views are available (use [Sparse View and Initialization Methods](9394-sparse-view-and-initialization-methods.md)).

## Background
The NeRF Training Loop was introduced in the seminal paper "NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis" by Mildenhall et al. in 2020 [S5, S11, unofficial]. It builds upon the concept of coordinate-based neural networks for implicit scene representation [S3, S5, unofficial]. The method was developed to solve the problem of novel view synthesis from a sparse set of images by optimizing a continuous volumetric function [S5, unofficial]. It has since become the standard baseline for many subsequent NeRF variants and inverse rendering methods [S11, S14, unofficial].

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
[S25] HDR-NeRF: High Dynamic Range Neural Radiance Fields, https://xhuangcv.github.io/hdr-nerf/, fetched 2026-10-27
