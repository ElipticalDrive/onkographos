# NeRF Training Loop

Also called: IPE (Interatomic Positional Encoding), Interatomic Positional Encoding, NeRF Optimization Loop, NeRF Training Procedure, Neural Radiance Field (NeRF) Training

## Overview

The NeRF Training Loop is the iterative optimization procedure that minimizes the photometric reconstruction error between rendered images and ground-truth training views by updating the parameters of a Neural Radiance Field (NeRF) [S5, S6, unofficial][S8][S14, unofficial]. It solves the inverse problem of recovering a continuous 5D radiance field from discrete 2D images by treating the neural network weights as scene parameters [S5, S14, unofficial]. It sets itself apart from its siblings by defining the core stochastic gradient descent cycle—ray sampling, differentiable volume rendering, loss computation, and weight update—that serves as the foundation for all NeRF variants, including those using adaptive sampling [8870-adaptive-ray-sampling.md] or specific architectures [8585-specific-model-architectures.md].

## How it works

**Inputs and Outputs:**
*   **Inputs:** A set of $N$ training images with known camera poses (extrinsics $C_i$ and intrinsics $K_i$) and pixel coordinates. A neural network model $f_\theta$ (typically an MLP) parameterized by $\theta$. An optimizer (e.g., Adam) [S5, S6, S14, unofficial].
*   **Outputs:** Optimized network weights $\theta^*$ and a trained radiance field representation.

**Data Structures:**
*   **Rays:** A ray is defined by an origin $\mathbf{o}$ and direction $\mathbf{d}$. In implementation, rays are often batched into a tensor `rays_rgb` of shape $[B, 3, 3]$ or $[B, 7]$ (origin, direction, near, far, target RGB) [S5, S14, unofficial].
    *   `rays_rgb` layout (pre-computed): Array of shape $[(N-1) \times H \times W, 3, 3]$. Axis 0: flattened pixel index. Axis 1: channel type (0: ray origin, 1: ray direction, 2: observed RGB). Axis 2: XYZ/RGB components. Data type: `float32` [S5, S14, unofficial].
*   **Model State:** Checkpoints store the model state dictionary (`state_dict`), optimizer state dictionary, and training step/epoch [S10, S11, unofficial].
    *   Checkpoint structure: Dictionary with keys `'step'` (int), `'coarse'` (state_dict), `'fine'` (state_dict), `'optim'` (optimizer state_dict) [S11, unofficial].

**Numbered Steps:**

1.  **Initialization:**
    *   Load training and validation datasets [S8][S11, unofficial].
    *   Initialize the NeRF model (MLP) with weights $\theta$ [S5, S11, unofficial]. Common initialization uses small random values or specific schemes like SIREN initialization [S3, S11, unofficial].
    *   Initialize the optimizer (e.g., Adam with learning rate $lr$) [S5, S11, S17, unofficial].
    *   Initialize learning rate scheduler (e.g., Exponential Decay or Cosine Annealing) [S5, unofficial][S8][S11, unofficial].
    *   Precompute ray bundles for all training pixels if batching is enabled [S5, S14, unofficial].

2.  **Iteration Loop (for $i = 1$ to $N_{iters}$):**
    *   **Ray Sampling:**
        *   Select a batch of $N_{rand}$ rays from the pre-computed pool or randomly from the current training image [S5, S14, unofficial].
        *   If `use_batching` is True, slice the pre-computed `rays_rgb` array [S5, S14, unofficial].
        *   If `use_batching` is False, randomly select a training image, generate rays for its pixels, optionally apply a central crop (precrop) for the first $precrop\_iters$ steps, and sample $N_{rand}$ unique pixel coordinates [S14, unofficial].
        *   Extract ray origins $\mathbf{o}$, directions $\mathbf{d}$, and target colors $\mathbf{c}_{gt}$ for the batch [S5, S14, unofficial].
    *   **Forward Pass (Differentiable Volume Rendering):**
        *   Sample $K$ points along each ray within the near/far bounds [S5, S6, unofficial].
        *   Query the NeRF model $f_\theta(\mathbf{x}, \mathbf{d})$ at each sample point to obtain density $\sigma$ and color $\mathbf{c}$ [S5, S6, unofficial].
        *   Compute the rendered color $\hat{\mathbf{c}}$ using the volume rendering equation (standard emission-absorption model) [S1][S5, S6, unofficial].
        *   If using a coarse-fine model, render with both coarse and fine networks [S11, unofficial].
    *   **Loss Computation:**
        *   Compute the Mean Squared Error (MSE) loss between the rendered color $\hat{\mathbf{c}}$ and the ground truth color $\mathbf{c}_{gt}$ [S5, S6, S14, unofficial].
        *   Formula: $\mathcal{L}_{MSE} = \frac{1}{B} \sum_{j=1}^B \|\hat{\mathbf{c}}_j - \mathbf{c}_{gt,j}\|^2$ [S14, S17, unofficial].
        *   If coarse output $\hat{\mathbf{c}}_0$ is available, add its MSE loss to the total loss: $\mathcal{L} = \mathcal{L}_{MSE}(\hat{\mathbf{c}}, \mathbf{c}_{gt}) + \mathcal{L}_{MSE}(\hat{\mathbf{c}}_0, \mathbf{c}_{gt})$ [S11, S18, unofficial].
        *   Add regularization terms if applicable (e.g., sparsity, Total Variation, Eikonal) [S7, unofficial][S8][S16, unofficial].
    *   **Backpropagation and Update:**
        *   Zero the gradients: `optimizer.zero_grad()` [S18, S17, unofficial].
        *   Compute gradients: `loss.backward()` [S18, S17, unofficial].
        *   Update weights: `optimizer.step()` [S18, S17, unofficial].
        *   Update learning rate according to the scheduler [S5, unofficial][S8][S11, unofficial].
    *   **Logging and Checkpointing:**
        *   Log loss and PSNR every $i_{print}$ steps [S2, S11, unofficial].
        *   Render validation images every $i_{img}$ steps [S2, S11, unofficial].
        *   Save checkpoint every $i_{weights}$ steps [S2, S11, unofficial].

**Formulas:**

*   **MSE Loss:**
    $$ \mathcal{L}_{MSE} = \text{mean}((\hat{\mathbf{c}} - \mathbf{c}_{gt})^2) $$
    Defined as: `tf.reduce_mean(tf.square(x - y))` or `torch.nn.functional.mse_loss(output, pixels)` [S14, S3, unofficial].

*   **PSNR:**
    $$ \text{PSNR} = -10 \log_{10}(\mathcal{L}_{MSE}) $$
    Defined as: `-10 * log10(x)` [S14, unofficial].

*   **Learning Rate Decay (Exponential):**
    $$ lr_{new} = lr_{initial} \times (\text{decay\_rate})^{\lfloor \text{global\_step} / \text{decay\_steps} \rfloor} $$
    Where `decay_steps` = `args.lrate_decay` * 1000 and `decay_rate` = 0.1 [S5, S18, unofficial].

*   **Learning Rate Scheduler (Lambda):**
    $$ lr(\text{epoch}) = lr_{initial} \times \gamma^{\lfloor \text{epoch} / \text{step\_size} \rfloor} $$
    Where $\gamma$ is `lr_scheduler_gamma` and `step_size` is `lr_scheduler_step_size` [S8][S10, unofficial].

**Parameters with Typical Values:**
*   $N_{rand}$ (Batch size in rays): 1024 or 2048 [S5, S11, S14, unofficial].
*   $N_{iters}$ (Total iterations): 100,000 [S11, unofficial] or 1,000,000 [S14, unofficial].
*   Learning Rate ($lr$): $5 \times 10^{-4}$ [S11, unofficial] or 0.01 [S17, unofficial].
*   Decay Rate: 0.1 [S5, S18, unofficial].
*   Decay Steps: `args.lrate_decay` * 1000 [S5, S18, unofficial].
*   Positional Encoding Frequency ($L$): 10 for position, 4 for direction [S11, unofficial][S20].
*   Chunk Size: Limits VRAM usage during forward pass (e.g., 8192 rays) [S5, S7, unofficial].

**Edge Cases and Pitfalls:**
*   **VRAM Overflow:** Large batch sizes or high-resolution images can exceed GPU memory. Mitigated by chunking rays during the forward pass (`chunk=args.chunk`) [S5, S7, unofficial].
*   **Gradient Computation Order:** Higher-order derivatives (e.g., for logging gradients or Laplacians) must be computed *before* `loss.backward()` because the backward pass frees the autograd graph [S3, unofficial].
*   **Checkpoint Resumption:** When resuming, the learning rate scheduler's last epoch must be set to `start_epoch - 1` to ensure correct scheduling [S9, unofficial]. If a checkpoint is loaded but the step is 0, it is considered empty and should trigger an error [S15, unofficial].
*   **Instability in Growing Networks:** Directly growing a randomly initialized sub-network causes instability. Alpha linear weights of grown sub-networks should be initialized to match the parent network [S1].
*   **Empty Checkpoints:** Loading a non-existent checkpoint logs an error and returns exit code 2; loading an empty checkpoint (step 0) returns exit code 3 [S15, unofficial].

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
        config: Configuration object with hyperparameters.
    """
    # 1. Initialization
    optimizer = optim.Adam(model.parameters(), lr=config.lr)
    # Example scheduler: Exponential Decay
    scheduler = optim.lr_scheduler.StepLR(optimizer, step_size=config.decay_steps, gamma=config.decay_rate)
    
    # Precompute rays if batching is enabled
    if config.use_batching:
        rays_rgb = precompute_rays(train_dataset) # Shape: [N_pixels, 3, 3]
        rays_rgb = rays_rgb.flatten(0, 1) # Flatten to [N_pixels, 3, 3]
        i_batch = 0
    
    n_iters = config.n_iters
    log_every = config.log_every
    render_every = config.render_every
    save_every = config.save_every
    
    model.train()
    
    for i in range(n_iters):
        start_time = time.time()
        
        # 2. Ray Sampling
        if config.use_batching:
            # Slice pre-computed rays
            batch_rays = rays_rgb[i_batch:i_batch + config.batch_size]
            i_batch += config.batch_size
            if i_batch >= rays_rgb.shape[0]:
                # Shuffle and reset
                indices = torch.randperm(rays_rgb.shape[0])
                rays_rgb = rays_rgb[indices]
                i_batch = 0
        else:
            # Random sampling from current image
            img_idx = torch.randint(0, len(train_dataset), (1,)).item()
            rays_o, rays_d, target_s = sample_rays_from_image(train_dataset[img_idx], config.N_rand)
            # Stack into batch tensor [3, N_rand, 3] (origin, dir, target)
            batch_rays = torch.stack([rays_o, rays_d, target_s], dim=0)
            
        # Move to device
        batch_rays = batch_rays.to(config.device)
        
        # 3. Forward Pass
        # Split batch_rays into origins/directions and targets
        rays_o_d = batch_rays[:2, :, :] # [2, N_rand, 3]
        target_s = batch_rays[2, :, :]  # [N_rand, 3]
        
        # Render rays
        # render_rays returns rgb, depth, acc, extras (including coarse rgb if applicable)
        rgb, depth, acc, extras = render_rays(
            model, 
            rays_o_d[0], # origins
            rays_o_d[1], # directions
            near=config.near, 
            far=config.far, 
            chunk=config.chunk
        )
        
        # 4. Loss Computation
        # MSE Loss
        img_loss = torch.mean((rgb - target_s) ** 2)
        loss = img_loss
        
        # Add coarse loss if available
        if 'rgb0' in extras:
            img_loss0 = torch.mean((extras['rgb0'] - target_s) ** 2)
            loss = loss + img_loss0
            
        # Add regularization if configured
        if config.tv_loss_weight > 0 and i > 1000:
            loss = loss + config.tv_loss_weight * compute_tv_loss(model)
            
        # 5. Backpropagation and Update
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
        scheduler.step()
        
        # 6. Logging and Checkpointing
        if i % log_every == 0:
            psnr = -10.0 * torch.log10(img_loss)
            print(f"Iter {i}: Loss {img_loss.item():.6f}, PSNR {psnr.item():.2f}")
            
        if i % render_every == 0:
            render_validation_image(model, val_dataset, i)
            
        if i % save_every == 0:
            save_checkpoint(model, optimizer, i, config.checkpoint_dir)
            
        end_time = time.time()
        # print(f"Iter {i} took {end_time - start_time:.2f}s")
```

## Subtopics

### [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md)
The differentiable forward pass that synthesizes RGB images from a NeRF during optimization. It combines ray generation, adaptive marching, neural network evaluation, and alpha compositing into a single differentiable pass.

### [Training Configuration](2332-training-configuration.md)
Defines hyperparameters, optimization routines, and architectural constraints (learning rate schedules, loss weights, batch sampling) critical for convergence.

### [Neural Volume Scene Setup](4168-neural-volume-scene-setup.md)
Constructs the scene graph, initializing density grids and spherical harmonics coefficients, binding neural network parameters to the scene.

### [MVSNeRF](6197-mvsnerf.md)
A feedforward method for fast, generalizable reconstruction from multi-view stereo images, reducing training time by learning a data prior.

### [Camera Pose and Parameter Optimization](7611-camera-pose-and-parameter-optimization.md)
Jointly optimizes NeRF weights and camera poses/intrinsics when poses are unknown or inaccurate.

### [Specific Model Architectures](8585-specific-model-architectures.md)
Architectural modifications to the NeRF function $f_\theta$ (e.g., discriminators, diffusion priors) rather than the sampling strategy.

### [Regularization and Loss Functions](8586-regularization-and-loss-functions.md)
Mathematical terms added to the photometric loss to constrain optimization (e.g., geometric baselines, normalizing flows).

### [Progressive Inpainting-and-Erasing](8658-progressive-inpainting-and-erasing.md)
A strategy for single-view NeRF that computes a mask of conflicted regions and eliminates them from training.

### [KNN-NeRF](8672-knn-nerf.md)
A baseline using sparse 3D point clouds as supervision, assigning nearest point colors to rays.

### [Hard Sample Mining](8673-hard-sample-mining.md)
Selectively updates the network based on pixel-level error to improve training efficiency.

### [Progressive and Dynamic Scene Growth](8869-progressive-and-dynamic-scene-growth.md)
Incrementally expands the neural representation's capacity or dataset to handle large-scale scenes.

### [Adaptive Ray Sampling](8870-adaptive-ray-sampling.md)
Optimizes sample point distribution along rays to concentrate resources on complex regions.

### [Sparse View and Initialization Methods](9394-sparse-view-and-initialization-methods.md)
Addresses quality degradation in sparse-view settings using external geometric scaffolds or semantic masks.

### [Scene Representation and Partitioning](9444-scene-representation-and-partitioning.md)
Decomposes the 3D domain into disjoint regions for distributed training and scalability.

### [Zip-NeRF Training Procedure](9538-zip-nerf-training-procedure.md)
A two-stage method for inverse rendering with radiance caches, using a control variate strategy.

### [DPPE (Dense Pose Estimation in a Plenoxels Environment)](9640-dppe-dense-pose-estimation-in-a-plenoxels-environment.md)
6-DoF monocular RGB-only pose estimation using numerical gradient approximation in Plenoxels.

### [NeRV Training](10253-nerv-training.md)
Optimizes a representation decoupling geometry, appearance, and lighting via a separate visibility MLP.

### [NeRV Rendering](10264-nerv-rendering.md)
Computes pixel color using physically-based shading and indirect lighting estimation.

### [Specific Model Architectures and Extensions](10278-specific-model-architectures-and-extensions.md)
Modifications for unbounded environments, defocus blur, or large-scale drone imagery.

### [Aug-NeRF](10294-aug-nerf.md)
Injects adversarial perturbations to implicitly regularize the model with physical grounds.

### [Cross-Guided Optimization of Radiance Fields (CROP)](10371-cross-guided-optimization-of-radiance-fields-crop.md)
Performs multi-view image super-resolution during optimization to improve high-resolution synthesis.

### [Truncated Depth-Based Sampling](10613-truncated-depth-based-sampling.md)
Uses coarse monocular depth priors to guide point sampling via a truncated normal distribution.

### [Diffusion-based Regularization for NeRF](10661-diffusion-based-regularization-for-nerf.md)
Incorporates a denoising diffusion model as a prior for density and color fields.

### [Multiplex Training via S3IM](10894-multiplex-training-via-s3im.md)
Combines pixel-wise loss with Stochastic Structural SIMilarity (S3IM) loss.

### [S3IM (Stochastic Structural SIMilarity)](10895-s3im-stochastic-structural-similarity.md)
A loss function capturing nonlocal structural similarity by processing multiple pixels as a whole set.

### Supersampling Baseline
A baseline method for evaluating NeRF on multiscale data by training on full-resolution images and using brute-force supersampling during evaluation, or training on multiscale data with supersampling during both training and evaluation [S21]. For every pixel, multiple jittered rays are cast, rendered, and averaged [S21]. It relies on 'oracle' scale information and performs worse than mip-NeRF while being significantly more expensive [S21].

### Two-stage training for proposers
A training schedule split into two equal-duration phases to stabilize learning of a trainable proposer network in NeRF-ID [S22, unofficial]. Phase 1 trains the proposer to mimic a heuristic proposer using L2 matching loss. Phase 2 trains all components end-to-end with a linear warmup for 1,000 steps [S22, unofficial]. This two-stage approach is consistently effective where training from scratch often fails [S22, unofficial].

### Multi-view SDS
A plug-and-play method that creates multi-view supervision during SDS optimization to address the Janus problem caused by one-by-one view sampling [S23]. Rendered images from 4 sampled views are stitched into a 2x2 image, and gradients are calculated under the condition of the multi-view normal map [S23]. It converts 3D assets into differentiable representations (NeRF, 3DGS, DMTet) through rendering supervision [S23].

### Guided NeRF Optimization for MVS
A multi-view depth estimation method that utilizes conventional reconstruction (sparse SfM+MVS) and learning-based priors over NeRF [S24]. It performs test-time optimization over implicit volumes to eliminate the need for accurate pixel matching across views [S24]. Step 1 runs COLMAP to acquire sparse depth maps. Step 2 adapts a monocular depth network to the target scene using scale-invariant loss [S24].

### Surface-aware density formulation
A method to extract high-quality surfaces from NeRF by modeling geometry as a distance function or occupancy field and converting its output to density using a probability density function [S25]. It incorporates an eikonal term to encourage geometric consistency, enforcing that the gradient norm of the learned SDF is equal to one [S25]. Density distribution: $\Phi_S(x) = \frac{se^{-sx}}{1+e^{-sx}}$, where $s$ is a scaling parameter and $x$ is the SDF or occupancy value [S25].

### Dynamically-informed iNeRF Estimator
A state estimation method that uses an iNeRF initialized with a dynamics prior, optimized only via photometric loss, acting as a recursive Bayesian filter with infinite state covariance [S26]. The process loss is set to 0 [S26]. An iNeRF estimator initialized without a dynamics prior will very quickly diverge [S26].

### Volumetric Supervision
A training strategy that uses high-quality volumetric reconstructions from conventional tomography (available at start/end times) to supervise the neural network training at intermediate times with sparse projections [S27, unofficial]. Step 1 obtains rich projection data at t=0 and t=1. Step 2 reconstructs 3D volumes using conventional tomography. Step 3 computes Ccor between the network's predicted volume and the CT-reconstructed volume. Step 4 minimizes $L_{vol} = -Ccor$ [S27, unofficial]. Volumetric loss $L_{vol} = -Ccor(y, \hat{y})$, where $Ccor$ is the normalized correlation coefficient between the target CT volume $y$ and the predicted volume $\hat{y}$ [S27, unofficial].

## Operating characteristics

*   **Time Complexity:** $O(N_{iters} \cdot N_{rand} \cdot K)$, where $N_{iters}$ is the number of iterations, $N_{rand}$ is the batch size in rays, and $K$ is the number of samples per ray [S5, S14, unofficial].
*   **Memory Complexity:** $O(N_{rand} \cdot K)$ for the forward pass if chunking is used, or $O(N_{pixels} \cdot K)$ if full images are rendered [S5, S14, unofficial]. Storing intermediate states for reverse-mode AD requires $O(N_{pixels} \cdot K)$ memory [S5, unofficial].
*   **Measured Performance:** Training on 128x128 images for 250 epochs [S4, unofficial]. On a dataset of 199 images, each epoch processes ~8M rays, with ~7,800 iterations per epoch for a batch size of 1024 [S17, unofficial].
*   **Scaling:** Larger batch sizes (e.g., $2^{14}$) hinder data loading performance and prevent full GPU occupancy, while smaller batch sizes (e.g., $2^{10}$) cap iterations per second [S15, unofficial].
*   **Strengths:** Stochastic optimization reduces compute per iteration compared to full-image rendering [S5, unofficial]. Differentiability allows end-to-end training of complex scene representations [S1, S8].
*   **Limitations:** High memory usage for large scenes or high-resolution images [S5, unofficial]. Slow convergence for high-frequency details without positional encoding or specific architectures [S20]. Blurry predictions if voxel grid resolution is coarse or advanced ray sampling is not used [S17, unofficial].
*   **When to Use:** When reconstructing a 3D scene from a set of 2D images with known poses.
*   **When Not to Use:** When camera poses are unknown (use [Camera Pose and Parameter Optimization](7611-camera-pose-and-parameter-optimization.md)) or when real-time inference is required without pre-training (use [MVSNeRF](6197-mvsnerf.md)).

## Background

The NeRF Training Loop was introduced by Mildenhall et al. in "NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis" (2020) [S5, S18, unofficial]. It builds on the concept of coordinate-based neural networks for implicit scene representation [S3, S18, unofficial]. The method was initially developed to solve the view synthesis problem by optimizing a continuous volumetric scene representation from multi-view images [S5, S18, unofficial]. It has since become the foundational training procedure for numerous extensions, including mip-NeRF [S21], Instant-NGP [S15, unofficial], and 3D Gaussian Splatting [S8].

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
[S27] Neural rendering enables dynamic tomography, https://doi.org/10.48550/arxiv.2410.20558, fetched 2026-10-26 (unofficial)
