# 3D Gaussian Splatting Training

Also called: 3DGS Learning, G Splats

## Overview
3D Gaussian Splatting Training is the optimization process that adjusts the parameters of a set of 3D Gaussian primitives to minimize the difference between rendered images and ground truth views [S2][S6, unofficial]. It solves the problem of reconstructing high-fidelity 3D scenes from 2D images by iteratively refining position, rotation, scale, opacity, and appearance (spherical harmonics) attributes [S5, unofficial]. Unlike static initialization methods, this method employs adaptive density control to dynamically manage the topology of the Gaussian set, splitting, cloning, or pruning primitives based on optimization metrics to balance reconstruction quality and computational cost [S2]. It sets itself apart from NeRF-based methods by offering significantly faster training and inference speeds while achieving comparable image quality [S2][S5, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ input images (or video frames) with known camera poses (rotation $R_{cw}$, translation $t_{cw}$, intrinsic matrix $K$) and initial 3D Gaussian estimates derived from Structure from Motion (SfM) [S2][S6, unofficial].
*   **Outputs:** Optimized 3D Gaussian parameters (positions $\mathbf{p}$, rotations $q$, scales $\mathbf{s}$, opacities $\alpha$, and spherical harmonic coefficients $h$) stored in a format such as PLY [S4, S6, unofficial].

**Data Structures:**
Each Gaussian primitive $i$ is defined by:
*   Position $\mathbf{p}_i \in \mathbb{R}^3$
*   Rotation $q_i \in \mathbb{R}^4$ (unit quaternion)
*   Scale $\mathbf{s}_i \in \mathbb{R}^3$
*   Opacity $\alpha_i \in [0, 1]$
*   Appearance parameters (spherical harmonic coefficients $h_i$) [Parent]

**Parameters:**
*   `numIters`: Total number of optimization steps. Default: 30,000 [S3, S4, unofficial].
*   `sh_degree`: Maximum spherical harmonics degree. Default: 3 [S3, S4, unofficial].
*   `shDegreeInterval`: Steps between increasing SH degree. Default: 1,000 [S3, S4, unofficial].
*   `densifyFrom`: Step number after which densification begins. Default: 500 [S3, unofficial].
*   `densifyUntil`: Step number after which densification stops. Default: -1 (calculated as $\min(15,000, \text{numIters} / 2)$) [S3, unofficial].
*   `refineEvery`: Number of steps between densification/pruning operations. Default: 500 [S3, unofficial].
*   `densify_grad_thresh`: Gradient threshold for densification. Default: 0.0002 [S4, unofficial].
*   `densify_size_thresh`: Size threshold for densification. Default: 0.01 [S4, unofficial].
*   `split_screen_size`: Screen size threshold for splitting Gaussians. Default: 0.05 [S4, unofficial].
*   `stop_screen_size_at`: Step at which screen-size based splitting stops. Default: 4,000 [S4, unofficial].
*   `ssim_weight`: Weight applied to SSIM loss. Default: 0.2 [S3, S4, unofficial].
*   `maxGaussians`: Maximum number of Gaussians. Default: 5,000,000 [S3, unofficial].
*   `downScaleFactor`: Initial dataset downscale factor. Default: 1.0 [S4, unofficial].
*   `num_downscales`: Number of resolution levels. Default: 2 [S4, unofficial].
*   `resolution_schedule`: Step at which resolution scaling changes. Default: 3,000 [S4, unofficial].

**Algorithm:**

1.  **Initialization:**
    *   Initialize positions $\mathbf{p}_i$ from SfM point clouds in normalized bounds $[-1, 1]$ [S2][S6, unofficial].
    *   Initialize scales $\mathbf{s}_i$ between 0 and 1 [S2].
    *   Initialize rotations $q_i$ as unit quaternions [S2].
    *   Initialize opacities $\alpha_i$ and spherical harmonics $h_i$ [S2][S6, unofficial].
    *   Set iteration counter $i = 0$ [S2].

2.  **Training Loop:**
    While $i \le \text{numIters}$:
    *   **Camera Sampling:** Select the next training camera $V$ and ground truth image $\hat{I}$ using a round-robin schedule with periodic shuffling [S2][S4, unofficial].
    *   **Resolution Scaling:** Compute the downscale factor for the current step. If $i$ exceeds `resolution_schedule` thresholds, increase image resolution [S4, unofficial].
    *   **Rendering:**
        *   Project 3D Gaussians to 2D screen space using the camera matrix [Parent].
        *   Sort Gaussians by depth [Parent].
        *   Rasterize to obtain rendered image $I$ via alpha blending [S2, Parent].
    *   **Loss Computation:**
        *   Compute loss $\mathcal{L}$ between $I$ and $\hat{I}$:
            $$ \mathcal{L} = (1 - \lambda) \mathcal{L}_1 + \lambda \mathcal{L}_{D-SSIM} $$
            where $\lambda = \text{ssim\_weight}$ [S6, unofficial].
    *   **Backpropagation:**
        *   Compute gradients $\nabla \mathcal{L}$ for all parameters ($\mathbf{p}, q, \mathbf{s}, h, \alpha$) using the chain rule [S6, unofficial].
        *   Update parameters using the Adam optimizer [S2][S6, unofficial].
    *   **Adaptive Density Control (Refinement):**
        *   If $i$ is a refinement iteration (e.g., $i \% \text{refineEvery} == 0$):
            *   For each Gaussian $i$:
                *   **Pruning:** If opacity $\alpha_i < \epsilon$ or the Gaussian is too large (screen size $> \text{split\_screen\_size}$ or covariance norm $||\mathbf{S}|| > \tau_S$), prune the Gaussian [S2][S4, unofficial].
                *   **Densification:** If positional gradient $\nabla_{\mathbf{p}} \mathcal{L} > \text{densify\_grad\_thresh}$:
                    *   **Over-reconstruction:** If $||\mathbf{S}|| > \tau_S$ (large covariance), split the Gaussian into two [S2].
                    *   **Under-reconstruction:** Else (small covariance), clone the Gaussian [S2].
    *   **Scheduler Update:** Advance learning rate schedulers [S4, unofficial].
    *   Increment $i$ [S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_gaussians(cameras, initial_gaussians, config):
    """
    config: dict containing numIters, sh_degree, shDegreeInterval, 
            densifyFrom, densifyUntil, refineEvery, densify_grad_thresh,
            densify_size_thresh, split_screen_size, stop_screen_size_at,
            ssim_weight, maxGaussians, downScaleFactor, num_downscales,
            resolution_schedule, device.
    """
    gaussians = initial_gaussians
    step = 1
    cam_iter = InfiniteRandomIterator(cameras)
    
    # Initialize SH degree
    current_sh_degree = 0
    
    while step <= config['numIters']:
        # 1. Camera Sampling
        cam = cam_iter.next()
        
        # 2. Resolution Scaling
        downscale_factor = get_downscale_factor(step, config)
        gt_image, gt_mask = acquire_ground_truth(cam, downscale_factor)
        
        # 3. SH Degree Increase
        if step % config['shDegreeInterval'] == 0 and current_sh_degree < config['sh_degree']:
            current_sh_degree += 1
            update_sh_degree(gaussians, current_sh_degree)
            
        # 4. Rendering
        rendered_image = rasterize(gaussians, cam, downscale_factor)
        
        # 5. Loss Computation
        loss = compute_loss(rendered_image, gt_image, gt_mask, config['ssim_weight'])
        
        # 6. Backpropagation
        gradients = backward_pass(loss, gaussians)
        update_parameters(gaussians, gradients, optimizer='adam')
        
        # 7. Adaptive Density Control
        if step % config['refineEvery'] == 0:
            refine_gaussians(gaussians, gradients, step, config)
            
        # 8. Post-training updates
        update_schedulers(step)
        
        step += 1

def refine_gaussians(gaussians, gradients, step, config):
    """
    Handles pruning, splitting, and cloning based on gradients and sizes.
    """
    to_prune = []
    to_split = []
    to_clone = []
    
    for i, g in enumerate(gaussians):
        # Pruning
        if g.opacity < 0.005 or g.screen_size > config['split_screen_size']:
            to_prune.append(i)
            continue
            
        # Densification Check
        grad_norm = norm(gradients.position[i])
        if grad_norm > config['densify_grad_thresh']:
            if step > config['densifyFrom'] and step < config['densifyUntil']:
                # Over-reconstruction: Large covariance
                if g.covariance_norm > config['densify_size_thresh']:
                    to_split.append(i)
                # Under-reconstruction: Small covariance
                else:
                    to_clone.append(i)
                    
    # Execute operations
    # Note: Indices must be handled carefully to avoid invalidation during modification
    apply_pruning(gaussians, to_prune)
    apply_splitting(gaussians, to_split)
    apply_cloning(gaussians, to_clone)
```

## Subtopics

### [Physical Parameter Optimization](4111-physical-parameter-optimization.md)
Replaces standard SH color and opacity with physical quantities (extinction, albedo) for dynamic relighting.

### [WildGaussians](6064-wildgaussians.md)
Models transient objects and illumination variations using a lightweight MLP and DINO features for "in-the-wild" scenes.

### [Non-Pinhole Camera Training](6088-non-pinhole-camera-training.md)
Adapts training to non-pinhole cameras using differentiable rasterizers that accommodate arbitrary projection models.

### [AbsGS](6127-absgs.md)
Uses absolute positional gradients to prevent gradient cancellation and sharpen fine details.

### [Densification Strategy](6234-densification-strategy.md)
Manages topology via dynamic adding, splitting, cloning, or removing based on optimization metrics.

### [Regularization and Geometry Constraints](6269-regularization-and-geometry-constraints.md)
Enforces geometric plausibility and prevents over-reconstruction via auxiliary constraints.

### [Base 3DGS Optimization](7549-base-3dgs-optimization.md)
Initializes geometric attributes with explicit normals for robust baseline optimization.

### [GSP Training Loss](9156-gsp-training-loss.md)
Extends color loss with normal consistency and anisotropy regularization for dynamic/physical contexts.

### [GPS-Gaussian](9197-gps-gaussian.md)
Regresses Gaussian parameters directly from sparse views to eliminate iterative optimization.

### [Optimization and Training Efficiency](9264-optimization-and-training-efficiency.md)
Manages memory, VRAM bottlenecks, and convergence via hardware-specific optimizations.

### [Dynamic and Temporal Modeling](9265-dynamic-and-temporal-modeling.md)
Parameterizes time to handle non-rigid geometry and appearance variations.

### [Compression and Representation Efficiency](9266-compression-and-representation-efficiency.md)

### [SLAM and Mapping](9268-slam-and-mapping.md)
Integrates SLAM for joint pose estimation and scene reconstruction.

### [Application-Specific Reconstructions](9269-application-specific-reconstructions.md)
Modifies training for domains like human avatars or robotics.

### [Geometry and Surface Priors](9270-geometry-and-surface-priors.md)
Couples Gaussians with SDFs or depth estimates for explicit surface structure.

### [Neural Network and Feed-Forward Approaches](9526-neural-network-and-feed-forward-approaches.md)

### [ClipGS](9546-clipgs.md)
Uses CLIP features for volumetric medical data visualization with dynamic clipping.

### [Representation and Kernel Modifications](10145-representation-and-kernel-modifications.md)
Alters primitives (e.g., negative Gaussians, voxel grids) for better expressivity.

### [TideGS Trajectory Ordering](10176-tidegs-trajectory-ordering.md)
Uses TSP-ordered view sequences to improve convergence stability.

### [Gradient Computation and Back-Propagation](10202-gradient-computation-and-back-propagation.md)
Defines the core differentiability mechanisms for the rendering equation.

### [VisibilitySplat](10927-visibilitysplat.md)
Augments rasterization with a visibility oracle for material-light decomposition.

### [MCMC-based 3D Gaussian Splatting](11914-mcmc-based-3d-gaussian-splatting.md)
Reformulates training as MCMC to handle discrete structural changes.

### [3D Gaussian Splatting as Markov Chain Monte Carlo (MCMC)](11915-3d-gaussian-splatting-as-markov-chain-monte-carlo-mcmc.md)
Reinterprets optimization as sampling from a target distribution.

### [Sliced Wasserstein-based Gaussian Cloning Update](11919-sliced-wasserstein-based-gaussian-cloning-update.md)
Minimizes 1D slice integral differences for cloned Gaussians.

### [Cross-Structure Collaboration Module](12071-cross-structure-collaboration-module.md)
Integrates global tri-plane and local grid features for large-scale scenes.

### HUGS
HUGS creates a coarse-to-fine deformation module using linear blend skinning and local learning-based refinement for animating virtual human avatars [S1]. It utilizes a tri-plane representation to factorize the canonical space, allowing reconstruction from monocular video (50-100 frames) within 30 minutes, achieving 20 FPS rendering [S1].

### Wolf et al. Stereo Method
This method trains a 3DGS to render stereo-calibrated novel views, applies stereo depth estimation on the rendered views, and fuses the estimated dense depth maps using a Truncated Signed Distance Function (TSDF) to form a triangular mesh [S1].

### Pseudo Ground Truth Acquisition for Gaussians
A labeling pipeline that generates Gaussian-specific pseudo ground truth labels from original point cloud labels, accounting for the densification and pruning processes of 3D-GS optimization [S8, unofficial].

### Cross-View Pruning Mechanism
A mechanism comprising a Cross-View Consistency Loss and adaptive pruning to reduce Gaussian redundancy [S9, unofficial]. It selects adjacent view pairs with SSIM > 0.6, computes a consistency loss to penalize residual discrepancies, and prunes Gaussians based on depth/distance consistency conditions [S9, unofficial].

### Structure-View Co-learning
A module combining structural and view-dependent gradients for robust joint optimization of Gaussian positions and appearance [S9, unofficial]. The loss is defined as $\mathcal{L} = \lambda (\lambda_V \mathcal{L}_V + \lambda_S \mathcal{L}_S)$, where $\mathcal{L}_V$ is view gradient loss and $\mathcal{L}_S$ is structure gradient loss [S9, unofficial].

### Single-view Mini-batch Strategy
A training data sampling strategy where data is organized per viewpoint, and a mini-batch consists of data from only a single viewpoint [S10, unofficial]. This involves collecting pixel-color pairs for each viewpoint $i$ into dataset $D_i$, organizing them into $D = \{D_i\}$, and sampling a mini-batch from a single $D_i$ [S10, unofficial].

## Operating characteristics
*   **Time Complexity:** Training typically requires 30,000 iterations [S3, S4, unofficial]. With GPU acceleration, training can be completed in under 90 seconds on modern hardware (e.g., M4 Max) [S4, unofficial]. CPU execution is approximately 100x slower than GPU [S3, unofficial].
*   **Memory Complexity:** Memory usage scales with the number of Gaussians, which can reach up to 5,000,000 by default [S3, unofficial]. Sinc-based variants consume less memory than standard Gaussians [S7].
*   **Performance:** Achieves comparable image quality to NeRFs but with significantly faster training and real-time rendering speeds [S2][S5, unofficial]. On the Replica dataset, SGS-SLAM achieves PSNR 34.15, SSIM 0.97, and LPIPS 0.096 [S1].
*   **Strengths:** Real-time rendering, high fidelity, fast training compared to NeRFs [S2][S5, unofficial].
*   **Limitations:** Sensitive to initialization; prone to "floaters" or over-reconstruction in empty space if not properly pruned [S2]. Standard 3DGS assumes pinhole cameras [S3, S4, unofficial].
*   **When to use:** Static or dynamic scene reconstruction from multi-view images where real-time rendering is required.
*   **When not to use:** Scenes with significant non-pinhole distortion without modification, or when explicit surface geometry is required without additional priors.

## Background
3D Gaussian Splatting was introduced as a method for real-time radiance field rendering [S2]. The original method initializes positions using sparse point clouds from SfM tools like COLMAP [S2][S6, unofficial]. It is implemented in various frameworks, including an IPOL demo application [S2] and C++ libraries like OpenSplat [S3, unofficial] and Msplat [S4, unofficial]. The training process is distinct from deep learning-based NeRFs as it optimizes explicit Gaussian primitives rather than implicit neural networks [S6, unofficial].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] Gaussian Splatting: An Introduction, https://doi.org/10.5201/ipol.2025.566, fetched 2026-10-26
[S3] OpenSplat – 3D gaussian splatting in C++ with CPU and GPU support, https://github.com/pierotofy/OpenSplat, fetched 2026-10-26 (unofficial)
[S4] Show HN: Msplat – 3D Gaussian Splatting training in ~90s on M4 Max, native Metal, https://github.com/rayanht/msplat, fetched 2026-10-26 (unofficial)
[S5] What is 3D Gaussian Splatting?, https://www.youtube.com/watch?v=Tnij_xHEnXc, fetched 2026-10-27 (unofficial)
[S6] 写真から超リアルな3D空間をどうやって復元するか? 「3D Gaussian Splatting」学習の徹底解説, https://qiita.com/scomup/items/92716342a3ef0b915e0c, fetched 2026-10-29 (unofficial)
[S7] DARB-Splatting: Generalizing Splatting with Decaying Anisotropic Radial Basis Functions, https://doi.org/10.1109/WACV61042.2026.00438, fetched 2026-10-30
[S8] Quantile Rendering: Efficiently Embedding High-dimensional Feature on 3D Gaussian Splatting, https://doi.org/10.48550/arxiv.2512.20927, fetched 2026-10-26 (unofficial)
[S9] SplatCo: Structure-View Collaborative Gaussian Splatting for Detail-Preserving Rendering of Large-Scale Unbounded Scenes, https://doi.org/10.48550/arXiv.2505.17951, fetched 2026-10-27 (unofficial)
[S10] Efficient multi-view training for 3D Gaussian Splatting, https://doi.org/10.48550/arxiv.2506.12727, fetched 2026-10-26 (unofficial)
