# NeRF Training Loop

Also called: IPE (Interatomic Positional Encoding), Interatomic Positional Encoding, NeRF Optimization Loop, NeRF Training Procedure, Neural Radiance Field (NeRF) Training

## Overview
The NeRF Training Loop is the iterative optimization procedure that minimizes the photometric error between rendered images and ground-truth observations by updating the parameters of a neural radiance field [S5, S6, S14, unofficial]. It solves the inverse problem of reconstructing a continuous 3D scene representation from 2D images by treating the radiance field as a differentiable function optimized via gradient descent [S5, S14, unofficial]. It sets itself apart from its siblings by defining the core stochastic gradient descent cycle—ray sampling, differentiable volume rendering, loss computation, and weight update—upon which specialized variants like [Adaptive Ray Sampling](8870-adaptive-ray-sampling.md) or [Camera Pose and Parameter Optimization](7611-camera-pose-and-parameter-optimization.md) build [S5, S14, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A dataset of $N$ training images with known camera poses (extrinsics/intrinsics) [S5, S11, unofficial]. A neural network model $F_\Theta$ (typically an MLP) parameterized by $\Theta$ [S5, S11, unofficial].
*   **Outputs:** Optimized network weights $\Theta$ and optionally camera poses if jointly optimized [S2, S11, unofficial].

**Data Structures:**
*   **Rays:** A batch of rays is represented as a tensor. Common layouts include:
    *   Shape `[B, 3, 3]` where axis 0 is channel type (0: ray origin, 1: ray direction, 2: target RGB) and axis 1 is batch index [S14, unofficial].
    *   Pre-computed array `rays_rgb` of shape `[(N-1)*H*W, 3, 3]` storing all possible rays from training images, flattened by pixel index [S5, S14, unofficial].
*   **Checkpoint:** A dictionary containing model state (`state_dict`), optimizer state (`state_dict`), and step/epoch counters [S10, S11, unofficial].

**Numbered Steps:**

1.  **Initialization:**
    *   Load training and validation datasets [S9, S11, unofficial].
    *   Initialize the neural network model $F_\Theta$ [S5, S11, unofficial]. For MLPs, weights are often initialized using specific schemes (e.g., SIREN initialization) [S3, S11, unofficial].
    *   Initialize the optimizer (typically Adam or AdamW) with a learning rate $lr$ [S5, S11, S13, unofficial].
    *   Initialize a learning rate scheduler (e.g., Exponential Decay, Cosine Annealing, or LambdaLR) [S5, unofficial][S8][S11, unofficial].
    *   If using a coarse-to-fine approach, initialize both coarse and fine networks [S11, unofficial].

2.  **Ray Sampling:**
    *   Select a batch of $N_{rand}$ rays [S5, S14, unofficial].
    *   If pre-computing rays, slice the `rays_rgb` array or randomly sample indices without replacement [S5, S14, unofficial].
    *   If not pre-computing, randomly select pixel coordinates from training images, convert to ray origins and directions using camera intrinsics, and gather target RGB colors [S4, S14, unofficial].
    *   Apply perturbations to sample locations if `perturb > 0` (stratified sampling) [S5, S18, unofficial].
    *   Apply precropping if iteration $i <$ `precrop_iters`, restricting sampling to a central fraction of the image [S14, unofficial].

3.  **Forward Pass (Differentiable Rendering):**
    *   For each ray in the batch, sample $N$ points along the ray between near and far bounds [S5, S11, unofficial].
    *   Query the neural network $F_\Theta$ at each point to obtain volume density $\sigma$ and color $\mathbf{c}$ [S5, S11, unofficial].
    *   Compute the rendered color $\hat{C}(\mathbf{r})$ using the volume rendering equation (alpha compositing) [S5, S11, unofficial].
    *   If a coarse network is used, render a coarse prediction $\hat{C}_{coarse}$ [S11, unofficial].

4.  **Loss Computation:**
    *   Compute the Mean Squared Error (MSE) between the predicted color $\hat{C}$ and the ground truth color $C_{gt}$ [S5, S6, S14, unofficial]:
        $$ \mathcal{L}_{MSE} = \frac{1}{N_{rays}} \sum_{i=1}^{N_{rays}} \| \hat{C}_i - C_{gt,i} \|^2 $$
    *   If a coarse network is used, compute the coarse loss $\mathcal{L}_{coarse}$ and add it to the total loss [S6, S11, S18, unofficial]:
        $$ \mathcal{L}_{total} = \mathcal{L}_{fine} + \mathcal{L}_{coarse} $$
    *   Optionally add regularization terms (e.g., sparsity, Total Variation, Eikonal) [S7, unofficial][S8][S16, unofficial].
    *   If using Time-of-Flight data, combine RGB and ToF phasor losses [S19, unofficial]:
        $$ \mathcal{L} = \sum \| L_{RGB} - \hat{L}_{RGB} \|^2 + \lambda \sum \| L_{ToF} - \hat{L}_{ToF} \|^2 $$
        where $\lambda$ is a scalar weight, often decayed over iterations [S19, unofficial].

5.  **Backpropagation and Update:**
    *   Zero the gradients of the optimizer [S18, unofficial].
    *   Compute gradients via backpropagation: `loss.backward()` [S18, unofficial].
    *   Step the optimizer: `optimizer.step()` [S18, unofficial].
    *   Update the learning rate according to the scheduler [S9, S18, unofficial].

6.  **Logging and Checkpointing:**
    *   Log loss and metrics (e.g., PSNR) at regular intervals (`i_print`) [S2, S14, unofficial].
    *   Render validation images at regular intervals (`i_img` or `render_interval`) [S2, S9, unofficial].
    *   Save model checkpoints at regular intervals (`i_weights` or `checkpoint_interval`) [S2, S9, S11, unofficial].

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
        train_dataset: Dataset returning (rays_o, rays_d, rgb_gt).
        val_dataset: Validation dataset.
        config: Dictionary containing hyperparameters.
    """
    # 1. Initialization
    optimizer = optim.Adam(model.parameters(), lr=config['lr'])
    # Example: Exponential Decay Scheduler
    scheduler = optim.lr_scheduler.ExponentialLR(optimizer, gamma=config['decay_rate'])
    
    # Pre-compute rays if batching is enabled
    if config['use_batching']:
        rays_rgb = precompute_rays(train_dataset) # Shape: [N_total, 3, 3]
        rays_rgb = shuffle(rays_rgb)
        i_batch = 0
    
    n_iters = config['n_iters']
    batch_size = config['batch_size']
    
    for i in range(n_iters):
        # 2. Ray Sampling
        if config['use_batching']:
            # Slice pre-computed rays
            end_idx = i_batch + batch_size
            batch_data = rays_rgb[i_batch:end_idx]
            i_batch = end_idx
            
            # Shuffle if exhausted
            if i_batch >= len(rays_rgb):
                rays_rgb = shuffle(rays_rgb)
                i_batch = 0
            
            # Split into rays (origin, dir) and targets (rgb)
            # batch_data shape: [batch_size, 3, 3] -> [origins, dirs, targets]
            rays_o = batch_data[:, 0, :]
            rays_d = batch_data[:, 1, :]
            target_rgb = batch_data[:, 2, :]
        else:
            # Random sampling from dataset
            rays_o, rays_d, target_rgb = sample_random_rays(train_dataset, batch_size)
            
        # 3. Forward Pass
        optimizer.zero_grad()
        
        # Render rays
        # model.render_rays returns predicted RGB, and potentially coarse RGB, depth, etc.
        outputs = model.render_rays(rays_o, rays_d, config)
        pred_rgb = outputs['rgb']
        
        # 4. Loss Computation
        # MSE Loss
        mse_loss = nn.functional.mse_loss(pred_rgb, target_rgb)
        loss = mse_loss
        
        # Add coarse loss if applicable
        if 'rgb_coarse' in outputs:
            coarse_loss = nn.functional.mse_loss(outputs['rgb_coarse'], target_rgb)
            loss = loss + coarse_loss
            
        # Add regularization if configured
        if 'eikonal_weight' in config and config['eikonal_weight'] > 0:
            eikonal_loss = compute_eikonal_loss(model, rays_o, rays_d)
            loss = loss + config['eikonal_weight'] * eikonal_loss
            
        # 5. Backpropagation and Update
        loss.backward()
        optimizer.step()
        scheduler.step()
        
        # 6. Logging and Checkpointing
        if i % config['i_print'] == 0:
            psnr = -10.0 * torch.log10(mse_loss)
            print(f"Iter {i}: Loss {loss.item():.4f}, PSNR {psnr.item():.2f}")
            
        if i % config['i_img'] == 0:
            render_validation_image(model, val_dataset, i)
            
        if i % config['i_weights'] == 0:
            save_checkpoint(model, optimizer, i, config['checkpoint_path'])

def precompute_rays(dataset):
    """
    Pre-compute all rays from training images.
    Returns numpy array of shape [N_total, 3, 3].
    """
    all_rays = []
    for img, pose in dataset:
        # Get ray origins and directions for all pixels
        rays_o, rays_d = get_rays_from_image(img, pose)
        # Flatten
        rays_o_flat = rays_o.reshape(-1, 3)
        rays_d_flat = rays_d.reshape(-1, 3)
        rgb_flat = img.reshape(-1, 3)
        
        # Stack: [origins, dirs, rgb] -> [3, H*W, 3]
        batch = torch.stack([rays_o_flat, rays_d_flat, rgb_flat], dim=1)
        all_rays.append(batch)
        
    rays_rgb = torch.cat(all_rays, dim=0).numpy()
    return rays_rgb

def shuffle(array):
    """Shuffle array along first axis."""
    indices = torch.randperm(array.shape[0])
    return array[indices]
```

**Parameters with Typical Values:**
*   `lr` (Learning Rate): $5 \times 10^{-4}$ [S11, unofficial].
*   `batch_size` ($N_{rand}$): 1024 or 2048 rays per iteration [S5, S11, unofficial].
*   `n_iters`: 100,000 to 1,000,000 steps [S11, S14, unofficial].
*   `decay_rate`: 0.1 (Exponential decay factor) [S5, S18, unofficial].
*   `decay_steps`: $args.lrate\_decay \times 1000$ [S5, S18, unofficial].
*   `chunk`: Maximum number of rays processed per forward pass to limit VRAM (e.g., 8192) [S5, S14, unofficial].
*   `perturb`: Jitter amount for sampling (0.0 to 1.0) [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **VRAM Overflow:** Rendering all rays at once can exceed GPU memory. Mitigated by chunking rays during the forward pass [S5, S14, unofficial].
*   **Gradient Freeing:** Higher-order derivatives (for logging gradients/Laplacians) must be computed *before* `loss.backward()` to avoid freeing the autograd graph [S3, unofficial].
*   **Checkpoint Resumption:** When resuming, the scheduler's last epoch must be set to `start_epoch - 1` to ensure correct scheduling [S9, unofficial]. Empty checkpoints (step 0) should be detected and handled [S15, unofficial].
*   **Instability in Growing Networks:** Directly growing sub-networks with random initialization causes instability. Weights of grown sub-networks should be initialized to match the parent network [S1].

## Subtopics

### [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md)
The differentiable forward pass that synthesizes RGB images from a neural radiance field during optimization. It sets itself apart from evaluation by using stochastic perturbations, adaptive sampling strategies (e.g., occupancy grid pruning), and chunked processing to manage memory, whereas evaluation disables these for deterministic results.

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
NeRF-GT is a baseline training protocol for Neural Radiance Fields that utilizes High Dynamic Range (HDR) input views to synthesize novel HDR imagery. It addresses the limitation of standard NeRF, which typically trains on Low Dynamic Range (LDR) images, by directly optimizing the radiance field against HDR ground truth. It differs from standard LDR-based training by explicitly handling high dynamic range data, though it faces challenges in encoding all scene areas uniformly due to the intensity variance.

### [Scene Representation and Architecture](11368-scene-representation-and-architecture.md)
Scene Representation and Architecture defines the neural network topologies and data structures used to approximate the continuous radiance field $F_\Theta: (\mathbf{x}, \mathbf{d}) \rightarrow (\sigma, \mathbf{c})$ in differentiable volume rendering. It solves the problem of representing complex 3D geometry and appearance as a differentiable function that can be optimized end-to-end via gradient descent. It sets itself apart from its siblings by establishing the canonical MLP-based baseline; [Specific Model Architectures](8585-specific-model-architectures.md) modifies the network topology for speed or generalization, while [Scene Representation and Partitioning](9444-scene-representation-and-partitioning.md) decomposes the scene volume for scalability.

### [Acceleration and Sparse Structures](11405-acceleration-and-sparse-structures.md)
This family of methods accelerates NeRF training and rendering by reducing the computational cost of evaluating the neural radiance field. It addresses the bottleneck of querying high-dimensional MLPs for every sample point along every ray. Unlike standard uniform or adaptive sampling over the entire scene bounding box, these methods restrict evaluation to sparse structures (visual hulls) or use lightweight approximations (caches) to predict radiance, thereby reducing the number of expensive neural network queries.

### [Deceptive-NeRF/3DGS Pipeline](11860-deceptive-nerf-3dgs-pipeline.md)
Deceptive-NeRF/3DGS is a sparse-view reconstruction method that enhances NeRF or 3D Gaussian Splatting (3DGS) by continuously densifying input observations with synthetic pseudo-observations generated by a diffusion model during training. It distinguishes itself from sibling methods by introducing an uncertainty map derived from view-consistency warping to guide the diffusion process, ensuring that generated pseudo-observations adhere to epipolar constraints and 3D geometry, rather than relying on ensemble learning or variational inference for uncertainty estimation.

### [MELON](11909-melon.md)
MELON is a method for 3D inverse rendering that simultaneously reconstructs a neural radiance field and estimates camera poses from a set of unposed images. It solves the problem of local minima in pose estimation by using an encoder to amortize inference and a "modulo loss" that maps the latent pose space to a quotient set, allowing gradient descent to converge despite rotational symmetries. Unlike adversarial methods such as GNeRF, MELON does not require a discriminator, enabling ab-initio reconstruction with few images.

### [Depth-Guided NeRF Training via Earth Mover’s Distance](11911-depth-guided-nerf-training-via-earth-mover-s-distance.md)
A NeRF training method that uses a monocular depth prior (DiffDP) and its uncertainty to guide the ray termination distance distribution using Earth Mover's Distance (EMD), rather than directly supervising the rendered depth.

### Supersampling Baseline
A baseline method for evaluating NeRF on multiscale data by training on full-resolution images and using brute-force supersampling during evaluation, or training on multiscale data with supersampling during both training and evaluation [S21]. For every pixel, it casts multiple jittered rays sampled uniformly at random through the spatial footprint of the pixel, renders each ray with NeRF, and uses the mean of those rendered values as the predicted color in the loss function [S21]. The full-resolution supersampling baseline relies on 'oracle' scale information that does not exist for most datasets [S21]. The 16x supersampling baseline performs worse than mip-NeRF and is significantly more expensive during both training and evaluation [S21].

### Two-stage training for proposers
A training schedule split into two equal-duration phases to stabilize learning of a trainable proposer network in NeRF-ID [S22, unofficial]. Phase 1 trains the learnt proposer to mimic the heuristic proposer by minimizing the L2 matching loss between their proposal sets [S22, unofficial]. Phase 2 trains all network components end-to-end, resetting the optimizer state and performing a linear warmup for 1,000 steps to reduce shock from switching regimes [S22, unofficial]. Training from scratch sometimes works but often fails; two-stage training is consistently effective [S22, unofficial].

### Surface-aware density formulation
A method to extract high-quality surfaces from NeRF by modeling geometry as a distance function or occupancy field and converting its output to density using a probability density function [S23]. It incorporates an eikonal term to encourage geometric consistency, enforcing that the gradient norm of the learned SDF is equal to one [S23]. The density distribution is modeled as $\Phi_S(x) = \frac{se^{-sx}}{1+e^{-sx}}$, where $s$ is a scaling parameter and $x$ is the SDF or occupancy value [S23].

### Incremental Registration
A training strategy that registers frames incrementally following temporal order, leveraging the intuition that camera motions in videos are smooth and adjacent poses are close [S24]. At the start of training, it jointly optimizes NeRF parameters $\Theta$ and poses $\{P_1, P_2\}$ from the first two frames $\{I_1, I_2\}$ [S24]. After every $k$ iterations, it adds a new frame $I_i$ and initializes its pose $P_i$ by the previous pose $P_{i-1}$ [S24]. It freezes the learning rate on poses $\{P_i\}$ and NeRF $\Theta$ until all frames are registered, applying a learning rate decay schedule after all $N$ images are added [S24]. The iteration interval $k$ is 1k for CO3D/HO3D and 500 for LLFF; freeze duration is 100k for HO3D/CO3D and 30k for LLFF [S24].

## Operating characteristics
*   **Time Complexity:** $O(N_{iters} \cdot N_{rays} \cdot N_{samples} \cdot C_{MLP})$, where $N_{iters}$ is the number of training steps, $N_{rays}$ is the batch size, $N_{samples}$ is the number of samples per ray, and $C_{MLP}$ is the cost of a forward pass through the neural network [S5, S14, unofficial].
*   **Memory Complexity:** $O(N_{rays} \cdot N_{samples} \cdot D)$, where $D$ is the dimensionality of the network outputs (density, color) [S5, unofficial]. VRAM usage is a primary constraint, mitigated by chunking rays [S5, S14, unofficial].
*   **Measured Performance:** Training typically requires 100,000 to 1,000,000 iterations [S11, S14, unofficial]. For a scene with 199 images, training on 128x128 resolution can take ~39,000 iterations for 5 epochs with a batch size of 1024 [S17, unofficial].
*   **Scaling:** Performance scales with the number of training views and resolution. Larger batch sizes (e.g., $2^{14}$) can improve throughput but may hinder data loading performance and GPU occupancy; $2^{10}$ is often a tradeoff [S15, unofficial].
*   **Strengths:** End-to-end differentiable, handles complex geometry and appearance, produces high-quality novel views.
*   **Limitations:** Slow training and rendering due to MLP evaluations, sensitive to hyperparameters (learning rate, sampling), prone to "floaters" in empty space without regularization.
*   **When to Use:** When high-quality novel view synthesis is required and computational resources are available.
*   **When Not to Use:** When real-time rendering is required (use [Acceleration and Sparse Structures](11405-acceleration-and-sparse-structures.md) or 3DGS), or when input views are extremely sparse without additional priors.

## Background
The NeRF Training Loop was introduced by Mildenhall et al. in "NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis" (2020) [S5, unofficial]. It established the standard approach of optimizing a coordinate-based MLP to represent a 3D scene by minimizing the photometric error between rendered and observed images. The method builds on earlier work in neural implicit representations and differentiable volume rendering [S18, unofficial]. It has since become the foundational training procedure for numerous variants, including mip-NeRF, Instant-NGP, and 3D Gaussian Splatting (which uses a differentiable rasterizer but similar optimization principles) [S11, S15, unofficial].

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
