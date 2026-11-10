# NeRF Training Loop

Also called: IPE (Interatomic Positional Encoding), Interatomic Positional Encoding, NeRF Optimization Loop, NeRF Training Procedure, Neural Radiance Field (NeRF) Training

## Overview

The NeRF Training Loop is the iterative optimization procedure that minimizes the photometric error between rendered images and ground-truth observations by updating the parameters of a neural radiance field [S5, S6, S14, unofficial]. It solves the inverse problem of reconstructing a continuous 3D scene representation from a set of 2D images by treating the scene as a differentiable function [S5, S14, unofficial]. It sets itself apart from evaluation by using stochastic ray sampling, adaptive perturbations, and chunked processing to manage memory and enable gradient descent, whereas evaluation disables these for deterministic results [S2, S4, S9, unofficial]. It builds on [Optimization Loops and Training Procedures](5622-optimization-loops-and-training-procedures.md) by implementing the specific data loading, ray generation, and loss computation mechanics required for coordinate-based MLPs.

## How it works

**Inputs and Outputs:**
*   **Inputs:** A dataset of $N$ training images with known camera poses (extrinsics $P$ and intrinsics $K$), near/far bounds, and image dimensions $(H, W)$.
*   **Outputs:** Optimized neural network weights $\Theta$ (and optionally camera poses $P$) that minimize the reconstruction loss.

**Data Structures:**
*   **Rays Array (`rays_rgb`):** A pre-computed or on-the-fly generated array containing ray origins, directions, and target RGB colors.
    *   Layout: Shape $[(N-1) \cdot H \cdot W, 3, 3]$ or $[3, B, 3]$ for batches.
    *   Axis 0 (or 1 in batch): Channel type (0: ray origin, 1: ray direction, 2: target RGB).
    *   Axis 1 (or 2 in batch): XYZ/RGB components.
    *   Type: `float32` [S5, S14, unofficial].
*   **Checkpoint:** A dictionary storing model state, optimizer state, and training step/epoch.
    *   Keys: `'model'` (state_dict), `'optimizer'` (state_dict), `'step'` or `'epoch'` (int) [S10, S11, unofficial].

**Numbered Steps:**

1.  **Initialization:**
    *   Load training and validation datasets [S9, S11, unofficial].
    *   Initialize the neural network model(s) (e.g., coarse and fine NeRFs) [S11, unofficial].
    *   Initialize the optimizer (typically Adam or AdamW) with a learning rate $lr$ [S5, S13, unofficial].
    *   Initialize a learning rate scheduler (e.g., Exponential Decay, Cosine Annealing, or LambdaLR) [S5, unofficial][S8][S11, unofficial].
    *   If resuming, load checkpoint and set scheduler last epoch to `start_epoch - 1` [S9, unofficial].

2.  **Ray Sampling (Per Iteration):**
    *   Select a batch of $N_{rand}$ rays from the training set [S5, S14, unofficial].
    *   If using pre-computed rays, slice the `rays_rgb` array and shuffle when exhausted [S5, S14, unofficial].
    *   If generating on-the-fly:
        *   Randomly select pixel coordinates $(x, y)$ from training images [S4, S14, unofficial].
        *   Compute ray origins $\mathbf{o}$ and directions $\mathbf{d}$ using camera intrinsics $K$ and extrinsics $P$ [S4, S14, unofficial].
        *   Optionally apply a "precrop" grid for the first $I_{precrop}$ iterations to focus on the center of the image [S14, unofficial].
        *   Sample target RGB colors $\mathbf{c}_{gt}$ at these coordinates using bilinear interpolation in NDC space [S4, unofficial].

3.  **Forward Pass (Rendering):**
    *   For each ray in the batch, sample $K$ points along the ray segment $[near, far]$ [S5, S14, unofficial].
        *   Uniform or stratified sampling in depth $t \in [0, 1]$, mapped to $z = near(1-t) + far(t)$ [S5, unofficial].
        *   Perturb sampling times if `perturb > 0` to break symmetry [S5, unofficial].
    *   Query the neural network $F_\Theta(\mathbf{x}, \mathbf{d})$ at each sample point to obtain volume density $\sigma$ and color $\mathbf{c}$ [S5, S6, unofficial].
    *   Compute transmittance $T_i$ and weights $w_i$ using the emission-absorption model [S5, S6, unofficial].
    *   Composite the final color $\hat{\mathbf{C}}$ for each ray [S5, S6, unofficial].
    *   If using a coarse-fine scheme, render coarse predictions, sample additional points based on coarse weights, and render fine predictions [S5, S11, unofficial].

4.  **Loss Computation:**
    *   Compute the Mean Squared Error (MSE) between predicted colors $\hat{\mathbf{C}}$ and ground truth $\mathbf{c}_{gt}$ [S5, S14, unofficial]:
        $$
        \mathcal{L}_{rgb} = \frac{1}{B} \sum_{i=1}^B \|\hat{\mathbf{C}}_i - \mathbf{c}_{gt, i}\|^2
        $$
    *   If coarse outputs are available, add coarse loss $\mathcal{L}_{coarse}$ to the total loss [S11, S18, unofficial]:
        $$
        \mathcal{L}_{total} = \mathcal{L}_{coarse} + \mathcal{L}_{rgb}
        $$
    *   (Optional) Add regularization terms (e.g., sparsity, TV, Eikonal) if defined in the specific variant [S7, unofficial][S8].

5.  **Backpropagation and Update:**
    *   Zero gradients: `optimizer.zero_grad()` [S18, unofficial].
    *   Compute gradients: `loss.backward()` [S18, unofficial].
    *   Update weights: `optimizer.step()` [S18, unofficial].
    *   Update learning rate according to the scheduler [S9, S18, unofficial].

6.  **Logging and Checkpointing:**
    *   Log loss and PSNR every $I_{print}$ steps [S2, S14, unofficial].
    *   Render validation images every $I_{img}$ steps [S2, S11, unofficial].
    *   Save checkpoint every $I_{weights}$ steps [S2, S11, unofficial].

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
        train_dataset: Dataset returning (rays_o, rays_d, rgb_gt, camera_params).
        val_dataset: Validation dataset.
        config: Dictionary containing hyperparameters.
    """
    # 1. Initialization
    device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
    model.to(device)
    
    optimizer = optim.Adam(model.parameters(), lr=config['lr'])
    scheduler = optim.lr_scheduler.ExponentialLR(optimizer, gamma=config['decay_rate'])
    
    data_loader = DataLoader(train_dataset, batch_size=config['batch_size'], shuffle=True)
    
    n_iters = config['n_iters']
    log_every = config['log_every']
    save_every = config['save_every']
    
    for step in range(n_iters):
        # 2. Ray Sampling
        # In a real implementation, this happens inside the DataLoader or a custom sampler.
        # Here we assume a batch of rays is provided: rays_o, rays_d, rgb_gt
        batch = next(iter(data_loader))
        rays_o = batch['rays_o'].to(device)      # [B, 3]
        rays_d = batch['rays_d'].to(device)      # [B, 3]
        rgb_gt = batch['rgb_gt'].to(device)      # [B, 3]
        
        # 3. Forward Pass
        # Chunking is used to manage VRAM if B is large
        chunk = config.get('chunk', 8192)
        rgb_pred = []
        for i in range(0, rays_o.shape[0], chunk):
            rays_o_chunk = rays_o[i:i+chunk]
            rays_d_chunk = rays_d[i:i+chunk]
            
            # Sample points along rays
            t_vals = torch.linspace(0., 1., config['num_samples']).to(device)
            z_vals = 0.5 * (t_vals[:-1] + t_vals[1:]) * (config['far'] - config['near']) + config['near']
            
            # Perturb sampling
            if config['perturb'] > 0.:
                mids = 0.5 * (t_vals[:-1] + t_vals[1:])
                upper = torch.cat([mids, t_vals[-1:]], -1)
                lower = torch.cat([t_vals[:1], mids], -1)
                t_rand = torch.rand(z_vals.shape).to(device)
                z_vals = lower + (upper - lower) * t_rand
            
            pts = rays_o_chunk[:, None, :] + rays_d_chunk[:, None, :] * z_vals[:, :, None] # [B, K, 3]
            
            # Query Model
            raw = model(pts, rays_d_chunk) # Returns [B, K, 4] (sigma, rgb)
            
            # Volume Rendering (Simplified)
            # alpha = 1 - exp(-sigma * delta)
            # weights = alpha * cumprod(1 - alpha)
            # rgb = sum(weights * color)
            # Note: Full raw2outputs logic from [S5, unofficial] is omitted for brevity but required for implementation.
            rgb_chunk = raw2outputs(raw, z_vals, rays_d_chunk, config)
            rgb_pred.append(rgb_chunk)
        
        rgb_pred = torch.cat(rgb_pred, dim=0) # [B, 3]
        
        # 4. Loss Computation
        loss_fn = nn.MSELoss()
        loss = loss_fn(rgb_pred, rgb_gt)
        
        # 5. Backpropagation
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
        
        # 6. Logging
        if step % log_every == 0:
            psnr = -10. * torch.log10(loss)
            print(f"Step {step} | Loss {loss.item():.6f} | PSNR {psnr.item():.2f}")
            
        if step % save_every == 0:
            torch.save({
                'step': step,
                'model_state_dict': model.state_dict(),
                'optimizer_state_dict': optimizer.state_dict(),
            }, f'checkpoint_{step}.pth')
            
        scheduler.step()

def raw2outputs(raw, z_vals, rays_d, config):
    """
    Converts raw network outputs to RGB using volume rendering.
    raw: [N_rays, N_samples, 4] (sigma, rgb)
    z_vals: [N_rays, N_samples]
    rays_d: [N_rays, 3]
    """
    # Compute distances between samples
    dists = z_vals[..., 1:] - z_vals[..., :-1]
    dists = torch.cat([dists, torch.tensor([1e10]).to(z_vals.device).expand(dists[..., :1].shape)], -1)
    dists = dists * torch.norm(rays_d[..., None, :], dim=-1)
    
    # Convert sigma to alpha
    noise_std = config.get('raw_noise_std', 0.)
    noise = torch.randn_like(raw[..., 3]) * noise_std
    alpha = 1.0 - torch.exp(-torch.nn.functional.relu(raw[..., 3] + noise) * dists)
    
    # Compute transmittance and weights
    transmittance = torch.cumprod(torch.cat([torch.ones_like(alpha[..., :1]), 1.0 - alpha + 1e-10], -1), -1)[:, :-1]
    weights = alpha * transmittance
    
    # Composite RGB
    rgb = torch.sum(weights[..., None] * raw[..., :3], dim=-2)
    
    return rgb
```

**Parameters with Typical Values:**
*   `lr`: $5 \times 10^{-4}$ [S11, unofficial].
*   `batch_size` ($N_{rand}$): 1024 or 2048 [S5, S11, unofficial].
*   `num_samples`: 128 (coarse), 128 (fine) [S11, S17, unofficial].
*   `near`: 2.0 (Blender), computed from data (LLFF) [S2, S13, unofficial].
*   `far`: 6.0 (Blender), computed from data (LLFF) [S2, S13, unofficial].
*   `decay_rate`: 0.1 [S5, S18, unofficial].
*   `decay_steps`: $args.lrate\_decay \times 1000$ [S5, S18, unofficial].
*   `multires`: 10 (log2 of max frequency for positional encoding) [S2, unofficial].
*   `multires_views`: 4 (log2 of max frequency for direction encoding) [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **VRAM Overflow:** Rendering all rays at once can exceed GPU memory. Mitigated by chunking rays into batches of size `chunk` (e.g., 8192) during the forward pass [S5, S14, unofficial].
*   **Gradient Freeing:** Higher-order derivatives (e.g., for logging gradients or Laplacians) must be computed *before* `loss.backward()` to avoid freeing the autograd graph [S3, unofficial].
*   **Empty Checkpoints:** If a checkpoint is loaded but the step is 0, it is considered invalid/empty and should trigger an error [S15, unofficial].
*   **Scheduler Resumption:** When resuming training, the scheduler's last epoch must be set to `start_epoch - 1` to maintain correct decay history [S9, unofficial].
*   **Instability in Growth:** In growing NeRF variants, randomly initialized sub-networks can collapse to zero density. Mitigated by initializing alpha weights of grown sub-networks to match the parent [S1].

## Subtopics

### [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md)
Training Ray Marching and Integration is the differentiable forward pass that synthesizes RGB images from a neural radiance field (NeRF) during optimization. It solves the problem of computing gradients for scene parameters by sampling points along camera rays, querying a neural network for volume density and color, and integrating them via alpha compositing. It sets itself apart from evaluation by using stochastic perturbations, adaptive sampling strategies (e.g., occupancy grid pruning), and chunked processing to manage memory, whereas evaluation disables these for deterministic results.

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

### Supersampling Baseline
A baseline method for evaluating NeRF on multiscale data by training on full-resolution images and using brute-force supersampling during evaluation, or training on multiscale data with supersampling during both training and evaluation [S21]. For every pixel, it casts multiple jittered rays sampled uniformly at random through the spatial footprint of the pixel, renders each ray with NeRF, and uses the mean of those rendered values as the predicted color in the loss function [S21]. The full-resolution supersampling baseline relies on 'oracle' scale information that does not exist for most datasets [S21]. Performance: The 16x supersampling baseline performs worse than mip-NeRF and is significantly more expensive during both training and evaluation [S21].

### Two-stage training for proposers
A training schedule split into two equal-duration phases to stabilize learning of a trainable proposer network in NeRF-ID [S22, unofficial]. Phase 1: Train the learnt proposer to mimic the heuristic proposer by minimizing the L2 matching loss between their proposal sets [S22, unofficial]. Phase 2: Train all network components end-to-end. At the start of this phase, reset the optimizer state and perform a linear warmup for 1,000 steps to reduce shock from switching regimes [S22, unofficial]. Training from scratch sometimes works but often fails; two-stage training is consistently effective [S22, unofficial].

### Surface-aware density formulation
A method to extract high-quality surfaces from NeRF by modeling geometry as a distance function or occupancy field and converting its output to density using a probability density function [S23]. It incorporates an eikonal term to encourage geometric consistency, enforcing that the gradient norm of the learned SDF is equal to one [S23]. Density distribution: $\Phi_S(x) = \frac{se^{-sx}}{1+e^{-sx}}$, where s is a scaling parameter and x is the SDF or occupancy value [S23].

### Incremental Registration
A training strategy that registers frames incrementally following temporal order, leveraging the intuition that camera motions in videos are smooth and adjacent poses are close [S24]. At the start of training, it jointly optimizes NeRF parameters $\Theta$ and poses $\{P_1, P_2\}$ from the first two frames $\{I_1, I_2\}$ [S24]. After every $k$ iterations, it adds a new frame $I_i$ and initializes its pose $P_i$ by the previous pose $P_{i-1}$ [S24]. It freezes the learning rate on poses $\{P_i\}$ and NeRF $\Theta$ until all frames are registered. A learning rate decay schedule may be applied after all $N$ images are added [S24]. $k$: 1k iterations for CO3D/HO3D datasets; 500 iterations for LLFF dataset [S24]. Freeze duration: 100k iterations for HO3D and CO3D; 30k for LLFF [S24].

## Operating characteristics

*   **Time Complexity:** $O(N_{iters} \cdot N_{rand} \cdot K \cdot C)$, where $N_{iters}$ is the number of training steps, $N_{rand}$ is the batch size (rays per step), $K$ is the number of samples per ray, and $C$ is the cost of a neural network forward pass [S5, S14, unofficial].
*   **Memory Complexity:** $O(N_{rand} \cdot K \cdot D)$, where $D$ is the dimensionality of the network activations. Chunking reduces peak memory usage to $O(chunk \cdot K \cdot D)$ [S5, S14, unofficial].
*   **Measured Performance:** Training typically requires 100,000 to 1,000,000 iterations [S11, S14, unofficial]. For a 199-image dataset, 5 epochs with batch size 1024 results in ~39,000 iterations [S17, unofficial].
*   **Scaling:** Performance scales with batch size up to GPU memory limits. Larger batch sizes (e.g., $2^{14}$) can hinder data loading performance, while smaller batches ($2^{10}$) cap iterations per second [S15, unofficial].
*   **Strengths:** Simple to implement, differentiable end-to-end, produces high-quality novel views.
*   **Limitations:** Slow convergence due to low-frequency bias of MLPs, high memory usage for large scenes, sensitive to hyperparameters (learning rate, sampling strategy) [S5, S17, unofficial].
*   **When to Use:** When high-quality novel view synthesis is required and computational resources are sufficient for iterative optimization.
*   **When Not to Use:** When real-time rendering is required without pre-training, or when input data is extremely sparse and unstructured without additional priors.

## Background

The NeRF Training Loop was introduced by Mildenhall et al. in "NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis" (2020) [S5, unofficial]. It established the standard protocol for optimizing coordinate-based MLPs for 3D scene representation. The method builds on earlier work in neural implicit representations and differentiable volume rendering. It is widely used in computer vision and graphics for novel view synthesis, 3D reconstruction, and augmented reality [S1][S2, S4, S9, unofficial].

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
