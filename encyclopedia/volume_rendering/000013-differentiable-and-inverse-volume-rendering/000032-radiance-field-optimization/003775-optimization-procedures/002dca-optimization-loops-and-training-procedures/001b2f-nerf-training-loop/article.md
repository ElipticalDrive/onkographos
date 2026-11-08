# NeRF Training Loop

Also called: IPE (Interatomic Positional Encoding), Interatomic Positional Encoding, NeRF Optimization Loop, NeRF Training Procedure, Neural Radiance Field (NeRF) Training

## Overview
The NeRF Training Loop is the iterative optimization procedure that reconstructs a Neural Radiance Field (NeRF) by minimizing the difference between rendered images and ground-truth observations [S5, S6, S14, unofficial]. It solves the inverse rendering problem by treating the scene as a continuous volumetric function parameterized by a neural network, optimizing this network via stochastic gradient descent [S5, S14, unofficial]. It sets itself apart from full-image rendering methods by sampling a small batch of random rays per iteration ($N_{rand}$) rather than all pixels, enabling efficient stochastic optimization and reducing memory usage [S5, S14, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A dataset of $N$ training images with known camera poses (extrinsics $c2w$ and intrinsics $K$), image dimensions $(H, W)$, and near/far bounds ($near, far$) [S2, S5, S14, unofficial].
*   **Outputs:** A trained neural network $f_\theta$ (and potentially a background model) that maps 3D coordinates $\mathbf{x}$ and view directions $\mathbf{d}$ to volume density $\sigma$ and color $\mathbf{c}$ [S5, S15, unofficial].

**Data Structures:**
*   **Rays Array (`rays_rgb`):** A pre-computed numpy array of shape $[(N-1) \cdot H \cdot W, 3, 3]$ containing all possible training rays [S5, S14, unofficial].
    *   Axis 0: Flattened pixel index.
    *   Axis 1: Channel type (0: ray origin, 1: ray direction, 2: target RGB color).
    *   Axis 2: XYZ/RGB components.
    *   Data type: `float32` [S5, S14, unofficial].
*   **Checkpoint:** A dictionary containing model states, optimizer states, and iteration counts.
    *   Keys: `'step'` (int), `'coarse'` (state_dict), `'fine'` (state_dict), `'optim'` (optimizer state_dict) [S11, unofficial].
    *   Alternative structure: `'model'`, `'optimizer'`, `'epoch'` [S10, unofficial].

**Numbered Steps:**

1.  **Initialization:**
    *   Load training and validation datasets [S8][S11, unofficial].
    *   Initialize the NeRF model(s). For standard NeRF, this is an MLP with skip connections [S5, S6, unofficial]. For HashNeRF, initialize hash table weights [S7, unofficial]. For growing networks (Recursive-NeRF), initialize alpha linear weights of grown sub-networks to match the parent to avoid instability [S1].
    *   Initialize the optimizer (typically Adam or AdamW) with a learning rate $lr$ (default $5 \times 10^{-4}$ or $0.01$) [S5, unofficial][S8][S11, S17, unofficial].
    *   Initialize a learning rate scheduler (e.g., ExponentialDecay or CosineAnnealingLR) [S5, unofficial][S8][S11, unofficial].
    *   If using occupancy grids (e.g., Instant-NGP), initialize the `OccupancyDensityGrid` with resolution (e.g., 128) [S15, unofficial].
    *   [G] Set random seeds for reproducibility.

2.  **Pre-computation (Optional but common):**
    *   Construct the `rays_rgb` array by stacking ray origins, directions, and target RGB colors from all training images [S5, S14, unofficial].
    *   Shuffle the array to ensure random sampling order [S5, S11, unofficial].

3.  **Iteration Loop (for $i = 1$ to $N_{iters}$):**
    *   **Ray Sampling:**
        *   If batching is enabled, slice a batch of size $N_{rand}$ (typical values: 1024, 2048, or 4096) from `rays_rgb` [S5, S11, S14, S17, unofficial].
        *   Transpose the batch to shape $[3, N_{rand}, 3]$ (origin, direction, target) [S5, S14, unofficial].
        *   If batching is disabled, randomly select $N_{rand}$ pixel coordinates from a single training image (optionally applying a central crop if $i < precrop\_iters$) [S14, unofficial].
        *   [G] If `perturb > 0.`, add jitter to sample positions along the ray to break symmetry and improve surface reconstruction [S2, S5, unofficial].
    *   **Forward Pass (Training Ray Marching):**
        *   Query the neural network at sample points along the rays to get raw outputs (density $\sigma$ and color $\mathbf{c}$) [S5, S6, unofficial].
        *   Perform volume rendering (alpha compositing) to compute the predicted color $\hat{C}(\mathbf{r})$ [S5, S6, unofficial].
        *   If using a coarse-fine scheme, render with the coarse network, sample additional points based on weights, and render with the fine network [S5, S11, unofficial].
    *   **Loss Computation:**
        *   Compute the Mean Squared Error (MSE) between predicted RGB and target RGB:
            $$ \mathcal{L}_{rgb} = \frac{1}{N_{rand}} \sum_{j=1}^{N_{rand}} \| \hat{C}_j(\mathbf{r}_j) - C_j(\mathbf{r}_j) \|^2 $$
            [S14, S17, unofficial].
        *   If coarse output $\hat{C}_0$ is available, add coarse loss: $\mathcal{L}_{coarse} = MSE(\hat{C}_0, C)$ [S6, S11, unofficial].
        *   Add regularization terms if applicable (e.g., sparsity loss, Total Variation, Eikonal loss) [S7, unofficial][S8][S16, unofficial].
        *   Total Loss: $\mathcal{L} = \mathcal{L}_{rgb} + \mathcal{L}_{coarse} + \lambda_{reg} \mathcal{L}_{reg}$ [S11, S16, unofficial].
    *   **Backpropagation:**
        *   Zero gradients: `optimizer.zero_grad()` [S18, unofficial].
        *   Compute gradients: `loss.backward()` [S18, unofficial].
        *   [G] If logging higher-order derivatives (gradient, laplacian), compute them *before* `loss.backward()` as the backward pass frees the autograd graph [S3, unofficial].
        *   Update weights: `optimizer.step()` [S18, unofficial].
    *   **Learning Rate Update:**
        *   Update scheduler. For ExponentialDecay: $lr_{new} = lr_{old} \cdot decay\_rate^{(step / decay\_steps)}$ where $decay\_steps = args.lrate\_decay \times 1000$ and $decay\_rate = 0.1$ [S5, S18, unofficial].
    *   **Logging and Checkpointing:**
        *   Log loss and PSNR ($PSNR = -10 \log_{10}(MSE)$) every $i_{print}$ steps [S14, unofficial].
        *   Render validation images every $i_{img}$ steps [S2, unofficial].
        *   Save checkpoint every $i_{weights}$ steps [S2, S11, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim
import numpy as np

def train_nerf_loop(model, rays_rgb, target_colors, num_iters, batch_size, optimizer, scheduler, device):
    """
    Standard NeRF training loop.
    
    Args:
        model: Neural network module (NeRF).
        rays_rgb: Pre-computed array of shape [N_rays, 3, 3] (origin, dir, target).
        target_colors: Not used directly if rays_rgb contains targets, but kept for interface consistency.
        num_iters: Total number of optimization steps.
        batch_size: Number of rays per batch (N_rand).
        optimizer: PyTorch optimizer.
        scheduler: PyTorch learning rate scheduler.
        device: 'cpu' or 'cuda'.
    """
    model.train()
    n_rays = rays_rgb.shape[0]
    i_batch = 0
    
    for i in range(num_iters):
        # 1. Ray Sampling
        # Slice batch from pre-computed rays
        batch_indices = slice(i_batch, i_batch + batch_size)
        batch_rays = rays_rgb[batch_indices] # Shape: [batch_size, 3, 3]
        
        # Transpose to [3, batch_size, 3] for stacking origins/dirs/targets
        batch_rays = batch_rays.transpose(0, 1) 
        
        ray_origins = batch_rays[0].to(device) # [batch_size, 3]
        ray_dirs = batch_rays[1].to(device)    # [batch_size, 3]
        targets = batch_rays[2].to(device)     # [batch_size, 3]
        
        # Increment batch index and shuffle if exhausted
        i_batch += batch_size
        if i_batch + batch_size > n_rays:
            # Shuffle rays_rgb to ensure random sampling across epochs
            np.random.shuffle(rays_rgb)
            i_batch = 0
            
        # 2. Forward Pass
        optimizer.zero_grad()
        
        # Render rays (calls volume rendering equation)
        # Returns predicted RGB and potentially extras (depth, weights, coarse_rgb)
        outputs = model.render_ray_bundle(ray_origins, ray_dirs)
        predicted_rgb = outputs['rgb'] # [batch_size, 3]
        
        # 3. Loss Computation
        # MSE Loss
        mse_loss = nn.functional.mse_loss(predicted_rgb, targets)
        loss = mse_loss
        
        # Add coarse loss if available (coarse-fine scheme)
        if 'rgb_coarse' in outputs:
            coarse_loss = nn.functional.mse_loss(outputs['rgb_coarse'], targets)
            loss = loss + coarse_loss
            
        # Add regularization if defined in model
        if hasattr(model, 'get_regularization_loss'):
            reg_loss = model.get_regularization_loss(outputs)
            loss = loss + reg_loss
            
        # 4. Backpropagation
        loss.backward()
        
        # 5. Optimization Step
        optimizer.step()
        scheduler.step()
        
        # 6. Logging
        if i % 100 == 0:
            psnr = -10.0 * torch.log10(mse_loss)
            print(f"Step {i}: Loss={loss.item():.6f}, PSNR={psnr.item():.2f}")
            
        if i % 5000 == 0:
            torch.save({
                'step': i,
                'model_state_dict': model.state_dict(),
                'optimizer_state_dict': optimizer.state_dict(),
            }, f'checkpoint_{i}.pth')
```

**Parameters with Typical Values:**
*   $N_{rand}$ (Batch Size): 1024, 2048, or 4096 [S5, S11, S17, unofficial].
*   Learning Rate ($lr$): $5 \times 10^{-4}$ (Adam) or $0.01$ (SGD) [S11, S17, unofficial].
*   Decay Rate: 0.1 [S5, S18, unofficial].
*   Decay Steps: $args.lrate\_decay \times 1000$ [S5, S18, unofficial].
*   Positional Encoding Frequency ($L$): 10 for position, 4 for direction [S11, unofficial].
*   Total Iterations ($N_{iters}$): 100,000 to 1,000,000 [S11, S14, unofficial].

**Edge Cases and Pitfalls:**
*   **VRAM Overflow:** Rendering all rays at once exceeds GPU memory. Mitigated by chunking rays or using small batch sizes ($N_{rand}$) [S5, S15, unofficial].
*   **Instability in Growing Networks:** Randomly initialized sub-networks may collapse to zero density. Solution: Initialize sub-network weights to match the parent network [S1].
*   **Gradient Graph Freeing:** Higher-order derivatives (for logging/regularization) must be computed before `loss.backward()` [S3, unofficial].
*   **Empty Checkpoints:** Loading a checkpoint with step 0 should be treated as an error or re-initialization [S15, unofficial].
*   **Scale Ambiguity:** Without proper near/far bounds or regularization, the network may learn arbitrary scales [S2, S17, unofficial].

## Subtopics

### [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md)
The differentiable forward pass that synthesizes RGB images from a NeRF during optimization.

### [Training Configuration](2332-training-configuration.md)
Static and dynamic hyperparameters, optimization routines, and architectural constraints.

### [Mip-NeRF 360](3306-mip-nerf-360.md)
Extends NeRF to unbounded 360-degree scenes using cone-casting and contract mapping.

### [Neural Volume Scene Setup](4168-neural-volume-scene-setup.md)
Constructs the scene graph and initializes density grids and spherical harmonics.

### [MVSNeRF](6197-mvsnerf.md)
Fast, generalizable reconstruction from multi-view stereo using a feedforward network.

### [Camera Pose and Parameter Optimization](7611-camera-pose-and-parameter-optimization.md)
Jointly optimizes camera extrinsics/intrinsics with the scene representation.

### [Specific Model Architectures](8585-specific-model-architectures.md)
Variations in network topology (e.g., SIREN, HashGrid) to address speed or quality.

### [Regularization and Loss Functions](8586-regularization-and-loss-functions.md)
Priors (diffusion, flow, geometric) to constrain the solution space beyond photometric consistency.

### [Progressive Inpainting-and-Erasing](8658-progressive-inpainting-and-erasing.md)
Strategy for single-view NeRF to handle invisible region ambiguity.

### [NeRF-FF](8669-nerf-ff.md)
Mitigates degradation from out-of-focus regions using binary masks.

### [KNN-NeRF](8672-knn-nerf.md)
Baseline using sparse 3D point clouds for supervision.

### [Hard Sample Mining](8673-hard-sample-mining.md)
Selectively updates the network based on pixel-level error.

### [Progressive and Dynamic Scene Growth](8869-progressive-and-dynamic-scene-growth.md)
Incrementally expands model capacity or dataset.

### [Adaptive Ray Sampling](8870-adaptive-ray-sampling.md)
Dynamically adjusts sample point distribution along rays.

### [Drone-NeRF](8972-drone-nerf.md)
Block-parallel training for large-scale drone imagery.

### [Sparse View and Initialization Methods](9394-sparse-view-and-initialization-methods.md)
Techniques for training on sparse input views.

### [Scene Representation and Partitioning](9444-scene-representation-and-partitioning.md)
Decomposing large scenes into localized sub-representations.

### [Zip-NeRF Training Procedure](9538-zip-nerf-training-procedure.md)
Two-stage optimization for physically-based scene models and radiance caches.

### [DPPE (Dense Pose Estimation in a Plenoxels Environment)](9640-dppe-dense-pose-estimation-in-a-plenoxels-environment.md)
6-DoF pose estimation using Plenoxels.

### ReconFusion
Uses a Diffusion Prior to constrain unobserved views during training [S21, unofficial].

### Supersampling Baseline
A baseline method for evaluating NeRF on multiscale data by training on full-resolution images and using brute-force supersampling during evaluation, or training on multiscale data with supersampling during both training and evaluation [S22]. For every pixel, it casts multiple jittered rays sampled uniformly at random through the spatial footprint of the pixel, renders each ray with NeRF, and uses the mean of those rendered values as the predicted color in the loss function [S22]. This baseline performs worse than mip-NeRF and is significantly more expensive during both training and evaluation [S22]. It relies on 'oracle' scale information that does not exist for most datasets [S22].

### Two-stage training for proposers
A training schedule split into two equal-duration phases to stabilize learning of a trainable proposer network in NeRF-ID [S23, unofficial]. Phase 1 trains the learnt proposer to mimic the heuristic proposer by minimizing the L2 matching loss between their proposal sets [S23, unofficial]. Phase 2 trains all network components end-to-end, resetting the optimizer state and performing a linear warmup for 1,000 steps to reduce shock from switching regimes [S23, unofficial]. Training from scratch sometimes works but often fails; two-stage training is consistently effective [S23, unofficial].

### Multi-view SDS
A plug-and-play method that creates multi-view supervision during SDS optimization to address the Janus problem caused by one-by-one view sampling [S24]. Rendered images from 4 sampled views are stitched into a large 2x2 image, and gradients are calculated under the condition of the multi-view normal map [S24]. The method converts a 3D asset into differentiable representations including NeRF, 3DGS, and DMTet through rendering supervision, which are then optimized to improve the quality of the 3D asset [S24].

### Guided NeRF Optimization for MVS
A multi-view depth estimation method that utilizes conventional reconstruction (sparse SfM+MVS) and learning-based priors over Neural Radiance Fields (NeRF) [S25]. It performs test-time optimization directly over implicit volumes to eliminate the need for accurate pixel matching across views, addressing the shape-radiance ambiguity of NeRF in indoor scenes [S25]. Step 1: Run COLMAP on the test scene to acquire per-view sparse depth maps by projecting fused 3D point clouds after multi-view stereo [S25]. Step 2: Adapt a monocular depth network to the target scene by finetuning it on the sparse depth maps from COLMAP using a scale-invariant loss [S25].

### Surface-aware density formulation
A method to extract high-quality surfaces from NeRF by modeling geometry as a distance function or occupancy field and converting its output to density using a probability density function [S26]. It incorporates an eikonal term to encourage geometric consistency, enforcing that the gradient norm of the learned SDF is equal to one [S26]. The density distribution is modeled as $\Phi_S(x) = \frac{se^{-sx}}{1+e^{-sx}}$, where $s$ is a scaling parameter and $x$ is the SDF or occupancy value [S26].

## Operating characteristics
*   **Time Complexity:** $O(N_{iters} \cdot N_{rand} \cdot N_{samples})$ per iteration, where $N_{samples}$ is the number of points sampled per ray [S5, S14, unofficial].
*   **Memory Complexity:** $O(N_{rand} \cdot N_{samples})$ for the forward pass, plus $O(N_{params})$ for the model and optimizer states [S5, unofficial].
*   **Measured Performance:** Training on 199 images (down-sampled by factor of 4) with batch size 1024 results in ~7,800 iterations per epoch, 39,000 total for 5 epochs [S17, unofficial]. Training PSNR increased from 10.62 to 18.97 over 20 epochs in a Keras implementation [S12].
*   **Scaling:** Larger batch sizes (e.g., $2^{14}$) hinder data loading performance and prevent full GPU occupancy, while smaller batch sizes (e.g., $2^{10}$) cap iterations per second [S15, unofficial].
*   **Strengths:** Efficient stochastic optimization; flexible scene representation; differentiable end-to-end training.
*   **Limitations:** Slow convergence for high-frequency details; prone to "floaters" and overfitting without regularization; memory-intensive for large scenes [S1][S17, unofficial].
*   **When to Use:** When high-quality novel view synthesis is required from multi-view images and computational resources allow for iterative optimization.
*   **When Not to Use:** For real-time applications requiring immediate inference without training, or when input views are extremely sparse and unstructured without additional priors.

## Background
The NeRF Training Loop was introduced by Mildenhall et al. in "NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis" (2020) [S5, S18, unofficial]. It builds on the concept of coordinate-based neural networks for implicit scene representation [S3, unofficial][S20]. The method was initially implemented in TensorFlow [S5, unofficial] and later ported to PyTorch [S6, S7, S11, unofficial]. It has become the standard baseline for inverse volume rendering and has inspired numerous variants including Mip-NeRF [S22], Instant-NGP [S15, unofficial], and 3D Gaussian Splatting [S8].

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
[S21] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
[S22] Mip-NeRF: A Multiscale Representation for Anti-Aliasing Neural Radiance Fields, https://doi.org/10.1109/iccv48922.2021.00580, fetched 2026-10-26
[S23] NeRF in detail: Learning to sample for view synthesis, https://doi.org/10.48550/arxiv.2106.05264, fetched 2026-10-27 (unofficial)
[S24] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S25] Depth-Guided Optimization of Neural Radiance Fields for Indoor Multi-View Stereo, https://doi.org/10.1109/tpami.2023.3263464, fetched 2026-10-27
[S26] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
