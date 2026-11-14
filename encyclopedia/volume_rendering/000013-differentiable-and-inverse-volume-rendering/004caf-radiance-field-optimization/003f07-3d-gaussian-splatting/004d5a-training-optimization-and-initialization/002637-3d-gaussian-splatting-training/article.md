# 3D Gaussian Splatting Training

Also called: 3DGS Learning, G Splats

## Overview
3D Gaussian Splatting Training is the iterative optimization process that adjusts the parameters of a set of 3D Gaussian primitives to minimize the photometric difference between rendered images and ground truth views [S2][S6, unofficial]. It solves the problem of reconstructing high-fidelity radiance fields from sparse or dense image collections by replacing implicit neural networks with explicit, differentiable geometric primitives [S5, unofficial]. Unlike NeRF-based methods which require iterative volume rendering per pixel, this method uses differentiable rasterization of 2D splats, offering significantly faster training and real-time rendering speeds while achieving comparable image quality [S2][S5, unofficial]. It builds on [Training Optimization and Initialization](13464-training-optimization-and-initialization.md) by defining the specific loss functions, densification heuristics, and update schedules used in the standard 3DGS pipeline.

## How it works
**Inputs and Outputs**
*   **Inputs:** A set of $N$ input images (or video frames) with known camera poses (extrinsics $R_{cw}, t_{cw}$ and intrinsics $K$) [S6, unofficial]. Initial 3D Gaussian parameters derived from Structure-from-Motion (SfM) point clouds [S2][S6, unofficial].
*   **Outputs:** Optimized 3D Gaussian parameters: positions $\mathbf{p}_i$, rotations $q_i$, scales $\mathbf{s}_i$, opacities $\alpha_i$, and spherical harmonic coefficients $h_i$ [S6, unofficial].

**Data Structures**
Each Gaussian primitive $i$ stores:
*   Position $\mathbf{p}_i \in \mathbb{R}^3$ (initialized from SfM) [S6, unofficial].
*   Rotation $q_i \in \mathbb{R}^4$ (unit quaternion) [S2].
*   Scale $\mathbf{s}_i \in \mathbb{R}^3$ (initialized between 0 and 1) [S2].
*   Opacity $\alpha_i \in [0, 1]$ (initialized as a constant) [S6, unofficial].
*   Color/SH coefficients $h_i$ (initialized from SfM colors) [S6, unofficial].

**Formulas**
The optimization minimizes a loss function $\mathcal{L}$ between the rendered image $\gamma$ and ground truth $\gamma_{gt}$ [S6, unofficial]:
$$ \mathcal{L} = (1 - \lambda) \mathcal{L}_1 + \lambda \mathcal{L}_{D-SSIM} $$
where $\lambda$ is the SSIM weight (default 0.2) [S6, unofficial].

The rendered pixel color $\gamma_j$ for pixel $j$ is computed via alpha blending of $N$ projected Gaussians sorted by depth:
$$ \gamma_j = \sum_{i \in N} \alpha'_{ij} c_i \tau_{ij}, \quad \tau_{ij} = \prod_{k=1}^{i-1} (1 - \alpha'_{kj}) $$
where $\alpha'_{ij}$ is the effective opacity, $c_i$ is the color, and $\tau_{ij}$ is the transmittance [S6, unofficial].

The effective opacity is $\alpha'_{ij} = g \alpha_i$, where $g = \exp(-0.5 \omega'_0 d_0^2 - 0.5 \omega'_2 d_1^2 - \omega'_1 d_0 d_1)$ [S6, unofficial]. Here, $\omega'$ are the upper triangular elements of the inverse 2D covariance matrix, and $d$ are the differences between the projected center $u_i$ and pixel coordinate $x_j$ [S6, unofficial].

**Parameters**
*   `numIters`: Total optimization steps. Default: 30,000 [S3, S4, unofficial].
*   `sh_degree`: Spherical harmonic degree. Default: 3 [S4, unofficial].
*   `sh_degree_interval`: Steps between increasing SH degree. Default: 1,000 [S4, unofficial].
*   `densifyFrom`: Step after which densification begins. Default: 500 [S3, unofficial].
*   `densifyUntil`: Step after which densification stops. Default: $\min(15,000, \text{numIters}/2)$ [S3, unofficial].
*   `refineEvery`: Steps between densification/pruning operations. Default: 500 [S3, unofficial].
*   `densify_grad_thresh`: Gradient threshold for densification. Default: 0.0002 [S4, unofficial].
*   `densify_size_thresh`: Size threshold for splitting. Default: 0.01 [S4, unofficial].
*   `split_screen_size`: Screen size threshold for splitting. Default: 0.05 [S4, unofficial].
*   `stop_screen_size_at`: Step at which screen-size splitting stops. Default: 4,000 [S4, unofficial].
*   `maxGaussians`: Maximum number of Gaussians. Default: 5,000,000 [S3, unofficial].
*   `downScaleFactor`: Initial image scaling. Default: 1.0 [S4, unofficial].
*   `numDownscales`: Number of resolution scales. Default: 0 (or 2 in some implementations) [S3, S4, unofficial].
*   `resolutionSchedule`: Steps between doubling resolution. Default: 3,000 [S3, unofficial].

**Steps**
1.  **Initialization:** Initialize Gaussian positions $\mathbf{p}_i$ from SfM point clouds in normalized bounds $[-1, 1]$. Initialize scales $\mathbf{s}_i$ between 0 and 1. Initialize rotations $q_i$ as unit quaternions and opacities $\alpha_i$ as constants [S2][S6, unofficial].
2.  **Camera Sampling:** Select a training camera $V$ and ground truth image $\hat{I}$ using a round-robin schedule with periodic shuffling [S4, unofficial]. Apply downscaling if the current step is within the resolution schedule [S4, unofficial].
3.  **Rendering:** Rasterize the Gaussians using camera $V$ to obtain rendered image $I$. This involves projecting 3D Gaussians to 2D screen space, sorting by depth, and alpha blending [Parent].
4.  **Loss Computation:** Compute loss $\mathcal{L}$ between $I$ and $\hat{I}$ using L1 and D-SSIM losses [S2][S6, unofficial].
5.  **Backpropagation:** Compute gradients for all parameters ($\mathbf{p}_w, q, \mathbf{s}, h, \alpha$) using the chain rule [S6, unofficial]. The Jacobians are:
    *   Rotation: $\frac{\partial \mathcal{L}}{\partial q_i} = \sum_j \frac{\partial \mathcal{L}}{\partial \gamma_j} \frac{\partial \gamma_j}{\partial \sigma'_i} \frac{\partial \sigma'_i}{\partial \sigma_i} \frac{\partial \sigma_i}{\partial q_i}$ [S6, unofficial].
    *   Scale: $\frac{\partial \mathcal{L}}{\partial s_i} = \sum_j \frac{\partial \mathcal{L}}{\partial \gamma_j} \frac{\partial \gamma_j}{\partial \sigma'_i} \frac{\partial \sigma'_i}{\partial \sigma_i} \frac{\partial \sigma_i}{\partial s_i}$ [S6, unofficial].
    *   SH: $\frac{\partial \mathcal{L}}{\partial h_i} = \sum_j \frac{\partial \mathcal{L}}{\partial \gamma_j} \frac{\partial \gamma_j}{\partial c_i} \frac{\partial c_i}{\partial h_i}$ [S6, unofficial].
    *   Opacity: $\frac{\partial \mathcal{L}}{\partial \alpha_i} = \sum_j \frac{\partial \mathcal{L}}{\partial \gamma_j} \frac{\partial \gamma_j}{\partial \alpha'_{ij}} \frac{\partial \alpha'_{ij}}{\partial \alpha_i}$ [S6, unofficial].
    *   Position: $\frac{\partial \mathcal{L}}{\partial p_{w,i}}$ involves three terms accounting for depth sorting, color, and covariance changes [S6, unofficial].
6.  **Optimizer Step:** Update parameters using the Adam optimizer [S2][S6, unofficial].
7.  **Adaptive Density Control:** If `IsRefinementIteration(i)` is true:
    *   **Pruning:** If $\alpha_i < \epsilon$ or the Gaussian is too large ($||\mathbf{S}|| > \tau_S$), prune the Gaussian [S2].
    *   **Densification:** If positional gradient $\nabla_p \mathcal{L} > \tau_p$:
        *   **Over-reconstruction:** If $||\mathbf{S}|| > \tau_S$, split the Gaussian using `SplitGaussian` [S2].
        *   **Under-reconstruction:** Else, clone the Gaussian using `CloneGaussian` [S2].
8.  **Progression:** Increment step counter. If `step` reaches `resolutionSchedule`, double image resolution. If `step` reaches `sh_degree_interval`, increase SH degree [S4, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Initialization
gaussians = initialize_from_sfm(points) # p, q, s, alpha, h
cameras = load_cameras(images)
step = 0
num_iters = 30000
densify_from = 500
densify_until = min(15000, num_iters // 2)
refine_every = 500

# Camera iterator with shuffle
cam_indices = list(range(len(cameras)))
shuffle(cam_indices)
cam_iter = Iterator(cam_indices)

while step < num_iters:
    # 1. Sample Camera
    cam = cameras[cam_iter.next()]
    
    # 2. Downscale Logic
    downscale = get_downscale_factor(step)
    gt_image = get_ground_truth(cam, downscale)
    
    # 3. Render
    rendered_image = rasterize(gaussians, cam)
    
    # 4. Compute Loss
    loss = compute_loss(rendered_image, gt_image, ssim_weight=0.2)
    
    # 5. Backpropagate
    loss.backward()
    
    # 6. Optimizer Step
    optimizer.step()
    optimizer.zero_grad()
    
    # 7. Adaptive Density Control
    if step >= densify_from and step <= densify_until and step % refine_every == 0:
        for g in gaussians:
            # Pruning
            if g.opacity < epsilon or is_too_large(g.scale):
                gaussians.remove(g)
                continue
            
            # Densification based on gradient
            if get_gradient_norm(g.position) > grad_thresh:
                if is_too_large(g.scale):
                    # Over-reconstruction: Split
                    split_gaussian(g)
                else:
                    # Under-reconstruction: Clone
                    clone_gaussian(g)
    
    # 8. Schedule Updates
    if step % resolution_schedule == 0:
        increase_resolution()
    if step % sh_degree_interval == 0:
        increase_sh_degree()
        
    step += 1
```

## Subtopics

### [Physical Parameter Optimization](4111-physical-parameter-optimization.md)
Replaces standard SH color and empirical opacity with physical quantities (extinction, albedo, scattering) to enable dynamic relighting by solving the radiative transfer equation.

### [WildGaussians](6064-wildgaussians.md)
Models image-dependent appearance variations via a lightweight MLP and separates static scene geometry from transient occluders to handle "in-the-wild" photo collections.

### [Non-Pinhole Camera Training](6088-non-pinhole-camera-training.md)
Adapts training to non-pinhole cameras (e.g., with lens distortion) by leveraging a differentiable rasterizer that accommodates arbitrary camera projection models.

### [AbsGS](6127-absgs.md)
Uses absolute sums of positional gradients across multiple views to prevent gradient cancellation, improving the detection of "fence-sitting" Gaussians and sharpening edges.

### [Densification Strategy](6234-densification-strategy.md)
Manages the dynamic lifecycle of Gaussians (splitting, cloning, pruning) based on optimization metrics to balance reconstruction quality and computational cost [Parent, 13466-densification-and-splitting.md].

### [Regularization and Geometry Constraints](6269-regularization-and-geometry-constraints.md)
Introduces auxiliary constraints (e.g., normal consistency, scale regularization) to enforce geometric plausibility and prevent over-reconstruction or floaters.

### [Base 3DGS Optimization](7549-base-3dgs-optimization.md)
Incorporates an explicit normal attribute and optimizes it via normal consistency loss to accelerate convergence of editable Gaussian optimization.

### [GSP Training Loss](9156-gsp-training-loss.md)
Extends the standard loss with normal consistency and anisotropy regularization to prevent artifacts from excessive kernel elongation in dynamic simulations.

### [GPS-Gaussian](9197-gps-gaussian.md)
Eliminates iterative optimization by regressing Gaussian parameters directly from sparse source views using a depth estimation module.

### [Optimization and Training Efficiency](9264-optimization-and-training-efficiency.md)
Focuses on computational strategies, memory management, and fused kernels to scale training to large scenes and high iteration counts.

### [Dynamic and Temporal Modeling](9265-dynamic-and-temporal-modeling.md)
Extends static 3DGS to handle non-rigid geometry and appearance variations by parameterizing Gaussian attributes as time-dependent functions.

### [Compression and Representation Efficiency](9266-compression-and-representation-efficiency.md)
Not given in the sources.

### [SLAM and Mapping](9268-slam-and-mapping.md)
Integrates the differentiable rendering pipeline into the pose optimization loop for simultaneous localization and mapping, often using semantic information.

### [Application-Specific Reconstructions](9269-application-specific-reconstructions.md)
Modifies the training loop for domain-specific challenges like dynamic human avatars using custom priors and deformation fields.

### [Geometry and Surface Priors](9270-geometry-and-surface-priors.md)
Enforces geometric consistency by coupling Gaussian representation with continuous signals like SDFs or monocular depth estimates.

### [Neural Network and Feed-Forward Approaches](9526-neural-network-and-feed-forward-approaches.md)
Not given in the sources.

### [ClipGS](9546-clipgs.md)
Uses CLIP features for volumetric medical data visualization with dynamic clipping planes, treating clipping as a geometric visibility constraint.

### [Representation and Kernel Modifications](10145-representation-and-kernel-modifications.md)
Alters the mathematical definition of the splat (e.g., negative coefficients, voxel scaffolding) rather than just the optimization schedule.

### [TideGS Trajectory Ordering](10176-tidegs-trajectory-ordering.md)
Replaces randomized view shuffling with a trajectory-aware ordered sequence (TSP) to improve optimization stability for large-scale scenes.

### [Gradient Computation and Back-Propagation](10202-gradient-computation-and-back-propagation.md)
Defines the core differentiability mechanisms for propagating loss gradients to Gaussian parameters and camera poses.

### [VisibilitySplat](10927-visibilitysplat.md)
Augments rasterization with a sparse visibility oracle (BVH ray tracer) to estimate transmittance-weighted occlusion for material-light decomposition.

### [MCMC-based 3D Gaussian Splatting](11914-mcmc-based-3d-gaussian-splatting.md)
Reforms training as an MCMC process, replacing heuristic densification with stochastic exploration via noise injection and relocation.

### [3D Gaussian Splatting as Markov Chain Monte Carlo (MCMC)](11915-3d-gaussian-splatting-as-markov-chain-monte-carlo-mcmc.md)
Reinterprets training as sampling from a target distribution, treating structural changes as valid state transitions to avoid local minima.

### [Sliced Wasserstein-based Gaussian Cloning Update](11919-sliced-wasserstein-based-gaussian-cloning-update.md)
Determines scale updates for cloned Gaussians by minimizing the difference in integrated color along 1D slices to ensure rasterization integral conservation.

### [Cross-Structure Collaboration Module](12071-cross-structure-collaboration-module.md)
Integrates global tri-plane features with local context-aware grid features to preserve global coherence and capture local detail in large scenes.

### HUGS
Creates a coarse-to-fine deformation module using linear blend skinning and local learning-based refinement for animating virtual human avatars based on 3DGS. It utilizes tri-plane representation to factorize the canonical space, reconstructing from monocular video (50-100 frames) within 30 minutes at 20 FPS [S1].

### Wolf et al. Stereo Method
Trains a 3DGS to render stereo-calibrated novel views, applies stereo depth estimation on the rendered views, and fuses the estimated dense depth maps using a Truncated Signed Distance Function (TSDF) to form a triangular mesh [S1].

### Pseudo Ground Truth Acquisition for Gaussians
A labeling pipeline that generates Gaussian-specific pseudo ground truth labels from original point cloud labels, accounting for the densification and pruning processes of 3D-GS optimization [S8, unofficial].

### Cross-View Pruning Mechanism
A mechanism comprising a Cross-View Consistency Loss and adaptive pruning to reduce Gaussian redundancy. It selects adjacent view pairs with SSIM > 0.6, computes a consistency loss to penalize residual discrepancies, and prunes Gaussians based on depth/distance consistency conditions [S9, unofficial].

### Structure-View Co-learning
A module combining structural and view-dependent gradients for robust joint optimization. The loss is $\mathcal{L} = \lambda (\lambda_V \mathcal{L}_V + \lambda_S \mathcal{L}_S)$, where $\mathcal{L}_V$ is view gradient loss and $\mathcal{L}_S$ is structure gradient loss [S9, unofficial].

### Single-view Mini-batch Strategy
A training data sampling strategy where data is organized per viewpoint, and a mini-batch consists of data from only a single viewpoint to improve efficiency [S10, unofficial].

## Operating characteristics
*   **Time Complexity:** Training typically takes 15-60 minutes for standard datasets (e.g., Mip-NeRF 360) on modern GPUs, significantly faster than NeRFs which can take hours [S2][S5, unofficial].
*   **Memory Complexity:** Memory usage scales with the number of Gaussians. Default limit is 5,000,000 Gaussians [S3, unofficial]. Sinc splatting variants consume less memory than standard Gaussians [S7].
*   **Performance:** Achieves real-time rendering speeds (100+ FPS) at 1080p on consumer hardware [S5, unofficial]. PSNR and SSIM are comparable to NeRFs [S2].
*   **Scaling:** Scales to large scenes via hierarchical representations or optimized training loops (e.g., Msplat) [S4, S10, unofficial].
*   **Strengths:** Fast training, real-time rendering, high visual fidelity [S5, unofficial].
*   **Limitations:** Can produce "floaters" or over-reconstruction in empty space; sensitive to initialization; requires careful densification tuning [S2][S3, unofficial].
*   **When to use:** When real-time rendering and fast training are priorities over absolute geometric accuracy or implicit surface extraction [S2].
*   **When not to use:** When explicit mesh extraction is required without post-processing, or when dealing with highly dynamic scenes without temporal extensions [S1].

## Background
3D Gaussian Splatting was introduced by Kerbl et al. in 2023 [S2]. It builds on the concept of radiance fields but replaces neural networks with explicit 3D Gaussians. The method initializes positions using sparse point clouds from SfM tools like COLMAP [S2][S6, unofficial]. It has since been extended to various domains including SLAM, dynamic scenes, and human avatars [S1].

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
