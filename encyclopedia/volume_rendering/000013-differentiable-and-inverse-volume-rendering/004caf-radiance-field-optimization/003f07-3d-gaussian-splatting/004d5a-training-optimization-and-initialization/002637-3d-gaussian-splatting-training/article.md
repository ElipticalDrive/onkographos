# 3D Gaussian Splatting Training

Also called: 3DGS Learning, G Splats

## Overview
3D Gaussian Splatting Training is the iterative optimization process that adjusts the parameters of a set of 3D Gaussian primitives to minimize the difference between rendered images and ground truth views [S2][S5, S6, unofficial]. It solves the problem of reconstructing high-fidelity 3D scenes from 2D images by converting sparse point clouds into a differentiable radiance field that can be rendered in real-time [S5, unofficial]. What sets it apart from NeRF-based methods is its explicit representation using analytic Gaussian kernels and adaptive density control, which yields significantly faster training and inference speeds while maintaining comparable image quality [S2][S5, unofficial]. It builds on [Base 3DGS Optimization](7549-base-3d-gaussian-splatting-optimization.md) by managing the dynamic population of primitives through densification and pruning [S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ input images (or video frames) with known camera poses (extrinsics $R_{cw}, t_{cw}$ and intrinsics $K$) [S6, unofficial]. Initial 3D Gaussian parameters derived from Structure from Motion (SfM) [S2][S6, unofficial].
*   **Outputs:** Optimized 3D Gaussian parameters: positions $\mathbf{p}_i$, rotations $q_i$, scales $\mathbf{s}_i$, opacities $\alpha_i$, and spherical harmonic coefficients $h_i$ [S6, unofficial].

**Data Structures:**
Each Gaussian primitive $i$ is defined by:
*   Position $\mathbf{p}_i \in \mathbb{R}^3$ (initialized from SfM points) [S6, unofficial].
*   Rotation $q_i \in \mathbb{R}^4$ (unit quaternion, initialized as constant) [S6, unofficial].
*   Scale $\mathbf{s}_i \in \mathbb{R}^3$ (initialized between 0 and 1) [S2][S6, unofficial].
*   Opacity $\alpha_i \in [0, 1]$ (initialized as constant) [S6, unofficial].
*   Appearance $h_i$ (Spherical Harmonic coefficients, initialized from SfM) [S6, unofficial].

The covariance matrix $\Sigma_i$ is computed from rotation and scale:
$$ \Sigma_i = R(q_i) S(\mathbf{s}_i) S(\mathbf{s}_i)^T R(q_i)^T $$
where $R(q_i)$ is the rotation matrix derived from the quaternion [S2]:
$$ R(q) = 2 \begin{bmatrix} \frac{1}{2} - (q_j^2 + q_k^2) & q_i q_j - q_r q_k & q_i q_k + q_r q_j \\ q_i q_j + q_r q_k & \frac{1}{2} - (q_i^2 + q_k^2) & q_j q_k - q_r q_i \\ q_i q_k - q_r q_j & q_j q_k + q_r q_i & \frac{1}{2} - (q_i^2 + q_j^2) \end{bmatrix} $$
[S2]

**Parameters:**
*   `numIters`: Total optimization steps. Default: 30,000 [S3, S4, unofficial].
*   `densifyFrom`: Step after which densification begins. Default: 500 [S3, unofficial].
*   `densifyUntil`: Step after which densification stops. Default: $\min(15000, \text{numIters}/2)$ [S3, unofficial].
*   `refineEvery`: Frequency of densification/pruning operations. Default: 500 [S3, unofficial].
*   `densify_grad_thresh`: Gradient threshold for densification. Default: 0.0002 [S4, unofficial].
*   `densify_size_thresh`: Size threshold for densification. Default: 0.01 [S4, unofficial].
*   `split_screen_size`: Screen size threshold for splitting. Default: 0.05 [S4, unofficial].
*   `stop_screen_size_at`: Step at which screen-size splitting stops. Default: 4000 [S4, unofficial].
*   `sh_degree`: Spherical harmonic degree. Default: 3 [S4, unofficial].
*   `sh_degree_interval`: Steps between increasing SH degree. Default: 1000 [S4, unofficial].
*   `resolutionSchedule`: Steps between doubling image resolution. Default: 3000 [S3, unofficial].
*   `ssim_weight`: Weight for SSIM loss. Default: 0.2 [S3, S4, unofficial].
*   `maxGaussians`: Maximum number of Gaussians. Default: 5,000,000 [S3, unofficial].

**Algorithm:**
The training loop minimizes the loss $\mathcal{L}$ between the rendered image $\gamma$ and ground truth $\gamma_{gt}$ [S6, unofficial]:
$$ \mathcal{L} = (1 - \lambda) \mathcal{L}_1 + \lambda \mathcal{L}_{D-SSIM} $$
where $\lambda$ is the SSIM weight [S6, unofficial].

1.  **Initialization:** Initialize positions $\mathbf{p}_w$ and colors $h$ from SfM point clouds (e.g., COLMAP). Initialize rotation $q$, scale $s$, and opacity $\alpha$ as constants [S2][S6, unofficial]. Set iteration counter $i = 0$ [S2].
2.  **Camera Sampling:** Select a training camera $V$ and retrieve the ground truth image $\hat{I}$ at the current resolution scale [S2]. Cameras are sampled via a shuffled round-robin iterator [S4, unofficial].
3.  **Rendering:** Project the 3D Gaussians to 2D screen space using the camera matrix. Sort by depth. Alpha blend to produce rendered image $I$ [S2][S6, unofficial].
    *   Pixel color $\gamma_j$:
        $$ \gamma_j = \sum_{i \in N} \alpha'_{ij} c_i \tau_{ij}, \quad \tau_{ij} = \prod_{k=1}^{i-1} (1 - \alpha'_{kj}) $$
    *   Effective opacity $\alpha'_{ij} = g \alpha_i$, where $g$ is the 2D Gaussian kernel value [S6, unofficial].
4.  **Loss Computation:** Compute $\mathcal{L}$ between $I$ and $\hat{I}$ [S2].
5.  **Backpropagation:** Compute gradients $\frac{\partial \mathcal{L}}{\partial \theta}$ for all parameters $\theta \in \{q, s, h, \alpha, \mathbf{p}_w\}$ using the chain rule [S6, unofficial].
    *   Jacobian for rotation $q_i$:
        $$ \frac{\partial \mathcal{L}}{\partial q_i} = \sum_j \frac{\partial \mathcal{L}}{\partial \gamma_j} \frac{\partial \gamma_j}{\partial \sigma'_i} \frac{\partial \sigma'_i}{\partial \sigma_i} \frac{\partial \sigma_i}{\partial q_i} $$
    *   Jacobian for scale $s_i$:
        $$ \frac{\partial \mathcal{L}}{\partial s_i} = \sum_j \frac{\partial \mathcal{L}}{\partial \gamma_j} \frac{\partial \gamma_j}{\partial \sigma'_i} \frac{\partial \sigma'_i}{\partial \sigma_i} \frac{\partial \sigma_i}{\partial s_i} $$
    *   Jacobian for SH $h_i$:
        $$ \frac{\partial \mathcal{L}}{\partial h_i} = \sum_j \frac{\partial \mathcal{L}}{\partial \gamma_j} \frac{\partial \gamma_j}{\partial c_i} \frac{\partial c_i}{\partial h_i} $$
    *   Jacobian for opacity $\alpha_i$:
        $$ \frac{\partial \mathcal{L}}{\partial \alpha_i} = \sum_j \frac{\partial \mathcal{L}}{\partial \gamma_j} \frac{\partial \gamma_j}{\partial \alpha'_{ij}} \frac{\partial \alpha'_{ij}}{\partial \alpha_i} $$
    *   Jacobian for position $\mathbf{p}_{w,i}$:
        $$ \frac{\partial \mathcal{L}}{\partial \mathbf{p}_{w,i}} = \sum_j \frac{\partial \mathcal{L}}{\partial \gamma_j} \frac{\partial \gamma_j}{\partial \alpha'_{ij}} \frac{\partial \alpha'_{ij}}{\partial u_i} \frac{\partial u_i}{\partial \mathbf{p}_{c,i}} \frac{\partial \mathbf{p}_{c,i}}{\partial \mathbf{p}_{w,i}} + \sum_j \frac{\partial \mathcal{L}}{\partial \gamma_j} \frac{\partial \gamma_j}{\partial c_i} \frac{\partial c_i}{\partial \mathbf{p}_{w,i}} + \sum_j \frac{\partial \mathcal{L}}{\partial \gamma_j} \frac{\partial \gamma_j}{\partial \alpha'_{ij}} \frac{\partial \alpha'_{ij}}{\partial \sigma'_i} \frac{\partial \sigma'_i}{\partial \mathbf{p}_{c,i}} \frac{\partial \mathbf{p}_{c,i}}{\partial \mathbf{p}_{w,i}} $$
    [S6, unofficial]
6.  **Optimization Step:** Update parameters using the Adam optimizer [S2][S6, unofficial].
7.  **Adaptive Density Control:** If $i$ is a refinement iteration (every `refineEvery` steps):
    *   **Pruning:** If $\alpha_i < \epsilon$ or the Gaussian is too large, remove it [S2].
    *   **Densification:** If positional gradient $\nabla_{\mathbf{p}} \mathcal{L} > \tau_p$:
        *   **Over-reconstruction:** If scale norm $\|\mathbf{S}\| > \tau_S$, split the Gaussian into two [S2].
        *   **Under-reconstruction:** Else, clone the Gaussian [S2].
    *   Increment $i$ [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_3dgs(training_data, num_iters=30000, refine_every=500, densify_from=500, densify_until=15000):
    # 1. Initialization
    gaussians = initialize_from_sfm(training_data.sfm_points)
    # q, s, alpha initialized as constants; p, h from SfM
    optimizer = Adam(gaussians.parameters())
    step = 0
    
    # Camera iterator
    cam_indices = shuffle(range(len(training_data.cameras)))
    cam_iter = Iterator(cam_indices)

    while step < num_iters:
        # 2. Camera Sampling
        cam_idx = cam_iter.next()
        if cam_iter.done():
            cam_indices = shuffle(range(len(training_data.cameras)))
            cam_iter = Iterator(cam_indices)
            
        camera = training_data.cameras[cam_idx]
        gt_image = training_data.get_image(cam_idx, step) # Handles downscaling
        
        # 3. Rendering
        rendered_image = rasterize(gaussians, camera)
        
        # 4. Loss Computation
        loss = compute_loss(rendered_image, gt_image)
        
        # 5. Backpropagation
        loss.backward()
        
        # 6. Optimization Step
        optimizer.step()
        optimizer.zero_grad()
        
        # 7. Adaptive Density Control
        if step >= densify_from and step < densify_until and step % refine_every == 0:
            densify_and_prune(gaussians, step)
            
        step += 1

def densify_and_prune(gaussians, step):
    for g in gaussians:
        # Pruning
        if g.opacity < epsilon or is_too_large(g.scale):
            gaussians.remove(g)
            continue
            
        # Densification
        if g.position_gradient > densify_grad_thresh:
            if g.scale_norm > densify_size_thresh:
                # Over-reconstruction: Split
                gaussians.split(g)
            else:
                # Under-reconstruction: Clone
                gaussians.clone(g)
```
[S2][S4, S6, unofficial]

**Edge Cases and Pitfalls:**
*   **No Cameras See Points:** If all cameras are discarded because they see no sparse points, training fails [S3, unofficial].
*   **Arbitrary Pose Rendering:** Requires exactly 16 elements for the `cam_to_world` matrix [S4, unofficial].
*   **Evaluation Mode:** Dataset must be loaded with `eval_mode=True` for evaluation, otherwise it throws a runtime error [S4, unofficial].
*   **CPU vs GPU:** CPU execution is ~100x slower than GPU [S3, unofficial].

## Subtopics

### [Physical Parameter Optimization](4111-physical-parameter-optimization.md)
Replaces SH color and empirical opacity with physical quantities (extinction, albedo) for dynamic relighting.

### [WildGaussians](6064-wildgaussians.md)
Integrates DINO features and appearance modeling for "in-the-wild" unconstrained photo collections.

### [Non-Pinhole Camera Training](6088-non-pinhole-camera-training.md)
Adapts training to non-pinhole cameras using differentiable rasterizers for arbitrary projection models.

### [AbsGS](6127-absgs.md)
Uses absolute positional gradients to prevent gradient cancellation and sharpen edges.

### [Densification Strategy](6234-densification-strategy.md)
Manages primitive topology via dynamic adding, splitting, cloning, and removal.

### [Regularization and Geometry Constraints](6269-regularization-and-geometry-constraints.md)
Enforces geometric plausibility to prevent floaters and stabilize convergence.

### [Base 3DGS Optimization](7549-base-3dgs-optimization.md)
Initializes geometric attributes with explicit normals for robust baseline optimization.

### [GSP Training Loss](9156-gsp-training-loss.md)
Extends color loss with normal consistency and anisotropy regularization for dynamic/physical rendering.

### [GPS-Gaussian](9197-gps-gaussian.md)
Regresses Gaussian parameters directly from sparse views, eliminating iterative optimization.

### [Optimization and Training Efficiency](9264-optimization-and-training-efficiency.md)
Manages memory, VRAM bottlenecks, and convergence via hardware-specific optimizations.

### [Dynamic and Temporal Modeling](9265-dynamic-and-temporal-modeling.md)
Parameterizes time to handle non-rigid geometry and appearance variations.

### [Compression and Representation Efficiency](9266-compression-and-representation-efficiency.md)
Reduces computational resources via vector quantization and other compression methods.

### [SLAM and Mapping](9268-slam-and-mapping.md)
Integrates SLAM for joint pose estimation and scene reconstruction.

### [Application-Specific Reconstructions](9269-application-specific-reconstructions.md)
Modifies training for domains like human avatars or robotics.

### [Geometry and Surface Priors](9270-geometry-and-surface-priors.md)
Couples Gaussians with SDFs or depth estimates for explicit surface structure.

### [Neural Network and Feed-Forward Approaches](9526-neural-network-and-feed-forward-approaches.md)
Uses neural networks to predict Gaussian parameters directly.

### [ClipGS](9546-clipgs.md)
Handles interactive clipping planes for volumetric medical data.

### [Representation and Kernel Modifications](10145-representation-and-kernel-modifications.md)
Modifies primitives or kernels (e.g., negative colors, voxel scaffolding).

### [TideGS Trajectory Ordering](10176-tidegs-trajectory-ordering.md)
Uses TSP-ordered view sequences for stable large-scale training.

### [Gradient Computation and Back-Propagation](10202-gradient-computation-and-back-propagation.md)
Defines the core differentiable rendering gradient path.

### [VisibilitySplat](10927-visibilitysplat.md)
Augments rasterization with a visibility oracle for material-light decomposition.

### [MCMC-based 3D Gaussian Splatting](11914-mcmc-based-3d-gaussian-splatting.md)
Reformulates training as MCMC to avoid local minima.

### [3D Gaussian Splatting as Markov Chain Monte Carlo (MCMC)](11915-3d-gaussian-splatting-as-markov-chain-monte-carlo-mcmc.md)
Interprets training as sampling from a target distribution.

### [Sliced Wasserstein-based Gaussian Cloning Update](11919-sliced-wasserstein-based-gaussian-cloning-update.md)
Minimizes 1D slice integral differences for cloned Gaussians.

### [Cross-Structure Collaboration Module](12071-cross-structure-collaboration-module.md)
Integrates global tri-plane and local grid features for large-scale scenes.

### HUGS
HUGS creates a coarse-to-fine deformation module using linear blend skinning and local learning-based refinement for animating virtual human avatars based on 3DGS [S1]. It utilizes the tri-plane representation to factorize the canonical space, reconstructing the person and scene from monocular video (50-100 frames) within 30 minutes, achieving state-of-the-art performance at 20 FPS [S1].

### Wolf et al. Stereo Method
This method trains a 3DGS to render stereo-calibrated novel views, applies stereo depth estimation on the rendered views, and fuses the estimated dense depth maps using a Truncated Signed Distance Function (TSDF) to form a triangular mesh [S1].

### Pseudo Ground Truth Acquisition for Gaussians
A labeling pipeline that generates Gaussian-specific pseudo ground truth labels from original point cloud labels, accounting for the densification and pruning processes of 3D-GS optimization [S8, unofficial].

### Cross-View Pruning Mechanism
A mechanism comprising a Cross-View Consistency Loss and adaptive pruning to reduce Gaussian redundancy and ensure geometric validity [S9, unofficial]. It selects adjacent view pairs with SSIM > 0.6, computes the Cross-View Consistent Loss ($L_{CVC}$) to penalize residual discrepancies, and prunes based on depth/distance consistency conditions ($d^i_p < 0.01 \land d^j_p < 0.5 \lor d^i_p > 3\sigma$) [S9, unofficial].

### Structure-View Co-learning
A module combining structural and view-dependent gradients for robust joint optimization of Gaussian positions and appearance [S9, unofficial]. It minimizes $\mathcal{L} = \lambda (\lambda_V \mathcal{L}_V + \lambda_S \mathcal{L}_S)$, where $\mathcal{L}_V$ is view gradient loss and $\mathcal{L}_S$ is structure gradient loss [S9, unofficial].

### Single-view Mini-batch Strategy
A training data sampling strategy where data is organized per viewpoint, and a mini-batch consists of data from only a single viewpoint [S10, unofficial]. It collects pixel-color pairs $\{p_{ij}, C_{ij}\}$ for each viewpoint $i$, organizes them into $D = \{D_i\}$, and samples a mini-batch $B_{3DGS}$ from count=1 viewpoint [S10, unofficial].

## Operating characteristics
*   **Time Complexity:** Training typically takes 15-30 minutes for standard scenes (e.g., Mip-NeRF 360) on modern GPUs, significantly faster than NeRFs which can take hours [S2][S5, unofficial].
*   **Memory Complexity:** Memory usage scales with the number of Gaussians. Default max is 5,000,000 Gaussians [S3, unofficial]. Sinc splatting variants consume less memory than standard Gaussians [S7].
*   **Performance:** Achieves real-time rendering speeds (100+ FPS) at 1080p resolution [S5, unofficial]. On the Replica dataset, SGS-SLAM achieves PSNR 34.15, SSIM 0.97, and LPIPS 0.096 [S1].
*   **Scaling:** Performance degrades on CPU (100x slower than GPU) [S3, unofficial]. Resolution scaling is handled via downscale factors, increasing resolution every 3000 steps [S3, S4, unofficial].
*   **Strengths:** Fast training, real-time rendering, high fidelity [S2][S5, unofficial].
*   **Limitations:** Can produce "floaters" or over-reconstruction artifacts if densification is not properly controlled [S2]. Requires careful tuning of densification thresholds [S4, unofficial].

## Background
3D Gaussian Splatting was introduced by Kerbl et al. [9] (cited in [S1, S2]). It initializes positions using sparse point clouds from SfM tools like COLMAP [S2]. The method was developed to address the slow training and rendering speeds of NeRFs while maintaining high visual quality [S5, unofficial]. It has since been extended to various domains including SLAM [S1], dynamic scenes [S1], and human avatars [S1].

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
