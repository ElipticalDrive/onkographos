# NeRF Training Loop

Also called: IPE (Interatomic Positional Encoding), Interatomic Positional Encoding, NeRF Optimization Loop, NeRF Training Procedure, Neural Radiance Field (NeRF) Training

## Overview

The NeRF Training Loop is the iterative optimization procedure that minimizes the photometric error between rendered images and ground-truth observations by updating the parameters of a coordinate-based neural radiance field [S5, S6, unofficial][S8][S14, unofficial]. It solves the inverse rendering problem by repeatedly sampling rays, evaluating the implicit function, integrating volume rendering, and backpropagating gradients [S5, S6, unofficial]. It sets itself apart from sibling methods like [Stage-based Differentiable Volume/Radiance Field Training](4241-stage-based-differentiable-volume-radiance-field-training.md) by typically employing a single unified optimization phase (or coarse-to-fine joint update) rather than decoupling geometry and appearance into distinct training stages, and differs from [Photometric Optimization](5790-photometric-optimization.md) by specifically targeting the neural network weights of the radiance field rather than just camera poses or lighting parameters [S14, unofficial].

## How it works

**Inputs and Outputs:**
*   **Inputs:** A set of $N$ training images with known camera poses (extrinsics $C_i$ and intrinsics $K$) and pixel colors. A neural network model $f_\theta$ (typically an MLP) parameterizing the radiance field, where $\theta$ are the trainable weights.
*   **Outputs:** Updated network weights $\theta$ that minimize the reconstruction loss.

**Data Structures:**
*   **Rays:** A batch of rays is represented as a tensor of shape $[B, 3, 3]$ or $[B, 6]$ depending on implementation, where $B$ is the batch size ($N_{rand}$). The first 3 components are the ray origin $(o_x, o_y, o_z)$, the next 3 are the ray direction $(d_x, d_y, d_z)$ [S5, S14, unofficial]. In some implementations, target RGB colors are appended, forming a shape of $[B, 3, 3]$ where the last channel is RGB [S5, S14, unofficial].
*   **Precomputed Rays:** To accelerate data loading, rays and target colors are often precomputed into a flat array `rays_rgb` of shape $[(N-1) \cdot H \cdot W, 3, 3]$ (for $N$ images of resolution $H \times W$), shuffled once at initialization [S5, S14, unofficial].
*   **Checkpoint:** A dictionary containing `model` (state dict), `optimizer` (state dict), and `epoch` or `step` counter [S10, S11, unofficial].

**Numbered Steps:**

1.  **Initialization:**
    *   Load training and validation datasets [S8][S11, unofficial].
    *   Initialize the neural network model $f_\theta$ [S8][S15, unofficial]. For HashNeRF, initialize the occupancy grid with resolution 128 [S15, unofficial]. For standard NeRF, initialize MLP weights [S5, unofficial].
    *   Initialize the optimizer (typically Adam) with a learning rate $lr$ (default $5 \times 10^{-4}$ or $0.01$) [S5, S11, S17, unofficial].
    *   Initialize a learning rate scheduler (e.g., Exponential Decay or Cosine Annealing) [S5, S11, unofficial].
    *   Precompute all possible rays from training images into `rays_rgb` if batching is enabled [S5, S14, unofficial].

2.  **Iteration Loop (for $i = 1$ to $N_{iters}$):**
    *   **Ray Sampling:**
        *   If using precomputed batching: Slice `rays_rgb` from index $i_{batch}$ to $i_{batch} + N_{rand}$ [S5, S14, unofficial].
        *   If not batching: Randomly select $N_{rand}$ pixel coordinates from a random training image (or a central crop for the first $P_{crop}$ iterations) [S14, unofficial].
        *   Construct the ray bundle: Stack origins and directions into `batch_rays` [S7, unofficial]. Extract target RGB colors `target_s` [S14, unofficial].
        *   Increment $i_{batch}$. If $i_{batch}$ exceeds the total number of rays, shuffle `rays_rgb` and reset $i_{batch}$ to 0 [S5, S11, S14, unofficial].
    *   **Forward Pass (Rendering):**
        *   Query the neural network $f_\theta$ with the sampled 3D points (derived from ray origins, directions, and sampled depths $z$) and view directions [S5, S6, unofficial].
        *   Compute density $\sigma$ and color $\mathbf{c}$ [S5, unofficial].
        *   Perform volume rendering integration (alpha compositing) to compute the predicted color $\hat{C}(\mathbf{r})$ for each ray in the batch [S5, S6, unofficial].
        *   If using a coarse-to-fine scheme (two networks), render both coarse and fine outputs [S11, unofficial].
    *   **Loss Computation:**
        *   Compute the Mean Squared Error (MSE) between predicted RGB $\hat{C}$ and target RGB $C_{gt}$:
            $$ \mathcal{L}_{rgb} = \frac{1}{B} \sum_{j=1}^B (\hat{C}_j - C_{gt,j})^2 $$
            [S14, S17, unofficial].
        *   If coarse-to-fine training is used, add the coarse loss $\mathcal{L}_{coarse}$ to the total loss: $\mathcal{L} = \mathcal{L}_{coarse} + \mathcal{L}_{fine}$ [S11, unofficial].
        *   Optionally add regularization terms (e.g., sparsity, TV, Eikonal) [S7, unofficial][S8].
    *   **Backpropagation and Update:**
        *   Zero gradients: `optimizer.zero_grad()` [S18, unofficial].
        *   Compute gradients: `loss.backward()` [S18, unofficial].
        *   Update weights: `optimizer.step()` [S18, unofficial].
        *   Update learning rate according to the scheduler [S18, unofficial].
    *   **Logging and Checkpointing:**
        *   Log loss and PSNR every $I_{print}$ steps [S2, S11, unofficial].
        *   Render validation images every $I_{img}$ steps [S2, S11, unofficial].
        *   Save checkpoint every $I_{weights}$ steps [S2, S11, unofficial].

**Formulas:**
*   **MSE Loss:**
    $$ \text{MSE}(\mathbf{x}, \mathbf{y}) = \frac{1}{N} \sum_{i=1}^N (x_i - y_i)^2 $$
    [S14, S17, unofficial].
*   **PSNR:**
    $$ \text{PSNR} = -10 \log_{10}(\text{MSE}) $$
    [S14, unofficial].
*   **Learning Rate Decay (Exponential):**
    $$ lr_{new} = lr_{initial} \cdot \text{decay\_rate}^{\left\lfloor \frac{\text{global\_step}}{\text{decay\_steps}} \right\rfloor} $$
    where $\text{decay\_steps} = \text{args.lrate\_decay} \times 1000$ and $\text{decay\_rate} = 0.1$ [S5, S18, unofficial].
*   **Learning Rate Decay (Lambda/Cosine):**
    $$ lr(\text{epoch}) = lr_{initial} \cdot \gamma^{\left\lfloor \frac{\text{epoch}}{\text{step\_size}} \right\rfloor} $$
    [S8][S10, unofficial].

**Parameters:**
*   $N_{rand}$ (Batch Size): Number of rays sampled per iteration. Typical values: 1024 or 2048 [S5, S11, S14, unofficial].
*   $lr$ (Learning Rate): Default $5 \times 10^{-4}$ for Adam [S11, unofficial] or $0.01$ for SGD [S17, unofficial].
*   $N_{iters}$ (Iterations): Total number of optimization steps. Default 100,000 [S11, unofficial] or 1,000,000 [S14, unofficial].
*   $\text{decay\_steps}$: Steps between LR decay. Calculated as $\text{args.lrate\_decay} \times 1000$ [S5, S18, unofficial].
*   $\text{decay\_rate}$: Factor for LR decay. Default 0.1 [S6, S18, unofficial].
*   $\text{chunk}$: Maximum number of rays processed per forward pass to limit VRAM [S5, unofficial].

**Edge Cases and Pitfalls:**
*   **VRAM Overflow:** Rendering all rays at once can exceed GPU memory. Mitigated by splitting rays into chunks (`chunk` argument) during the forward pass [S5, unofficial].
*   **Instability in Growing Networks:** Directly growing a randomly initialized sub-network causes density to collapse to 0. Solution: Initialize alpha linear weights of grown sub-networks to match the parent network [S1].
*   **Gradient Graph Destruction:** Higher-order derivatives (for logging gradients/laplacians) must be computed *before* calling `loss.backward()`, as the backward pass frees the autograd graph [S3, unofficial].
*   **Checkpoint Resumption:** When resuming, the scheduler's last epoch must be set to `start_epoch - 1` to ensure correct scheduling [S9, unofficial]. If a checkpoint has step 0, it is treated as empty/invalid [S15, unofficial].
*   **Precision:** Use `float32` for ray coordinates and colors [S5, S14, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim
import numpy as np

def train_nerf_loop(model, train_data, config):
    """
    Main training loop for NeRF.
    
    Args:
        model: Neural Radiance Field model (MLP).
        train_data: Dictionary containing 'images' (N, H, W, 3), 'poses' (N, 4, 4), 'intrinsics' (K).
        config: Configuration object with hyperparameters.
    """
    # 1. Initialization
    device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
    model.to(device)
    
    # Initialize Optimizer
    optimizer = optim.Adam(model.parameters(), lr=config.lr)
    
    # Initialize Scheduler (Exponential Decay)
    decay_steps = config.lrate_decay * 1000
    decay_rate = 0.1
    def lr_lambda(step):
        return decay_rate ** (step // decay_steps)
    scheduler = optim.lr_scheduler.LambdaLR(optimizer, lr_lambda)
    
    # Precompute Rays (Optional but recommended for batching)
    # rays_rgb shape: (Total_Pixels, 3, 3) -> [origin, direction, target_rgb]
    rays_rgb = precompute_rays(train_data['images'], train_data['poses'], train_data['intrinsics'])
    rays_rgb = rays_rgb.reshape(-1, 3, 3) # Flatten
    # Shuffle once
    indices = np.random.permutation(rays_rgb.shape[0])
    rays_rgb = rays_rgb[indices]
    
    batch_size = config.N_rand
    i_batch = 0
    n_iters = config.n_iters
    
    for step in range(n_iters):
        # 2. Ray Sampling
        if i_batch + batch_size < rays_rgb.shape[0]:
            batch = rays_rgb[i_batch : i_batch + batch_size]
            i_batch += batch_size
        else:
            # Reshuffle and restart
            indices = np.random.permutation(rays_rgb.shape[0])
            rays_rgb = rays_rgb[indices]
            batch = rays_rgb[0 : batch_size]
            i_batch = batch_size
            
        # Split batch into rays (origins, dirs) and targets
        # batch shape: (B, 3, 3) -> axis 0: origin, dir, target
        rays_o = batch[:, 0, :].to(device) # (B, 3)
        rays_d = batch[:, 1, :].to(device) # (B, 3)
        target_s = batch[:, 2, :].to(device) # (B, 3) RGB
        
        # 3. Forward Pass
        optimizer.zero_grad()
        
        # Sample depths along rays (simplified uniform sampling)
        # In practice, this involves stratified sampling or importance sampling
        t_vals = torch.linspace(0., 1., steps=config.num_samples).to(device)
        z_vals = 2.0 * t_vals # Assuming near=2, far=4 for simplicity, or use scene bounds
        pts = rays_o[:, None, :] + rays_d[:, None, :] * z_vals[:, :, None] # (B, N_samples, 3)
        
        # Query Model
        # model returns raw output: (B, N_samples, 4) -> (R, G, B, sigma)
        raw = model(pts, rays_d) 
        
        # Volume Rendering (Alpha Compositing)
        # Convert raw sigma to alpha
        noise_std = config.raw_noise_std
        noise = torch.randn_like(raw[..., 3]) * noise_std
        alpha = 1.0 - torch.exp(-torch.relu(raw[..., 3] + noise)) # (B, N_samples)
        
        # Compute weights
        dists = z_vals[:, 1:] - z_vals[:, :-1]
        dists = torch.cat([dists, torch.tensor([1e10]).to(device)], dim=1)
        
        # Transmittance T_i = exp(-sum_{j<i} sigma_j * delta_j)
        # alpha = 1 - exp(-sigma * delta) => sigma * delta = -log(1-alpha)
        # weights_i = alpha_i * T_i
        # T_i = prod_{j<i} (1 - alpha_j)
        
        alpha_cumprod = torch.cumprod(1.0 - alpha + 1e-10, dim=1)
        transmittance = torch.cat([torch.ones_like(alpha_cumprod[:, :1]), alpha_cumprod[:, :-1]], dim=1)
        weights = alpha * transmittance # (B, N_samples)
        
        # Rendered Color
        rgb = raw[..., :3] # (B, N_samples, 3)
        rgb_map = torch.sum(weights[:, :, None] * rgb, dim=1) # (B, 3)
        
        # 4. Loss Computation
        loss = nn.functional.mse_loss(rgb_map, target_s)
        
        # 5. Backpropagation
        loss.backward()
        optimizer.step()
        scheduler.step()
        
        # 6. Logging
        if step % config.i_print == 0:
            mse = loss.item()
            psnr = -10. * np.log(mse) / np.log(10.)
            print(f"Step {step} | Loss {mse:.6f} | PSNR {psnr:.2f}")
            
        # 7. Checkpointing
        if step % config.i_weights == 0:
            save_checkpoint(model, optimizer, step, config.checkpoint_dir)

def precompute_rays(images, poses, intrinsics):
    """
    Precompute rays and target colors for all pixels in all images.
    """
    N = images.shape[0]
    H, W, _ = images.shape[1:]
    K = intrinsics
    
    all_rays = []
    for i in range(N):
        # Get camera pose
        c2w = poses[i]
        
        # Generate pixel coordinates
        u, v = np.meshgrid(np.arange(W), np.arange(H))
        u = u.flatten()
        v = v.flatten()
        
        # Convert pixel coords to ray directions in camera space
        # x = (u - cx) / fx, y = (v - cy) / fy, z = 1
        fx = K[0, 0]
        fy = K[1, 1]
        cx = K[0, 2]
        cy = K[1, 2]
        
        dirs = np.stack([(u - cx) / fx, -(v - cy) / fy, -np.ones_like(u)], -1) # Note: OpenCV convention often flips y/z
        
        # Transform to world space
        # Ray origin is camera center
        cam_center = c2w[:3, 3]
        ray_dirs = dirs @ c2w[:3, :3].T # (H*W, 3)
        
        # Target colors
        targets = images[i].reshape(-1, 3)
        
        # Stack: origin, direction, target
        # Origin is same for all rays in this image
        origins = np.tile(cam_center, (H*W, 1))
        
        batch_rays = np.stack([origins, ray_dirs, targets], axis=1) # (H*W, 3, 3)
        all_rays.append(batch_rays)
        
    return np.concatenate(all_rays, axis=0)
```

## Subtopics

### [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md)
Training Ray Marching and Integration is the differentiable forward pass that synthesizes RGB images from a Neural Radiance Field (NeRF) during the optimization loop. It solves the problem of mapping discrete, noisy neural network outputs (volume density and color) into a coherent, continuous image by sampling points along camera rays, querying the implicit function, and integrating them via alpha compositing. It sets itself apart from its siblings by combining ray generation, adaptive marching with occupancy grid pruning, neural network evaluation, and alpha compositing into a single differentiable forward pass that supports gradient computation for backpropagation.

### [Training Configuration](2332-training-configuration.md)
Training Configuration defines the static and dynamic hyperparameters, optimization routines, and architectural constraints required to train differentiable volume rendering models. It solves the problem of translating theoretical radiance field formulations into stable, convergent training procedures by specifying exact network structures, sampling strategies, and loss functions. It differs from [Training Loop Setup](1522-training-loop-setup.md) by defining *what* parameters are used (e.g., learning rates, batch sizes, loss weights) rather than *how* the loop executes (tensor layouts, optimizer state interactions). It serves as the foundational configuration for higher-level procedures like [Training Loop](4290-training-loop.md) and [Training Loop with Adaptive Regularization & LR Decay](2408-training-loop-with-adaptive-regularization-lr-decay.md).

### [Neural Volume Scene Setup](4168-neural-volume-scene-setup.md)
Constructs a Mitsuba scene containing a volumetric cube and a custom neural radiance field integrator, initializing the density grid and spherical harmonics coefficients. It differs from generic training setups by directly binding the neural network parameters to the scene graph's heterogeneous volume and integrator components.

### [MVSNeRF](6197-mvsnerf.md)
MVSNeRF is a method for fast, generalizable radiance field reconstruction from multi-view stereo (MVS) images. Unlike standard NeRF which requires per-scene optimization, MVSNeRF uses a feedforward neural network to predict the radiance field directly from input images, significantly reducing training time and iteration counts. It sets itself apart by learning a data prior that accelerates convergence, enabling rapid inference without iterative scene-specific optimization.

### [Camera Pose and Parameter Optimization](7611-camera-pose-and-parameter-optimization.md)
Camera Pose and Parameter Optimization addresses the problem of reconstructing a Neural Radiance Field (NeRF) when the input camera poses are unknown, inaccurate, or require refinement. While standard NeRF pipelines rely on pre-estimated poses from Structure-from-Motion (SfM) tools like COLMAP, this family of methods treats camera parameters (extrinsics and intrinsics) as differentiable variables optimized jointly with the scene representation [7405-nerf-with-unknown-camera-parameters.md]. It sets itself apart from sibling methods like [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) by focusing specifically on the gradient flow through the ray generation and pose update steps, rather than the volume rendering integral itself.

### [Specific Model Architectures](8585-specific-model-architectures.md)
Specific Model Architectures define the neural network structures used to parameterize the radiance field function $f_\theta(\mathbf{x}, \mathbf{d}) \to (\sigma, \mathbf{c})$ within the NeRF training loop. While the standard NeRF uses a single MLP mapping 3D coordinates and view directions to density and color, this family encompasses variants that modify the network topology, input representations, and output mechanisms to address specific challenges such as rendering speed, generalization, geometric ambiguity, and single-view synthesis. These architectures differ from their siblings in the training loop (e.g., [Training Ray Marching](2199-training-ray-marching-and-integration.md) or [Regularization](8586-regularization-and-loss-functions.md)) by fundamentally altering the function being optimized rather than just the sampling strategy or loss weights.

### [Regularization and Loss Functions](8586-regularization-and-loss-functions.md)
Regularization and Loss Functions address the severe under-constraint problem in Neural Radiance Fields (NeRFs), where limited or inconsistent input views lead to geometric artifacts (e.g., "floaters"), overfitting, and implausible geometry. This family sets itself apart from sibling methods like [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) by focusing on the objective function rather than the sampling strategy, and from [Mip-NeRF 360](3306-mip-nerf-360.md) by explicitly introducing priors (diffusion, flow, or geometric) to constrain the solution space beyond photometric consistency.

### [Progressive Inpainting-and-Erasing](8658-progressive-inpainting-and-erasing.md)
Progressive Inpainting-and-Erasing (PERF) is a training strategy for single-view NeRF that avoids geometry conflicts between visible and invisible regions by computing a mask of conflicted regions and eliminating them from NeRF training. It sets itself apart from standard single-view NeRF methods by integrating a collaborative RGBD inpainting step with a progressive erasing mechanism to handle the ambiguity of unseen geometry, building on the [NeRF Training Loop](970-nerf-training-loop.md) by dynamically expanding the reference view set.

### [KNN-NeRF](8672-knn-nerf.md)
KNN-NeRF is a baseline method for training Neural Radiance Fields (NeRFs) using sparse 3D point clouds as supervision instead of multi-view images. It solves the problem of generating ground-truth color labels for rays that do not pass through existing points by assigning the color of the nearest point in the cloud to the ray. Unlike standard NeRF which relies on pixel colors from images, KNN-NeRF constructs a dense training signal from sparse geometry, though it struggles with sharp object borders due to the low-frequency bias of neural networks and the sparsity of the input.

### [Hard Sample Mining](8673-hard-sample-mining.md)
Hard Sample Mining (HSM) is an optimization procedure for neural radiance fields that improves training efficiency and reconstruction quality by selectively updating the network based on pixel-level error. It solves the problem of uniform computational waste in standard NeRF training, where samples contributing little to the gradient are treated equally to those with high error. Unlike sibling methods such as [Probability-based Ray Sampling with Quadtree Subdivision](7481-probability-based-ray-sampling-with-quadtree-subdivision.md) or [Nerfacc](https://github.com/NVlabs/nerfacc) which prune low-weight samples based on density or transmittance, HSM ranks samples using the propagated pixel loss to subsample a "hard" minibatch that retains most of the update information.

### [Progressive and Dynamic Scene Growth](8869-progressive-and-dynamic-scene-growth.md)
Progressive and Dynamic Scene Growth addresses the challenge of reconstructing large-scale or complex 3D scenes from limited input data by incrementally expanding the neural representation's capacity or the training dataset. Unlike static NeRF architectures that require pre-defining the entire hierarchy or fixed network size, these methods adaptively grow the model structure or add synthetic reference views during training to resolve occlusions and capture high-frequency details. This family distinguishes itself from sibling methods like [Adaptive Ray Sampling](8870-adaptive-ray-sampling.md) (which focuses on ray selection) and [Sparse NeRF Reconstruction with Geometric and Semantic Initialization](8779-sparse-nerf-reconstruction-with-geometric-and-semantic-initi.md) (which relies on external priors) by actively modifying the model's topology or the set of supervision signals to improve reconstruction quality in complex regions.

### [Adaptive Ray Sampling](8870-adaptive-ray-sampling.md)
Adaptive Ray Sampling refers to a family of strategies in neural radiance field (NeRF) training that dynamically adjust the distribution of sample points along viewing rays to improve reconstruction quality and training efficiency. Unlike uniform sampling, which treats all space along a ray equally, these methods concentrate computational resources in regions of high geometric or photometric complexity, such as object surfaces or high-frequency details. This family is distinct from sibling methods like [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) by focusing specifically on *where* to sample rather than *how* to integrate, and from [Mip-NeRF 360](3306-mip-nerf-360.md) by typically operating in Euclidean space with point-wise samples rather than conical frustums.

### [Sparse View and Initialization Methods](9394-sparse-view-and-initialization-methods.md)
This family addresses the degradation of NeRF quality—such as blurred geometry, slow convergence, and floaters—when trained on sparse input views (typically 15–20 images or fewer than 50–100 views). It distinguishes itself from standard NeRF training by decoupling the initialization and regularization processes from the default random or empty-space assumptions, instead leveraging external geometric scaffolds, semantic masks, or visibility priors to anchor the optimization.

### [Scene Representation and Partitioning](9444-scene-representation-and-partitioning.md)
Scene Representation and Partitioning addresses the scalability limits of single-NeRF models by decomposing large or dynamic scenes into localized sub-representations. It enables training and rendering of arbitrarily large environments or complex temporal dynamics by distributing computational load across multiple specialized networks, rather than relying on a single monolithic MLP. It differs from siblings like [Mip-NeRF 360](3306-mip-nerf-360.md) which focuses on multi-scale sampling within a single representation, and [Adaptive Ray Sampling](8870-adaptive-ray-sampling.md) which optimizes point density within a single model, by fundamentally altering the scene's structural representation to allow parallelism or dynamic handling.

### [Zip-NeRF Training Procedure](9538-zip-nerf-training-procedure.md)
The Zip-NeRF Training Procedure is a two-stage optimization method for inverse rendering that jointly trains a physically-based scene model, environment map, and radiance caches (Zip-NeRF and Fast Cache) to estimate incoming radiance. It solves the bias introduced by approximating incoming radiance using neural caches by employing a control variate strategy and a specific gradient trick for the photometric loss. It sets itself apart from standard NeRF training by decoupling the cache initialization from the joint optimization of the physical model and by using an occlusion-aware importance sampler to reduce variance in the radiance estimation.

### [DPPE (Dense Pose Estimation in a Plenoxels Environment)](9640-dppe-dense-pose-estimation-in-a-plenoxels-environment.md)
DPPE is a 6-DoF monocular RGB-only pose estimation procedure for Plenoxels environments that recovers the ground truth camera pose after a perturbation by minimizing photometric errors in re-rendering. It distinguishes itself from analytical derivative-based methods (e.g., Teigen et al. [30]) by using numerical gradient approximation, making it generalizable to any scene representation where re-renderings are not prohibitively expensive, and from NeRF-based methods (e.g., iNeRF) by leveraging Plenoxels' rendering speed to re-render the full image at each optimization step rather than subsampling rays.

### [NeRV Training](10253-nerv-training.md)
NeRV training optimizes a continuous volumetric representation that decouples scene geometry (shape) from material appearance (reflectance) and lighting interactions (visibility) to enable efficient relighting and view synthesis. It differs from standard NeRF training by introducing a separate "visibility MLP" that predicts ray visibility and termination depth, supervised by consistency terms with the primary shape MLP rather than ground-truth labels, allowing the simulation of direct and one-bounce indirect illumination within the training loop.

### [NeRV Rendering](10264-nerv-rendering.md)
NeRV rendering computes the pixel color for a camera ray by integrating reflected radiance along the ray, where the radiance at each sample point is derived from a physically-based shading model that accounts for direct and indirect illumination. It differs from standard NeRF volume rendering by explicitly modeling surface normals and microfacet BRDF parameters via separate MLPs, and by estimating indirect lighting through secondary visibility queries rather than simple volumetric absorption.

### [Specific Model Architectures and Extensions](10278-specific-model-architectures-and-extensions.md)
#### Supersampling Baseline
A baseline method for evaluating NeRF on multiscale data by training on full-resolution images and using brute-force supersampling during evaluation, or training on multiscale data with supersampling during both training and evaluation [S21]. For every pixel, multiple jittered rays are sampled uniformly at random through the spatial footprint of the pixel. The mean of those rendered values is used as the predicted color in the loss function [S21]. This baseline relies on 'oracle' scale information that does not exist for most datasets [S21]. The 16x supersampling baseline performs worse than mip-NeRF and is significantly more expensive during both training and evaluation [S21].

#### Two-stage training for proposers
A training schedule split into two equal-duration phases to stabilize learning of a trainable proposer network in NeRF-ID [S22, unofficial]. Phase 1 trains the learnt proposer to mimic the heuristic proposer by minimizing the L2 matching loss between their proposal sets [S22, unofficial]. Phase 2 trains all network components end-to-end, resetting the optimizer state and performing a linear warmup for 1,000 steps to reduce shock from switching regimes [S22, unofficial]. Training from scratch sometimes works but often fails; two-stage training is consistently effective [S22, unofficial].

#### Multi-view SDS
A plug-and-play method that creates multi-view supervision during SDS optimization to address the Janus problem caused by one-by-one view sampling [S23]. Rendered images from 4 sampled views are stitched into a large 2x2 image. Gradients are calculated under the condition of the multi-view normal map [S23]. The method converts a 3D asset into differentiable representations including NeRF, 3DGS, and DMTet through rendering supervision, which are then optimized to improve the quality of the 3D asset [S23].

#### Guided NeRF Optimization for MVS
A multi-view depth estimation method that utilizes conventional reconstruction (sparse SfM+MVS) and learning-based priors over Neural Radiance Fields (NeRF) [S24]. It performs test-time optimization directly over implicit volumes to eliminate the need for accurate pixel matching across views, addressing the shape-radiance ambiguity of NeRF in indoor scenes [S24]. Step 1: Run COLMAP on the test scene to acquire per-view sparse depth maps by projecting fused 3D point clouds after multi-view stereo [S24]. Step 2: Adapt a monocular depth network to the target scene by finetuning it on the sparse depth maps from COLMAP using a scale-invariant loss [S24].

#### Surface-aware density formulation
A method to extract high-quality surfaces from NeRF by modeling geometry as a distance function or occupancy field and converting its output to density using a probability density function [S25]. It incorporates an eikonal term to encourage geometric consistency, enforcing that the gradient norm of the learned SDF is equal to one [S25]. The density distribution is modeled as $\Phi_S(x) = \frac{se^{-sx}}{1+e^{-sx}}$, where $s$ is a scaling parameter and $x$ is the SDF or occupancy value [S25].

## Operating characteristics

*   **Time Complexity:** $O(N_{iters} \cdot N_{rand} \cdot N_{samples})$, where $N_{iters}$ is the number of iterations, $N_{rand}$ is the batch size (rays per step), and $N_{samples}$ is the number of samples per ray [S5, S14, unofficial].
*   **Memory Complexity:** $O(N_{rand} \cdot N_{samples})$ for the forward pass buffers, plus $O(N_{iters})$ if storing history for logging [S5, unofficial]. VRAM usage is a primary constraint, often mitigated by chunking rays [S5, unofficial].
*   **Measured Performance:** Training on 128x128 images typically takes 250 epochs [S4, unofficial]. For larger scenes, 100,000 to 1,000,000 iterations are standard [S11, S14, unofficial]. A batch size of 1024 is common [S5, S11, unofficial].
*   **Scaling:** Performance scales with batch size, but larger batches hinder data loading performance and prevent full GPU occupancy, while smaller batches cap iterations per second [S15, unofficial]. A batch size of $2^{10}$ (1024) is often chosen as a tradeoff [S15, unofficial].
*   **Strengths:** Simple to implement, differentiable end-to-end, produces high-quality novel views.
*   **Limitations:** Slow convergence (requires many iterations), prone to overfitting and "floaters" without regularization, high memory usage for large scenes [S1][S17, unofficial].
*   **When to Use:** When high-quality novel view synthesis is required and computational resources allow for iterative optimization.
*   **When Not to Use:** When real-time inference is required without pre-training, or when input views are extremely sparse and unstructured without additional priors.

## Background

The NeRF Training Loop was introduced by Mildenhall et al. in "NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis" (2020) [S5, S18, unofficial]. It established the standard approach of optimizing a coordinate MLP to minimize photometric error via stochastic gradient descent. Subsequent works have refined the loop with various sampling strategies (e.g., [Adaptive Ray Sampling](8870-adaptive-ray-sampling.md)), architectures (e.g., [Specific Model Architectures](8585-specific-model-architectures.md)), and regularization techniques (e.g., [Regularization and Loss Functions](8586-regularization-and-loss-functions.md)) [S1][S7, S11, unofficial].

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
[S23] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S24] Depth-Guided Optimization of Neural Radiance Fields for Indoor Multi-View Stereo, https://doi.org/10.1109/tpami.2023.3263464, fetched 2026-10-27
[S25] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
