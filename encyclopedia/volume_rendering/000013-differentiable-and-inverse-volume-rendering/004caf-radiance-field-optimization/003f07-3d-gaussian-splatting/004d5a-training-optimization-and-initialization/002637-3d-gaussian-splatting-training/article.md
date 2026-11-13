# 3D Gaussian Splatting Training

Also called: 3DGS Learning, G Splats

## Overview
3D Gaussian Splatting Training is the optimization process that adjusts the parameters of a set of 3D Gaussian primitives to minimize the difference between rendered images and ground truth views [S2][S5, S6, unofficial]. It solves the problem of reconstructing high-fidelity, real-time renderable radiance fields from sparse or dense input images by iteratively refining position, rotation, scale, opacity, and appearance [S5, S6, unofficial]. Unlike static initialization methods, it employs adaptive density control to dynamically split, clone, or prune Gaussians based on gradient magnitude and opacity, addressing under- and over-reconstruction [S2][S6, unofficial]. It sets itself apart from NeRF-based methods by offering significantly faster training and inference speeds while achieving comparable image quality [S2][S5, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ input images (or video frames) with known camera poses (rotation $R_{cw}$, translation $t_{cw}$, intrinsic matrix $K$) [S6, unofficial]. Initial 3D Gaussian parameters derived from Structure from Motion (SfM) [S2][S6, unofficial].
*   **Outputs:** Optimized 3D Gaussian parameters (means, covariances, spherical harmonic coefficients, opacities) stored in a format such as PLY [S4, S6, unofficial].

**Data Structures:**
Each Gaussian primitive $i$ is defined by:
*   Position $\mathbf{p}_i \in \mathbb{R}^3$ (initialized from SfM points) [S6, unofficial].
*   Rotation $q_i \in \mathbb{R}^4$ (unit quaternion, initialized as constant) [S2][S6, unofficial].
*   Scale $\mathbf{s}_i \in \mathbb{R}^3$ (initialized between 0 and 1) [S2][S6, unofficial].
*   Opacity $\alpha_i \in [0, 1]$ (initialized as constant) [S2][S6, unofficial].
*   Appearance parameters: Spherical Harmonic coefficients $h_i$ (initialized from SfM) [S6, unofficial].

The rotation matrix $R(q)$ is computed from the quaternion $q = (q_r, q_i, q_j, q_k)$ as [S2][S6, unofficial]:
$$
R(q) = 2 \begin{bmatrix} \frac{1}{2} - (q_j^2 + q_k^2) & q_i q_j - q_r q_k & q_i q_k + q_r q_j \\ q_i q_j + q_r q_k & \frac{1}{2} - (q_i^2 + q_k^2) & q_j q_k - q_r q_i \\ q_i q_k - q_r q_j & q_j q_k + q_r q_i & \frac{1}{2} - (q_i^2 + q_j^2) \end{bmatrix}
$$
The covariance matrix $\Sigma$ is derived from $R$ and $\mathbf{s}$ [S2].

**Numbered Steps:**
1.  **Initialization:** Initialize positions $\mathbf{p}_w$ and spherical harmonics $h$ from SfM photogrammetry (e.g., COLMAP) [S2][S6, unofficial]. Initialize rotation $q$, scale $s$, and opacity $\alpha$ as constants [S6, unofficial]. Set iteration count $i = 0$ [S2].
2.  **Camera Sampling:** Select a training camera $V$ and retrieve the corresponding ground truth image $\hat{I}$ at the current resolution [S2]. Cameras are sampled via a shuffled round-robin iterator [S4, unofficial].
3.  **Rendering:** Rasterize the Gaussians $(\mathbf{M}, \mathbf{S}, \mathbf{C}, \mathbf{A})$ using camera $V$ to obtain the rendered image $I$ [S2]. This involves projecting 3D Gaussians to 2D screen space, sorting by depth, and alpha blending [Parent]. The pixel color $\gamma_j$ is computed as [S6, unofficial]:
    $$
    \gamma_j = \sum_{i \in N} \alpha'_{ij} c_i \tau_{ij}, \quad \tau_{ij} = \prod_{k=1}^{i-1} (1 - \alpha'_{kj})
    $$
    where $\alpha'_{ij} = g \alpha_i$ is the effective opacity, $g$ is a depth-dependent attenuation factor, and $c_i$ is the color [S6, unofficial].
4.  **Loss Computation:** Compute the loss $\mathcal{L}$ between the rendered image $I$ and ground truth $\hat{I}$ [S2]. The loss is a combination of L1 loss ($\mathcal{L}_1$) and D-SSIM loss ($\mathcal{L}_{D-SSIM}$) weighted by $\lambda$ [S6, unofficial]:
    $$
    \mathcal{L} = (1 - \lambda) \mathcal{L}_1 + \lambda \mathcal{L}_{D-SSIM}
    $$
    Typical default for $\lambda$ (ssim_weight) is 0.2 [S4, unofficial].
5.  **Backpropagation:** Compute gradients $\nabla \mathcal{L}$ for all parameters ($q, s, h, \alpha, \mathbf{p}_w$) using the chain rule [S6, unofficial]. The Jacobians for each parameter are [S6, unofficial]:
    *   Rotation: $\frac{\partial \mathcal{L}}{\partial q_i} = \sum_j \frac{\partial \mathcal{L}}{\partial \gamma_j} \frac{\partial \gamma_j}{\partial \sigma'_i} \frac{\partial \sigma'_i}{\partial \sigma_i} \frac{\partial \sigma_i}{\partial q_i}$
    *   Scale: $\frac{\partial \mathcal{L}}{\partial s_i} = \sum_j \frac{\partial \mathcal{L}}{\partial \gamma_j} \frac{\partial \gamma_j}{\partial \sigma'_i} \frac{\partial \sigma'_i}{\partial \sigma_i} \frac{\partial \sigma_i}{\partial s_i}$
    *   SH: $\frac{\partial \mathcal{L}}{\partial h_i} = \sum_j \frac{\partial \mathcal{L}}{\partial \gamma_j} \frac{\partial \gamma_j}{\partial c_i} \frac{\partial c_i}{\partial h_i}$
    *   Opacity: $\frac{\partial \mathcal{L}}{\partial \alpha_i} = \sum_j \frac{\partial \mathcal{L}}{\partial \gamma_j} \frac{\partial \gamma_j}{\partial \alpha'_{ij}} \frac{\partial \alpha'_{ij}}{\partial \alpha_i}$
    *   Position: $\frac{\partial \mathcal{L}}{\partial \mathbf{p}_{w,i}}$ involves terms for depth sorting, color, and covariance changes [S6, unofficial].
6.  **Optimization Step:** Update parameters using the Adam optimizer [S2][S6, unofficial].
7.  **Adaptive Density Control:** If the current step is a refinement iteration (e.g., every 100 steps [S4, unofficial] or 500 steps [S3, unofficial]), perform densification and pruning [S2]:
    *   **Pruning:** If opacity $\alpha < \epsilon$ or the Gaussian is too large ($||\mathbf{S}|| > \tau_S$), prune the Gaussian [S2].
    *   **Densification:** If the positional gradient $\nabla_{\mathbf{p}} \mathcal{L} > \tau_p$:
        *   **Over-reconstruction:** If $||\mathbf{S}|| > \tau_S$, split the Gaussian into two [S2].
        *   **Under-reconstruction:** If $||\mathbf{S}|| \le \tau_S$, clone the Gaussian [S2].
    *   Densification typically starts after step 500 and stops at step 15000 or half the total iterations [S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_3dgs(input_images, camera_poses, num_iters=30000, ssim_weight=0.2, densify_grad_thresh=0.0002, densify_size_thresh=0.01):
    # 1. Initialization
    gaussians = initialize_from_sfm(input_images, camera_poses)
    # gaussians contains: means (p_w), quaternions (q), scales (s), opacities (alpha), sh_coeffs (h)
    optimizer = Adam(gaussians.parameters(), lr=1e-4)
    
    camera_iterator = InfiniteRandomIterator(camera_poses)
    
    for step in range(1, num_iters + 1):
        # 2. Camera Sampling
        cam = camera_iterator.next()
        gt_image = get_ground_truth(cam, step) # Handles downscaling schedule
        
        # 3. Rendering
        rendered_image = rasterize(gaussians, cam)
        
        # 4. Loss Computation
        loss = (1 - ssim_weight) * l1_loss(rendered_image, gt_image) + ssim_weight * ssim_loss(rendered_image, gt_image)
        
        # 5. Backpropagation
        optimizer.zero_grad()
        loss.backward()
        
        # 6. Optimization Step
        optimizer.step()
        
        # 7. Adaptive Density Control (every refine_every steps)
        if step % refine_every == 0:
            prune_gaussians(gaussians, opacity_threshold=0.005)
            densify_gaussians(gaussians, grad_thresh=densify_grad_thresh, size_thresh=densify_size_thresh)
            
    return gaussians
```

**Parameters:**
*   `numIters` (iterations): Total number of optimization steps. Default: 30000 [S3, S4, unofficial].
*   `sh_degree`: Spherical harmonic degree. Default: 3 [S3, S4, unofficial].
*   `shDegreeInterval`: Steps between increasing SH degree. Default: 1000 [S3, S4, unofficial].
*   `resolutionSchedule`: Steps between doubling image resolution. Default: 3000 [S3, S4, unofficial].
*   `densifyFrom`: Step after which densification begins. Default: 500 [S3, unofficial].
*   `densifyUntil`: Step after which densification stops. Default: min(15000, numIters / 2) [S3, unofficial].
*   `densify_grad_thresh`: Gradient threshold for densification. Default: 0.0002 [S4, unofficial].
*   `densify_size_thresh`: Size threshold for densification. Default: 0.01 [S4, unofficial].
*   `ssim_weight`: Weight for SSIM loss. Default: 0.2 [S3, S4, unofficial].
*   `maxGaussians`: Maximum number of Gaussians. Default: 5,000,000 [S3, unofficial].

**Edge Cases and Pitfalls:**
*   **No Cameras See Points:** If all cameras are discarded because they see no sparse points, training fails [S3, unofficial].
*   **Arbitrary Pose Rendering:** Requires exactly 16 elements for the `cam_to_world` matrix [S4, unofficial].
*   **Evaluation Mode:** Dataset must be loaded with `eval_mode=True` for evaluation, otherwise it throws a runtime error [S4, unofficial].
*   **CPU vs GPU:** CPU execution is ~100x slower than GPU [S3, unofficial].

## Subtopics

### [Physical Parameter Optimization](4111-physical-parameter-optimization.md)
Replaces standard SH color and empirical opacity with physical quantities (extinction, albedo, scattering) to enable dynamic relighting.

### [WildGaussians](6064-wildgaussians.md)
Integrates robust DINO features and appearance modeling to handle unconstrained "in-the-wild" photo collections with varying illumination and transients.

### [Non-Pinhole Camera Training](6088-non-pinhole-camera-training.md)
Adapts training to non-pinhole cameras (e.g., with lens distortion) using a differentiable rasterizer that accommodates arbitrary projection models.

### [AbsGS](6127-absgs.md)
Uses absolute sums of positional gradients across views to prevent gradient cancellation, improving edge sharpness and reducing "fence-sitting" Gaussians.

### [Densification Strategy](6234-densification-strategy.md)
Defines mechanisms for dynamically managing Gaussian count via splitting, cloning, and pruning based on optimization metrics.

### [Regularization and Geometry Constraints](6269-regularization-and-geometry-constraints.md)
Introduces auxiliary constraints (e.g., normal consistency) to enforce geometric plausibility and prevent floaters.

### [Base 3DGS Optimization](7549-base-3dgs-optimization.md)
Initializes geometric attributes with an explicit normal attribute optimized via normal consistency loss to accelerate convergence.

### [GSP Training Loss](9156-gsp-training-loss.md)
Extends color loss with normal consistency and anisotropy regularization for dynamic/physical rendering contexts.

### [GPS-Gaussian](9197-gps-gaussian.md)
Regresses Gaussian parameters directly from sparse views using image-based regression, eliminating iterative optimization.

### [Optimization and Training Efficiency](9264-optimization-and-training-efficiency.md)
Strategies for scaling to large scenes, including multi-GPU distribution, backward pass restructuring, and half-precision rasterization.

### [Dynamic and Temporal Modeling](9265-dynamic-and-temporal-modeling.md)
Extends static 3DGS to reconstruct scenes changing over time via deformation or temporal attributes.

### [Compression and Representation Efficiency](9266-compression-and-representation-efficiency.md)
Methods to reduce memory and computational requirements through compression and representation changes.

### [SLAM and Mapping](9268-slam-and-mapping.md)
Integrates Simultaneous Localization and Mapping to jointly estimate camera poses and reconstruct scenes.

### [Application-Specific Reconstructions](9269-application-specific-reconstructions.md)
Extends 3DGS to domains like human avatars and robotics using domain-specific constraints.

### [Geometry and Surface Priors](9270-geometry-and-surface-priors.md)
Enforces consistency with SDFs, depth maps, or normals to enable surface reconstruction and mesh extraction.

### [Neural Network and Feed-Forward Approaches](9526-neural-network-and-feed-forward-approaches.md)
Uses neural networks to predict Gaussian parameters directly.

### [ClipGS](9546-clipgs.md)
Enables real-time interactive cinematic visualization of volumetric medical data with dynamic clipping planes.

### [Representation and Kernel Modifications](10145-representation-and-kernel-modifications.md)
Modifies Gaussian primitives or kernel functions (e.g., negative colors, voxel scaffolding) for better geometry.

### [TideGS Trajectory Ordering](10176-tidegs-trajectory-ordering.md)
Uses trajectory-aware ordered view sampling (TSP) instead of random shuffling for stable convergence.

### [Gradient Computation and Back-Propagation](10202-gradient-computation-and-back-propagation.md)
Details the mathematical foundation for computing gradients through alpha-compositing and coordinate transformations.

### [VisibilitySplat](10927-visibilitysplat.md)
Augments rasterization with a sparse visibility oracle for material-light decomposition.

### [MCMC-based 3D Gaussian Splatting](11914-mcmc-based-3d-gaussian-splatting.md)
Reformulates training as MCMC to replace heuristic densification with stochastic exploration.

### [3D Gaussian Splatting as Markov Chain Monte Carlo (MCMC)](11915-3d-gaussian-splatting-as-markov-chain-monte-carlo-mcmc.md)
Reinterprets training as sampling from a target distribution, treating structural changes as valid state transitions.

### [Sliced Wasserstein-based Gaussian Cloning Update](11919-sliced-wasserstein-based-gaussian-cloning-update.md)
Determines scale updates for cloned Gaussians by minimizing 1D slice integral differences.

### [Cross-Structure Collaboration Module](12071-cross-structure-collaboration-module.md)
Integrates global tri-plane features with local grid features for large-scale scene coherence.

### HUGS
HUGS creates a coarse-to-fine deformation module using linear blend skinning and local learning-based refinement for animating virtual human avatars based on 3DGS [S1]. It utilizes tri-plane representation to factorize canonical space, reconstructing persons from monocular video (50-100 frames) within 30 minutes at 20 FPS [S1].

### Wolf et al. Stereo Method
This method trains a 3DGS to render stereo-calibrated novel views, applies stereo depth estimation on the rendered views, and fuses the estimated dense depth maps using Truncated Signed Distance Function (TSDF) to form a triangular mesh [S1].

### Pseudo Ground Truth Acquisition for Gaussians
A labeling pipeline that generates Gaussian-specific pseudo ground truth labels from original point cloud labels, accounting for the densification and pruning processes of 3D-GS optimization [S8, unofficial].

### Cross-View Pruning Mechanism
A mechanism comprising a Cross-View Consistency Loss and adaptive pruning to reduce Gaussian redundancy [S9, unofficial]. It selects adjacent view pairs with SSIM > 0.6, computes a Cross-View Consistent Loss ($L_{CVC}$) to penalize residual discrepancies, and prunes Gaussians based on depth/distance consistency conditions [S9, unofficial].

### Structure-View Co-learning
A module combining structural and view-dependent gradients for robust joint optimization of Gaussian positions and appearance [S9, unofficial]. The loss is $\mathcal{L} = \lambda (\lambda_V \mathcal{L}_V + \lambda_S \mathcal{L}_S)$, where $\mathcal{L}_V$ is view gradient loss and $\mathcal{L}_S$ is structure gradient loss [S9, unofficial].

### Single-view Mini-batch Strategy
A training data sampling strategy where a mini-batch consists of data from only a single viewpoint [S10, unofficial]. Datasets are organized per viewpoint $D_i$, and a mini-batch $B_{3DGS}$ is sampled from $D$ with count=1 [S10, unofficial].

## Operating characteristics
*   **Time Complexity:** Training typically takes 15-30 minutes for standard datasets (e.g., Mip-NeRF 360) on modern GPUs [S5, unofficial]. The original implementation is significantly faster than NeRFs [S2][S5, unofficial].
*   **Memory Complexity:** Memory usage scales with the number of Gaussians. Default max is 5,000,000 Gaussians [S3, unofficial]. Sinc splatting variants consume less memory than standard Gaussians [S7].
*   **Measured Performance:** On the Replica dataset, SGS-SLAM achieves PSNR 34.15, SSIM 0.97, and LPIPS 0.096 [S1]. HUGS achieves 20 FPS for human avatars [S1].
*   **Scaling:** Scaling to large scenes requires optimization strategies like multi-GPU distribution and efficient gradient accumulation [S1, S2][S3, S4, S9, unofficial]. CPU execution is ~100x slower than GPU [S3, unofficial].
*   **Strengths:** Real-time rendering speeds, high fidelity, faster training than NeRFs [S2][S5, unofficial].
*   **Limitations:** Can produce "floaters" or over-reconstructed artifacts if not properly regularized [S2]. Sensitive to initialization and local minima in some variants [S11914, unofficial].

## Background
3D Gaussian Splatting was introduced by Kerbl et al. in 2023 [S2][S6, unofficial]. It initializes positions using sparse point clouds from SfM tools like COLMAP [S2][S6, unofficial]. The method has since been extended to various domains including SLAM [S1], dynamic scenes [S1], and human avatars [S1]. It is implemented in various frameworks including IPOL demos [S2], OpenSplat [S3, unofficial], and Msplat [S4, unofficial].

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
