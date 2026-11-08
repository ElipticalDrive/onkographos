# NeRF Training Loop

Also called: IPE (Interatomic Positional Encoding), Interatomic Positional Encoding, NeRF Optimization Loop, NeRF Training Procedure, Neural Radiance Field (NeRF) Training

## Overview
The NeRF Training Loop is the iterative optimization procedure that learns a continuous volumetric scene representation by minimizing the difference between rendered images and ground truth observations [S5, S6, S14, unofficial]. It solves the inverse rendering problem by updating the parameters of a coordinate-based neural network (MLP) via gradient descent [S5, unofficial]. It sets itself apart from its siblings by establishing the fundamental stochastic optimization cycle—ray sampling, differentiable volume rendering, and loss minimization—that serves as the baseline for all advanced NeRF variants [S8][S14, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ training images with known camera poses (extrinsics $C2W$ and intrinsics $K$) and resolution $(H, W)$.
*   **Outputs:** A trained neural network $f_\theta$ parameterizing the radiance field, where $\theta$ are the optimized weights.

**Data Structures:**
*   **Rays:** A ray is defined by origin $\mathbf{o}$ and direction $\mathbf{d}$. In implementation, rays are often pre-computed and stored in a flat array `rays_rgb` of shape $[(N-1)HW, 3, 3]$ (axis 0: pixel index; axis 1: [origin, direction, target_rgb]; axis 2: components) [S5, S14, unofficial].
*   **Checkpoint:** A dictionary containing model state (`coarse`, `fine`), optimizer state (`optim`), and current step count [S11, unofficial].

**Numbered Steps:**

1.  **Initialization:**
    *   Load training and validation datasets [S8][S11, unofficial].
    *   Initialize the NeRF model(s). For standard NeRF, this is a single MLP [S5, unofficial]. For two-stage methods (e.g., Mip-NeRF), initialize coarse and fine MLPs [S11, unofficial].
    *   Initialize the optimizer (typically Adam) with learning rate $lr$ [S5, S11, unofficial].
    *   Initialize the learning rate scheduler (e.g., Exponential Decay or Cosine Annealing) [S5, S11, unofficial].
    *   Precompute all possible rays from training images into `rays_rgb` and shuffle them [S5, S11, unofficial].

2.  **Iteration Loop (for $i = 1$ to $N_{iters}$):**
    *   **Ray Sampling:**
        *   Select a batch of $N_{rand}$ rays from `rays_rgb` [S5, S14, unofficial].
        *   If batching is enabled, slice the pre-computed array; otherwise, randomly select pixel coordinates from a random training image [S14, unofficial].
        *   Extract ray origins $\mathbf{o}$, directions $\mathbf{d}$, and target colors $\mathbf{C}_{gt}$ [S7, S14, unofficial].
    *   **Forward Pass (Differentiable Volume Rendering):**
        *   Sample $K$ points along each ray within near/far bounds [S5, S13, unofficial].
        *   Query the neural network to obtain density $\sigma$ and color $\mathbf{c}$ for each point [S5, unofficial].
        *   Compute the rendered color $\hat{\mathbf{C}}$ using the volume rendering equation (see Parent article) [S5, S6, unofficial].
        *   If using a coarse-fine scheme, compute coarse output $\hat{\mathbf{C}}_{coarse}$ and fine output $\hat{\mathbf{C}}_{fine}$ [S11, unofficial].
    *   **Loss Computation:**
        *   Compute the Mean Squared Error (MSE) between predicted and target colors:
            $$ \mathcal{L}_{rgb} = \frac{1}{N_{rand}} \sum_{j=1}^{N_{rand}} \|\hat{\mathbf{C}}_j - \mathbf{C}_{gt,j}\|^2 $$
            [S14, S17, unofficial].
        *   If coarse output is available, add coarse loss: $\mathcal{L}_{coarse} = \text{MSE}(\hat{\mathbf{C}}_{coarse}, \mathbf{C}_{gt})$ [S11, S18, unofficial].
        *   Total Loss $\mathcal{L} = \mathcal{L}_{fine} + \lambda_{coarse} \mathcal{L}_{coarse}$ (if applicable) [S11, unofficial].
    *   **Backpropagation and Update:**
        *   Zero gradients: `optimizer.zero_grad()` [S18, unofficial].
        *   Compute gradients: `loss.backward()` [S18, unofficial].
        *   Update weights: `optimizer.step()` [S18, unofficial].
        *   Update learning rate according to the scheduler [S5, S11, unofficial].
    *   **Logging and Checkpointing:**
        *   Log loss and PSNR every $I_{print}$ steps [S2, S11, unofficial].
        *   Render validation images every $I_{img}$ steps [S2, S11, unofficial].
        *   Save checkpoint every $I_{weights}$ steps [S2, S11, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_nerf_loop(model, optimizer, scheduler, rays_rgb, targets, 
                    n_iters=100000, batch_size=1024, log_every=100, 
                    save_every=10000, render_every=5000):
    """
    Standard NeRF training loop.
    
    Args:
        model: Neural Radiance Field MLP.
        optimizer: Adam optimizer.
        scheduler: Learning rate scheduler (e.g., ExponentialDecay).
        rays_rgb: Pre-computed array of shape [Total_Rays, 3, 3] 
                  (origin, direction, target_rgb).
        targets: Not used if rays_rgb contains targets, but kept for interface consistency.
        n_iters: Total number of optimization steps.
        batch_size: Number of rays per iteration (N_rand).
        log_every: Frequency of logging metrics.
        save_every: Frequency of saving checkpoints.
        render_every: Frequency of rendering validation images.
    """
    total_rays = rays_rgb.shape[0]
    ray_indices = np.arange(total_rays)
    
    for step in range(1, n_iters + 1):
        # 1. Ray Sampling
        # Randomly select batch_size unique indices
        indices = np.random.choice(total_rays, size=batch_size, replace=False)
        
        # Extract origins, directions, and targets
        # rays_rgb shape: [Total_Rays, 3, 3] -> [Origin, Dir, Target]
        batch_data = rays_rgb[indices] 
        rays_o = batch_data[:, 0, :] # [B, 3]
        rays_d = batch_data[:, 1, :] # [B, 3]
        target_rgb = batch_data[:, 2, :] # [B, 3]
        
        # 2. Forward Pass
        # Convert to tensors if necessary
        rays_o_tensor = torch.tensor(rays_o, dtype=torch.float32)
        rays_d_tensor = torch.tensor(rays_d, dtype=torch.float32)
        target_tensor = torch.tensor(target_rgb, dtype=torch.float32)
        
        # Render rays (calls differentiable volume rendering)
        # Returns predicted RGB, and optionally coarse RGB if two-stage
        pred_rgb, extras = model.render(rays_o_tensor, rays_d_tensor)
        
        # 3. Loss Computation
        # Standard MSE Loss
        loss = torch.nn.functional.mse_loss(pred_rgb, target_tensor)
        
        # Add coarse loss if available (e.g., from two-stage training)
        if 'rgb_coarse' in extras:
            loss_coarse = torch.nn.functional.mse_loss(extras['rgb_coarse'], target_tensor)
            loss = loss + loss_coarse
            
        # 4. Backpropagation and Update
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
        
        # 5. Scheduler Update
        scheduler.step()
        
        # 6. Logging and Checkpointing
        if step % log_every == 0:
            psnr = -10.0 * torch.log10(loss).item()
            print(f"Step {step}: Loss={loss.item():.6f}, PSNR={psnr:.2f}")
            
        if step % save_every == 0:
            save_checkpoint(model, optimizer, step)
            
        if step % render_every == 0:
            render_validation_image(model, validation_data)

```

**Parameters with Typical Values:**
*   $N_{iters}$: 100,000 to 1,000,000 [S11, S14, unofficial].
*   $N_{rand}$ (Batch Size): 1024 or 2048 [S5, S11, unofficial].
*   Learning Rate ($lr$): $5 \times 10^{-4}$ [S11, unofficial].
*   Decay Rate: 0.1 [S6, S18, unofficial].
*   Decay Steps: $args.lrate\_decay \times 1000$ [S5, S18, unofficial].
*   Positional Encoding Frequency ($L$): 10 for position, 4 for direction [S11, unofficial].

**Edge Cases and Pitfalls:**
*   **VRAM Overflow:** Large batches or high-resolution rays can exceed GPU memory. Mitigated by chunking rays during the forward pass [S5, S15, unofficial].
*   **Resuming Training:** When loading a checkpoint, the scheduler's last epoch must be set to `start_epoch - 1` to ensure correct decay continuation [S9, unofficial].
*   **Empty Checkpoints:** If a checkpoint is loaded but the step count is 0, it is treated as invalid/empty [S15, unofficial].
*   **Gradient Computation Order:** Higher-order derivatives (e.g., for logging gradients) must be computed *before* `loss.backward()` to avoid freeing the autograd graph [S3, unofficial].

## Subtopics

### [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md)
The differentiable forward pass that synthesizes RGB images from a NeRF during optimization. It combines ray generation, adaptive marching, neural evaluation, and alpha compositing into a single gradient-capable step.

### [Training Configuration](2332-training-configuration.md)
Defines static hyperparameters, optimizer settings, and structural constraints (e.g., learning rates, batch sizes) to stabilize optimization. It differs from dynamic runtime sequences [1522-training-loop-setup.md] and specific loss penalties [6281-regularization-and-priors.md].

### [Mip-NeRF 360](3306-mip-nerf-360.md)
Integrates non-linear scene parameterization, online distillation between two MLPs, and distortion-based regularization to handle unbounded, 360-degree scenes. It reduces MSE by 57% compared to standard mip-NeRF.

### [Neural Volume Scene Setup](4168-neural-volume-scene-setup.md)
Constructs a Mitsuba scene with a volumetric cube and custom integrator, binding neural parameters directly to the scene graph.

### [MVSNeRF](6197-mvsnerf.md)
Uses a feedforward network to predict radiance fields from multi-view stereo images, eliminating per-scene iterative optimization.

### [Camera Pose and Parameter Optimization](7611-camera-pose-and-parameter-optimization.md)
Treats camera extrinsics and intrinsics as differentiable variables optimized jointly with the scene, rather than relying on pre-estimated poses [7405-nerf-with-unknown-camera-parameters.md].

### [Specific Model Architectures](8585-specific-model-architectures.md)
Defines network topologies (e.g., SIREN, HashGrid) that parameterize $f_\theta(\mathbf{x}, \mathbf{d}) \to (\sigma, \mathbf{c})$.

### [Regularization and Loss Functions](8586-regularization-and-loss-functions.md)
Addresses under-constrained problems by adding priors (e.g., eikonal, diffusion) to the objective function beyond photometric consistency.

### [Progressive Inpainting-and-Erasing](8658-progressive-inpainting-and-erasing.md)
A single-view NeRF strategy that computes a mask of conflicted regions (visible vs. invisible) and erases them from training to avoid geometry conflicts.

### [NeRF-FF](8669-nerf-ff.md)
Mitigates degradation from out-of-focus images by estimating binary masks to omit blurred pixels during training.

### [KNN-NeRF](8672-knn-nerf.md)
A baseline using sparse 3D point clouds for supervision, assigning nearest-point colors to rays.

### [Hard Sample Mining](8673-hard-sample-mining.md)
Selectively updates the network based on pixel-level error, subsampling a "hard" minibatch to improve efficiency.

### [Progressive and Dynamic Scene Growth](8869-progressive-and-dynamic-scene-growth.md)
Incrementally expands model capacity or dataset during training to resolve occlusions and capture high-frequency details.

### [Adaptive Ray Sampling](8870-adaptive-ray-sampling.md)
Dynamically adjusts sample point distribution along rays to concentrate computation in complex regions.

### [Drone-NeRF](8972-drone-nerf.md)
Uses block-parallel training for large-scale drone imagery, handling exposure variations and shadows via appearance embeddings and expanded bounding boxes.

### [Sparse View and Initialization Methods](9394-sparse-view-and-initialization-methods.md)
Decouples initialization from random assumptions, using external geometric scaffolds or semantic masks for sparse inputs (15–20 views).

### [Scene Representation and Partitioning](9444-scene-representation-and-partitioning.md)
Decomposes large scenes into localized sub-representations for parallelism and scalability.

### [Zip-NeRF Training Procedure](9538-zip-nerf-training-procedure.md)
A two-stage method jointly training a physical scene model, environment map, and radiance caches using a control variate strategy.

### [DPPE (Dense Pose Estimation in a Plenoxels Environment)](9640-dppe-dense-pose-estimation-in-a-plenoxels-environment.md)
Uses numerical gradient approximation on Plenoxels to recover 6-DoF camera poses by minimizing photometric errors.

#### ReconFusion
Uses a Diffusion Prior to constrain unobserved views during training [S21, unofficial].

#### Supersampling Baseline
A baseline method evaluating NeRF on multiscale data by training on full-resolution images and using brute-force supersampling (jittered rays per pixel) during evaluation [S22]. It relies on "oracle" scale information and performs worse than mip-NeRF while being more expensive [S22].

#### Two-stage training for proposers
A schedule split into two phases to stabilize a trainable proposer network in NeRF-ID: Phase 1 mimics a heuristic proposer via L2 matching loss; Phase 2 trains all components end-to-end with a 1,000-step linear warmup [S23, unofficial].

#### Multi-view SDS
Creates multi-view supervision during SDS optimization by stitching 4 rendered views into a 2x2 image to address the Janus problem [S24].

#### Guided NeRF Optimization for MVS
Performs test-time optimization over implicit volumes using sparse SfM+MVS depth maps and a finetuned monocular depth network to eliminate pixel matching requirements [S25].

#### Surface-aware density formulation
Extracts high-quality surfaces by modeling geometry as an SDF and converting it to density using $\Phi_S(x) = \frac{se^{-sx}}{1+e^{-sx}}$, incorporating an eikonal term to enforce gradient norm = 1 [S26].

## Operating characteristics
*   **Time Complexity:** $O(N_{iters} \cdot N_{rand} \cdot K)$, where $N_{iters}$ is iterations, $N_{rand}$ is batch size, and $K$ is samples per ray [S5, S14, unofficial].
*   **Memory Complexity:** $O(N_{rand} \cdot K)$ for the forward pass graph; reverse-mode AD requires storing $O(kn)$ intermediates [S5, unofficial].
*   **Performance:** Standard NeRF trains for ~100,000–1,000,000 steps [S11, S14, unofficial]. PSNR typically increases from ~10 dB to ~18–20 dB over 20–250 epochs depending on resolution and dataset [S4, unofficial][S12].
*   **Strengths:** Simple to implement; differentiable end-to-end; produces high-quality novel views.
*   **Limitations:** Slow convergence; high VRAM usage for large batches; prone to overfitting without regularization [S1][S5, unofficial].
*   **When to Use:** Baseline for novel view synthesis; when ground truth poses are available.
*   **When Not to Use:** When real-time training is required (use MVSNeRF [6197-mvsnerf.md]); when poses are unknown (use Camera Pose Optimization [7611-camera-pose-and-parameter-optimization.md]); when views are sparse (use Sparse View Methods [9394-sparse-view-and-initialization-methods.md]).

## Background
Originated in the "NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis" paper by Mildenhall et al. (2020) [S5, S18, unofficial]. It established the paradigm of optimizing a coordinate MLP via stochastic gradient descent on volume-rendered images. The method has become the foundational training procedure for subsequent variants like Mip-NeRF, Instant-NGP, and 3D Gaussian Splatting (which adapted the loop for explicit representations) [S1, S8][S11, unofficial].

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
