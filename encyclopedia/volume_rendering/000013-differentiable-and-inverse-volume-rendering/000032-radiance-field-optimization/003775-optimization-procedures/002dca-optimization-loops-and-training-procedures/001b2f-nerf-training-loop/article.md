# NeRF Training Loop

Also called: IPE (Interatomic Positional Encoding), Interatomic Positional Encoding, NeRF Optimization Loop, NeRF Training Procedure, Neural Radiance Field (NeRF) Training

## Overview
The NeRF Training Loop is the iterative optimization procedure that minimizes the photometric error between rendered images and ground-truth observations by updating the parameters of a coordinate-based neural network [S5, S14, unofficial]. It solves the inverse rendering problem by treating the scene as a continuous volumetric function, optimizing density and color fields via stochastic gradient descent [S5, S14, unofficial]. It sets itself apart from its siblings by defining the core iterative cycle of ray sampling, differentiable volume rendering, and weight updates, whereas methods like [Training Configuration](2332-training-configuration.md) define the static hyperparameters and [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) define the forward pass mechanics [S2, S11, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ training images with known camera poses (extrinsics $C$ and intrinsics $K$) and pixel coordinates. A neural network model $f_\theta$ (typically an MLP) initialized with parameters $\theta$.
*   **Outputs:** Optimized network weights $\theta$ that minimize the reconstruction loss.

**Data Structures:**
*   **Rays:** A ray is defined by an origin $\mathbf{o}$ and direction $\mathbf{d}$. In implementation, rays are often pre-computed and stored in a flat array `rays_rgb` of shape $[(N-1) \times H \times W, 3, 3]$ (where the last 3 is the flattened pixel index, and the inner 3 are origin, direction, and target RGB) [S5, S14, unofficial].
*   **Batch:** A subset of rays sampled for each optimization step, typically of size $N_{rand}$ (e.g., 1024 or 2048) [S5, S14, unofficial].
*   **Checkpoint:** A dictionary containing the model state dict, optimizer state dict, and current step/epoch index [S11, unofficial].

**Numbered Steps:**

1.  **Initialization:**
    *   Load training and validation datasets [S8][S11, unofficial].
    *   Initialize the neural network model $f_\theta$ (e.g., with SIREN initialization or standard Xavier/He initialization) [S3, S15, unofficial].
    *   Initialize the optimizer (typically Adam or AdamW) with a learning rate $lr$ (e.g., $5 \times 10^{-4}$) [S5, S11, S13, unofficial].
    *   Initialize a learning rate scheduler (e.g., Exponential Decay or Cosine Annealing) [S5, S11, unofficial].
    *   Pre-compute all possible rays from training images into `rays_rgb` if batching is enabled [S5, S14, unofficial].

2.  **Iteration Loop:** For iteration $i$ from 1 to $N_{iters}$ (e.g., 1,000,000) [S14, unofficial]:
    *   **Ray Sampling:**
        *   If batching is enabled: Select a batch of $N_{rand}$ rays from `rays_rgb` using index $i_{batch}$ [S5, S14, unofficial]. If $i_{batch}$ exceeds the array length, shuffle `rays_rgb` and reset $i_{batch}$ to 0 [S5, S14, unofficial].
        *   If not batching: Randomly select a training image, generate all rays for that image, optionally apply a central crop (precrop) for early iterations, and sample $N_{rand}$ unique pixel coordinates [S14, unofficial].
        *   Extract ray origins $\mathbf{o}$, directions $\mathbf{d}$, and target colors $\mathbf{c}_{gt}$ for the batch [S7, S14, unofficial].
    *   **Forward Pass (Rendering):**
        *   For each ray in the batch, sample $N$ points along the ray segment $[t_{near}, t_{far}]$ [S5, S13, unofficial].
        *   Query the neural network $f_\theta(\mathbf{x}, \mathbf{d})$ at each sample point to obtain density $\sigma$ and color $\mathbf{c}$ [S5, S6, unofficial].
        *   Compute the rendered color $\hat{\mathbf{c}}$ using the volume rendering equation (alpha compositing) [S1][S5, S6, unofficial].
        *   If using a coarse-fine two-stage model, render with the coarse network, sample additional points based on weights, and render with the fine network [S5, S11, unofficial].
    *   **Loss Computation:**
        *   Compute the Mean Squared Error (MSE) between the rendered color $\hat{\mathbf{c}}$ and the ground truth color $\mathbf{c}_{gt}$ [S5, S6, S14, unofficial]:
            $$ \mathcal{L}_{MSE} = \frac{1}{N_{batch}} \sum_{j=1}^{N_{batch}} \|\hat{\mathbf{c}}_j - \mathbf{c}_{gt,j}\|^2 $$
        *   If coarse outputs are available, add the coarse loss $\mathcal{L}_{coarse}$ to the total loss [S11, S18, unofficial].
        *   Optionally add regularization terms (e.g., sparsity, Total Variation, Eikonal) [S7, unofficial][S8][S16, unofficial].
    *   **Backpropagation and Update:**
        *   Zero the gradients of the optimizer [S18, unofficial].
        *   Compute gradients via backpropagation: $\mathcal{L}_{total}.backward()$ [S18, unofficial].
        *   Step the optimizer: `optimizer.step()` [S18, unofficial].
        *   Update the learning rate according to the scheduler [S5, S11, unofficial].
    *   **Logging and Checkpointing:**
        *   Log loss and PSNR every $i_{print}$ steps [S2, S14, unofficial].
        *   Render validation images every $i_{img}$ steps [S2, S11, unofficial].
        *   Save checkpoints every $i_{weights}$ steps [S2, S11, unofficial].

**Parameters with Typical Values:**
*   $N_{rand}$ (Batch size): 1024 or 2048 [S5, S11, S14, unofficial].
*   $N_{iters}$: 100,000 to 1,000,000 [S11, S14, unofficial].
*   $lr$ (Learning Rate): $5 \times 10^{-4}$ [S11, unofficial].
*   $decay\_rate$: 0.1 [S6, S18, unofficial].
*   $decay\_steps$: $args.lrate\_decay \times 1000$ [S5, S18, unofficial].
*   $chunk$: Maximum rays per forward pass to limit VRAM (e.g., 4096) [S5, S7, unofficial].

**Edge Cases and Pitfalls:**
*   **VRAM Overflow:** Large batches or high-resolution images can exceed GPU memory. Mitigated by chunking rays during the forward pass [S5, S7, unofficial].
*   **Gradient Graph Freeing:** Higher-order derivatives (e.g., for logging gradients or Laplacians) must be computed *before* calling `loss.backward()`, as the backward pass frees the autograd graph [S3, unofficial].
*   **Checkpoint Resumption:** When resuming, the scheduler's last epoch must be set to `start_epoch - 1` to ensure correct scheduling [S9, unofficial]. If a checkpoint step is 0, it is treated as empty/invalid [S15, unofficial].
*   **Instability in Dynamic Growth:** Directly growing sub-networks with random initialization causes density to collapse to 0. Initialize grown sub-network weights to match the parent network to inherit density information [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim
import numpy as np

def train_nerf_loop(model, rays_rgb, target_colors, optimizer, scheduler, 
                    n_iters, n_rand, render_func, loss_fn, log_interval=100, 
                    checkpoint_interval=10000):
    """
    Standard NeRF training loop.
    
    Args:
        model: Neural network model (MLP) taking (x, d) and returning (sigma, rgb).
        rays_rgb: Pre-computed array of shape [N_total_rays, 3, 3] containing 
                  [origin, direction, target_rgb] for all training pixels.
        target_colors: Not used directly if rays_rgb contains targets, but kept for 
                       compatibility with non-batched sampling variants.
        optimizer: PyTorch optimizer (e.g., Adam).
        scheduler: Learning rate scheduler.
        n_iters: Total number of optimization steps.
        n_rand: Number of rays to sample per iteration (batch size).
        render_func: Function that takes rays and model, returns rendered RGB.
        loss_fn: Loss function (e.g., nn.MSELoss).
        log_interval: Steps between logging.
        checkpoint_interval: Steps between saving checkpoints.
    """
    i_batch = 0
    n_total_rays = rays_rgb.shape[0]
    
    for i in range(n_iters):
        # 1. Ray Sampling
        # Slice batch from pre-computed rays
        batch_indices = slice(i_batch, i_batch + n_rand)
        batch_data = rays_rgb[batch_indices]
        
        # Transpose to [3, n_rand, 3]: [origin, direction, target_rgb]
        batch_rays = batch_data.transpose(0, 1)
        rays_o = batch_rays[0] # [n_rand, 3]
        rays_d = batch_rays[1] # [n_rand, 3]
        target_s = batch_rays[2] # [n_rand, 3]
        
        # Increment batch index
        i_batch += n_rand
        
        # Shuffle and reset if exhausted
        if i_batch >= n_total_rays:
            np.random.shuffle(rays_rgb)
            i_batch = 0
            
        # 2. Forward Pass
        # Stack rays for render function
        rays_batch = torch.stack([rays_o, rays_d], dim=0).to(device) # [2, n_rand, 3]
        
        # Render
        # render_func handles chunking internally if needed
        rgb_rendered = render_func(rays_batch, model) # [n_rand, 3]
        
        # 3. Loss Computation
        # Convert to tensors if not already
        target_tensor = torch.tensor(target_s, dtype=torch.float32, device=device)
        loss = loss_fn(rgb_rendered, target_tensor)
        
        # Optional: Add coarse loss if model returns it
        # if 'rgb_coarse' in extras:
        #     loss += loss_fn(extras['rgb_coarse'], target_tensor)
        
        # 4. Backpropagation and Update
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
        
        # 5. Scheduler Step
        scheduler.step()
        
        # 6. Logging and Checkpointing
        if i % log_interval == 0:
            psnr = -10.0 * torch.log10(loss.item())
            print(f"Iter {i}: Loss {loss.item():.6f}, PSNR {psnr:.2f}")
            
        if i % checkpoint_interval == 0:
            save_checkpoint(model, optimizer, scheduler, i)

def render_func(rays, model):
    """
    Simplified render function.
    In practice, this involves sampling points along rays, querying the model,
    and performing alpha compositing.
    """
    # Placeholder for the actual volume rendering logic defined in 
    # [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md)
    # This function must be differentiable.
    pass
```

## Subtopics

### [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md)
The differentiable forward pass that synthesizes RGB images from a NeRF during optimization. It combines ray generation, adaptive marching, neural network evaluation, and alpha compositing into a single differentiable pass.

### [Training Configuration](2332-training-configuration.md)
Defines static and dynamic hyperparameters, optimization routines, and architectural constraints. It specifies *what* parameters are used (learning rates, batch sizes) rather than *how* the loop executes.

### [Neural Volume Scene Setup](4168-neural-volume-scene-setup.md)
Constructs the scene graph, initializing density grids and spherical harmonics coefficients, binding neural network parameters to volume components.

### [MVSNeRF](6197-mvsnerf.md)
A fast, generalizable method using a feedforward network to predict radiance fields directly from images, reducing training time by avoiding per-scene optimization.

### [Camera Pose and Parameter Optimization](7611-camera-pose-and-parameter-optimization.md)
Jointly optimizes NeRF weights and camera poses/intrinsics when parameters are unknown, treating them as learnable variables.

### [Specific Model Architectures](8585-specific-model-architectures.md)
Modifications to the neural network's internal logic, inputs, or outputs to encode specific priors (geometric, generative, semantic).

### [Regularization and Loss Functions](8586-regularization-and-loss-functions.md)
Mathematical terms added to the photometric loss to constrain optimization, addressing floaters and under-constrained geometry.

### [Progressive Inpainting-and-Erasing](8658-progressive-inpainting-and-erasing.md)
A strategy for single-view NeRF that computes a mask of conflicted regions and eliminates them from training to avoid geometry conflicts.

### [KNN-NeRF](8672-knn-nerf.md)
A baseline using sparse 3D point clouds as supervision, assigning nearest point colors to rays.

### [Hard Sample Mining](8673-hard-sample-mining.md)
Selectively updates the network based on pixel-level error, subsampling a "hard" minibatch to retain most update information.

### [Progressive and Dynamic Scene Growth](8869-progressive-and-dynamic-scene-growth.md)
Incrementally expands the neural representation's capacity or training dataset to resolve occlusions and capture high-frequency details.

### [Adaptive Ray Sampling](8870-adaptive-ray-sampling.md)
Optimizes sample point distribution along rays to concentrate resources on complex regions.

### [Sparse View and Initialization Methods](9394-sparse-view-and-initialization-methods.md)
Decouples initialization from random assumptions, leveraging external geometric scaffolds or semantic masks.

### [Scene Representation and Partitioning](9444-scene-representation-and-partitioning.md)
Decomposes the 3D domain into disjoint regions managed by local networks to enable distributed training and scalability.

### [Zip-NeRF Training Procedure](9538-zip-nerf-training-procedure.md)
A two-stage method jointly training a physical scene model, environment map, and radiance caches using a control variate strategy.

### [DPPE (Dense Pose Estimation in a Plenoxels Environment)](9640-dppe-dense-pose-estimation-in-a-plenoxels-environment.md)
A 6-DoF monocular RGB-only pose estimation procedure using numerical gradient approximation in Plenoxels.

### [NeRV Training](10253-nerv-training.md)
Optimizes a representation decoupling geometry, appearance, and lighting via a separate "visibility MLP".

### [NeRV Rendering](10264-nerv-rendering.md)
Computes pixel color by integrating reflected radiance with physically-based shading models.

### [Specific Model Architectures and Extensions](10278-specific-model-architectures-and-extensions.md)
Integrates specialized sampling and preprocessing for unbounded or large-scale scenes.

### [Aug-NeRF](10294-aug-nerf.md)
Injects adversarial perturbations into coordinates, features, and outputs to implicitly regularize the model.

### [Cross-Guided Optimization of Radiance Fields (CROP)](10371-cross-guided-optimization-of-radiance-fields-crop.md)
Performs multi-view image super-resolution during optimization to improve high-resolution novel view synthesis.

### [Truncated Depth-Based Sampling](10613-truncated-depth-based-sampling.md)
Uses coarse monocular depth priors to guide point sampling via a truncated normal distribution.

### [Diffusion-based Regularization for NeRF](10661-diffusion-based-regularization-for-nerf.md)
Incorporates a denoising diffusion model as a prior for density and color fields to mitigate interference in limited-view scenarios.

### [Multiplex Training via S3IM](10894-multiplex-training-via-s3im.md)
Combines pixel-wise loss with Stochastic Structural SIMilarity (S3IM) loss to capture non-local structural information.

### [S3IM (Stochastic Structural SIMilarity)](10895-s3im-stochastic-structural-similarity.md)
A loss function capturing nonlocal structural similarity by processing multiple pixels as a whole set.

### Supersampling Baseline
A baseline method for evaluating NeRF on multiscale data by training on full-resolution images and using brute-force supersampling during evaluation, or training on multiscale data with supersampling during both training and evaluation [S21]. For every pixel, multiple jittered rays are cast through the pixel's spatial footprint, rendered, and averaged [S21]. This approach relies on "oracle" scale information and performs worse than mip-NeRF while being significantly more expensive [S21].

### Two-stage training for proposers
A training schedule split into two equal-duration phases to stabilize learning of a trainable proposer network in NeRF-ID [S22, unofficial]. Phase 1 trains the proposer to mimic a heuristic proposer by minimizing L2 matching loss between proposal sets. Phase 2 trains all components end-to-end, resetting the optimizer and performing a linear warmup for 1,000 steps [S22, unofficial]. This two-stage approach is consistently effective where training from scratch often fails [S22, unofficial].

### Multi-view SDS
A plug-and-play method that creates multi-view supervision during SDS optimization to address the Janus problem caused by one-by-one view sampling [S23]. Rendered images from 4 sampled views are stitched into a 2x2 image, and gradients are calculated under the condition of the multi-view normal map [S23]. It converts 3D assets into differentiable representations (NeRF, 3DGS, DMTet) through rendering supervision [S23].

### Guided NeRF Optimization for MVS
A multi-view depth estimation method utilizing conventional reconstruction (sparse SfM+MVS) and learning-based priors over NeRF [S24]. It performs test-time optimization over implicit volumes to eliminate the need for accurate pixel matching, addressing shape-radiance ambiguity [S24]. Step 1 runs COLMAP for sparse depth maps; Step 2 finetunes a monocular depth network on these maps using scale-invariant loss [S24].

### Surface-aware density formulation
A method to extract high-quality surfaces from NeRF by modeling geometry as a distance function or occupancy field and converting its output to density using a probability density function [S25]. It incorporates an eikonal term to encourage geometric consistency, enforcing that the gradient norm of the learned SDF is equal to one [S25]. The density distribution is $\Phi_S(x) = \frac{se^{-sx}}{1+e^{-sx}}$, where $s$ is a scaling parameter and $x$ is the SDF or occupancy value [S25].

### Dynamically-informed iNeRF Estimator
A state estimation method that uses an iNeRF initialized with a dynamics prior, optimized only via photometric loss, acting as a recursive Bayesian filter with infinite state covariance [S26]. The process loss is set to 0 [S26]. An iNeRF estimator initialized without a dynamics prior will very quickly diverge [S26].

## Operating characteristics
*   **Time Complexity:** $O(N_{iters} \times N_{rand} \times N_{samples})$ per iteration, where $N_{iters}$ is total iterations, $N_{rand}$ is batch size, and $N_{samples}$ is points per ray [S5, S14, unofficial].
*   **Memory Complexity:** $O(N_{rand} \times N_{samples})$ for intermediate rendering states during the forward pass, plus $O(\theta)$ for model parameters [S5, S18, unofficial]. VRAM usage is a primary constraint, mitigated by chunking rays [S5, S7, unofficial].
*   **Measured Performance:** Training typically requires 100,000 to 1,000,000 iterations [S11, S14, unofficial]. Batch sizes of 1024-2048 are common [S5, S11, S14, unofficial]. Larger batch sizes (e.g., $2^{14}$) can hinder data loading performance and reduce throughput, while smaller batches cap iterations per second [S15, unofficial].
*   **Strengths:** End-to-end differentiable, handles complex geometry and appearance, produces high-quality novel views [S5, S14, unofficial].
*   **Limitations:** Slow convergence due to high iteration counts, high memory usage, sensitive to hyperparameters (learning rate, batch size) [S5, S15, S17, unofficial].
*   **When to Use:** When high-quality novel view synthesis is required and computational resources allow for long training times [S5, S14, unofficial].
*   **When Not to Use:** When real-time or fast training is required (consider [MVSNeRF](6197-mvsnerf.md) or [Sparse and Efficient Representations](9326-sparse-and-efficient-representations.md)) [S1].

## Background
The NeRF Training Loop was introduced in the seminal paper "NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis" by Mildenhall et al. (2020) [S5, S18, unofficial]. It builds on the concept of coordinate-based neural networks for implicit scene representation [S3, S18, unofficial]. The method was developed to address the limitations of traditional 3D reconstruction (like MVS) in handling complex geometry and appearance, particularly in scenes with thin structures or reflective surfaces [S5, S18, unofficial]. It is widely used in computer vision and graphics for novel view synthesis, 3D reconstruction, and robotics simulation [S1][S18, unofficial].

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
[S26] Vision-Only Robot Navigation in a Neural Radiance World, https://doi.org/10.1109/lra.2022.3150497, fetched 2026-10-26
