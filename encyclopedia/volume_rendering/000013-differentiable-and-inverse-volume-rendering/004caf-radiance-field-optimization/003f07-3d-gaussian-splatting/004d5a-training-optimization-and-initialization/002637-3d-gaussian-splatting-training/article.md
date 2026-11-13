# 3D Gaussian Splatting Training

Also called: 3DGS Learning, G Splats

## Overview
3D Gaussian Splatting Training is the iterative optimization process that adjusts the parameters of a set of 3D Gaussian primitives to minimize the photometric difference between rendered images and ground truth views [S2][S6, unofficial]. It solves the problem of reconstructing high-fidelity 3D scenes from 2D images by converting sparse point clouds into a differentiable radiance field representation [S5, unofficial]. Unlike implicit neural fields (e.g., NeRFs), which require expensive volume rendering and neural network inference, 3DGS training optimizes explicit geometric primitives using a differentiable rasterizer, enabling significantly faster convergence and real-time rendering capabilities [S2][S5, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs:** A set of $N$ input images (or video frames) with known camera poses (extrinsics $R_{cw}, t_{cw}$ and intrinsics $K$) [S6, unofficial]. Initial 3D Gaussian parameters derived from Structure from Motion (SfM) [S2][S6, unofficial].
*   **Outputs:** Optimized 3D Gaussian parameters: positions $\mathbf{p}_i$, rotations $q_i$, scales $\mathbf{s}_i$, opacities $\alpha_i$, and spherical harmonic coefficients $h_i$ [S6, unofficial].

**Data Structures**
Each Gaussian primitive $i$ is defined by:
*   Position $\mathbf{p}_i \in \mathbb{R}^3$ (initialized from SfM points) [S6, unofficial].
*   Rotation $q_i \in \mathbb{R}^4$ (unit quaternion, initialized as constant) [S6, unofficial].
*   Scale $\mathbf{s}_i \in \mathbb{R}^3$ (initialized between 0 and 1) [S2][S6, unofficial].
*   Opacity $\alpha_i \in [0, 1]$ (initialized as constant) [S6, unofficial].
*   Appearance: Spherical Harmonic coefficients $h_i$ (initialized from SfM) [S6, unofficial].
*   Covariance $\Sigma_i \in \mathbb{R}^{3 \times 3}$: Derived from $\mathbf{s}_i$ and $q_i$ via rotation matrix $R(q_i)$ [S2].

**Numbered Steps**
1.  **Initialization:** Initialize positions $\mathbf{p}_i$ in normalized bounds $[-1, 1]$ (or from SfM points) [S2][S6, unofficial]. Initialize scales $\mathbf{s}_i$ between 0 and 1 [S2]. Initialize rotations $q_i$ as unit quaternions [S2]. Initialize opacities $\alpha_i$ and colors $h_i$ [S2][S6, unofficial]. Set iteration counter $i = 0$ [S2].
2.  **Camera Sampling:** Select a training camera $V$ and its corresponding ground truth image $\hat{I}$ [S2]. Cameras are typically sampled via round-robin with periodic shuffling [S4, unofficial].
3.  **Rendering:** Project the 3D Gaussians onto the 2D image plane using camera $V$ to obtain rendered image $I$ [S2]. This involves computing the 2D covariance matrix $\Sigma'$ and projecting means to screen space [S6, unofficial].
4.  **Loss Computation:** Compute the loss $\mathcal{L}$ between the rendered image $I$ and ground truth $\hat{I}$ [S2]. The loss is a combination of L1 loss ($\mathcal{L}_1$) and D-SSIM loss ($\mathcal{L}_{D-SSIM}$) weighted by $\lambda$ [S6, unofficial]:
    $$ \mathcal{L} = (1 - \lambda) \mathcal{L}_1 + \lambda \mathcal{L}_{D-SSIM} $$
    Typical default for $\lambda$ (ssim_weight) is 0.2 [S4, unofficial].
5.  **Backpropagation:** Compute gradients $\nabla \mathcal{L}$ for all parameters ($\mathbf{p}_w, q, \mathbf{s}, h, \alpha$) using the chain rule through the 5 forward steps of image generation [S6, unofficial].
6.  **Optimization Step:** Update parameters using the Adam optimizer [S2][S6, unofficial].
7.  **Adaptive Density Control:** Periodically (every `refineEvery` steps, default 500 [S3, unofficial]), perform densification and pruning:
    *   **Pruning:** If opacity $\alpha_i < \epsilon$ or the Gaussian is too large ($\|\mathbf{s}_i\| > \tau_S$), prune the Gaussian [S2].
    *   **Densification Check:** If the positional gradient magnitude $\|\nabla_{\mathbf{p}} \mathcal{L}\| > \tau_p$ (default 0.0002 [S4, unofficial]):
        *   **Over-reconstruction:** If $\|\mathbf{s}_i\| > \tau_S$ (default 0.01 [S4, unofficial]), split the Gaussian into two [S2].
        *   **Under-reconstruction:** If $\|\mathbf{s}_i\| \leq \tau_S$, clone the Gaussian [S2].
8.  **Resolution Schedule:** Increase image resolution every `resolutionSchedule` steps (default 3000 [S3, unofficial]).
9.  **SH Degree Increase:** Increase spherical harmonic degree every `shDegreeInterval` steps (default 1000 [S3, unofficial]) up to `shDegree` (default 3 [S3, unofficial]).
10. **Termination:** Stop after `numIters` (default 30,000 [S3, unofficial]) or when converged.

**Formulas**
*   **Rotation Matrix from Quaternion:**
    Given $q = (q_r, q_i, q_j, q_k)$:
    $$ R(q) = 2 \begin{bmatrix} \frac{1}{2} - (q_j^2 + q_k^2) & q_i q_j - q_r q_k & q_i q_k + q_r q_j \\ q_i q_j + q_r q_k & \frac{1}{2} - (q_i^2 + q_k^2) & q_j q_k - q_r q_i \\ q_i q_k - q_r q_j & q_j q_k + q_r q_i & \frac{1}{2} - (q_i^2 + q_j^2) \end{bmatrix} $$
    [S2]
*   **Pixel Color (Alpha Blending):**
    $$ \gamma_j = \sum_{i \in N} \alpha'_{ij} c_i \tau_{ij}, \quad \tau_{ij} = \prod_{k=1}^{i-1} (1 - \alpha'_{kj}) $$
    where $N$ is the number of Gaussians sorted by depth, $\alpha'_{ij}$ is effective opacity, $c_i$ is color, and $\tau_{ij}$ is transmittance [S6, unofficial].
*   **Effective Opacity:**
    $$ \alpha'_{ij} = g \alpha_i $$
    where $g = \exp(-0.5 \omega'_0 d_0^2 - 0.5 \omega'_2 d_1^2 - \omega'_1 d_0 d_1)$, $\omega'$ are upper triangular elements of the inverse 2D covariance matrix, and $d$ are elements of $u_i - x_j$ (difference between projected mean and pixel center) [S6, unofficial].
*   **Jacobian for Position $\mathbf{p}_{w,i}$:**
    $$ \frac{\partial \mathcal{L}}{\partial \mathbf{p}_{w,i}} = \sum_j \frac{\partial \mathcal{L}}{\partial \gamma_j} \frac{\partial \gamma_j}{\partial \alpha'_{ij}} \frac{\partial \alpha'_{ij}}{\partial u_i} \frac{\partial u_i}{\partial \mathbf{p}_{c,i}} \frac{\partial \mathbf{p}_{c,i}}{\partial \mathbf{p}_{w,i}} + \sum_j \frac{\partial \mathcal{L}}{\partial \gamma_j} \frac{\partial \gamma_j}{\partial c_i} \frac{\partial c_i}{\partial \mathbf{p}_{w,i}} + \sum_j \frac{\partial \mathcal{L}}{\partial \gamma_j} \frac{\partial \gamma_j}{\partial \alpha'_{ij}} \frac{\partial \alpha'_{ij}}{\partial \sigma'_i} \frac{\partial \sigma'_i}{\partial \mathbf{p}_{c,i}} \frac{\partial \mathbf{p}_{c,i}}{\partial \mathbf{p}_{w,i}} $$
    [S6, unofficial]
*   **Jacobian for Rotation $q_i$:**
    $$ \frac{\partial \mathcal{L}}{\partial q_i} = \sum_j \frac{\partial \mathcal{L}}{\partial \gamma_j} \frac{\partial \gamma_j}{\partial \sigma'_i} \frac{\partial \sigma'_i}{\partial \sigma_i} \frac{\partial \sigma_i}{\partial q_i} $$
    [S6, unofficial]
*   **Jacobian for Scale $\mathbf{s}_i$:**
    $$ \frac{\partial \mathcal{L}}{\partial \mathbf{s}_i} = \sum_j \frac{\partial \mathcal{L}}{\partial \gamma_j} \frac{\partial \gamma_j}{\partial \sigma'_i} \frac{\partial \sigma'_i}{\partial \sigma_i} \frac{\partial \sigma_i}{\partial \mathbf{s}_i} $$
    [S6, unofficial]
*   **Jacobian for SH $h_i$:**
    $$ \frac{\partial \mathcal{L}}{\partial h_i} = \sum_j \frac{\partial \mathcal{L}}{\partial \gamma_j} \frac{\partial \gamma_j}{\partial c_i} \frac{\partial c_i}{\partial h_i} $$
    [S6, unofficial]
*   **Jacobian for Opacity $\alpha_i$:**
    $$ \frac{\partial \mathcal{L}}{\partial \alpha_i} = \sum_j \frac{\partial \mathcal{L}}{\partial \gamma_j} \frac{\partial \gamma_j}{\partial \alpha'_{ij}} \frac{\partial \alpha'_{ij}}{\partial \alpha_i} $$
    [S6, unofficial]

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_3dgs(input_images, camera_poses, initial_points, num_iters=30000):
    # 1. Initialization
    gaussians = []
    for p in initial_points:
        g = Gaussian()
        g.position = p
        g.rotation = unit_quaternion() # Constant init
        g.scale = random_uniform(0, 1) # Constant init
        g.opacity = 0.5 # Constant init
        g.sh_coeffs = get_sh_from_sfm(p)
        gaussians.append(g)
    
    step = 0
    while step < num_iters:
        # 2. Camera Sampling
        cam = sample_camera(step, camera_poses)
        gt_image = get_ground_truth(cam, step)
        
        # 3. Rendering
        rendered_image = rasterize(gaussians, cam)
        
        # 4. Loss Computation
        loss = compute_loss(rendered_image, gt_image)
        
        # 5. Backpropagation
        gradients = compute_gradients(loss, gaussians, cam)
        
        # 6. Optimization Step
        update_parameters(gaussians, gradients)
        
        # 7. Adaptive Density Control (every 500 steps)
        if step % 500 == 0:
            densify_and_prune(gaussians, gradients)
            
        # 8. Resolution Schedule (every 3000 steps)
        if step % 3000 == 0:
            increase_resolution()
            
        # 9. SH Degree Increase (every 1000 steps)
        if step % 1000 == 0:
            increase_sh_degree()
            
        step += 1
        
    return gaussians

def densify_and_prune(gaussians, gradients):
    new_gaussians = []
    for g in gaussians:
        # Pruning
        if g.opacity < 0.005 or norm(g.scale) > 0.01:
            continue # Prune
            
        grad_norm = norm(gradients[g].position)
        
        # Densification
        if grad_norm > 0.0002:
            if norm(g.scale) > 0.01:
                # Over-reconstruction: Split
                g1, g2 = split_gaussian(g)
                new_gaussians.append(g1)
                new_gaussians.append(g2)
            else:
                # Under-reconstruction: Clone
                g_clone = clone_gaussian(g)
                new_gaussians.append(g)
                new_gaussians.append(g_clone)
        else:
            new_gaussians.append(g)
            
    gaussians.clear()
    gaussians.extend(new_gaussians)
```

## Subtopics

### [Physical Parameter Optimization](4111-physical-parameter-optimization.md)
Physical Parameter Optimization replaces the standard spherical harmonic (SH) color and empirical opacity attributes of 3D Gaussian Splatting with a set of physical quantities representing a participating medium. This allows for real-time rendering and dynamic relighting (arbitrary sun direction) by solving the radiative transfer equation through the Gaussian primitives rather than approximating lighting with SH coefficients. It is distinct from standard 3DGS training by optimizing extinction, albedo, and scattering asymmetry instead of color and opacity.

### [WildGaussians](6064-wildgaussians.md)
WildGaussians (SWAG) is a 3D Gaussian Splatting method designed for "in-the-wild" 3D reconstruction from unconstrained photo collections characterized by transient objects, occlusions, and varying illumination. It solves the problem of photometric inconsistency and transient artifacts that cause standard 3DGS to produce floaters or color-shifted renders in such settings. It sets itself apart from its siblings by explicitly modeling image-dependent appearance variations via a lightweight MLP and separating static scene geometry from transient occluders using a learnable opacity variation term, enabling the discarding of transient Gaussians to recover a clean static scene. Unlike [ClipGS](9546-clipgs.md) which uses CLIP features, SWAG relies on DINO features and direct color prediction.

### [Non-Pinhole Camera Training](6088-non-pinhole-camera-training.md)
Non-Pinhole Camera Training is a method for adapting 3D Gaussian Splatting (3DGS) to scenes captured by non-pinhole cameras, such as those with significant lens distortion or complex optical properties. It solves the problem where standard 3DGS, which assumes a pinhole camera model, fails to reconstruct scenes from such data. It differs from standard training by leveraging a differentiable rasterizer that can accommodate arbitrary camera projection models, allowing the optimization to proceed despite the deviation from ideal pinhole geometry.

### AbsGS
AbsGS (Absolute Positional Gradients) is a densification strategy for 3D Gaussian Splatting that addresses the smoothing of fine details and thin structures during optimization [S1]. It improves upon standard view-space gradient accumulation by using absolute sums of positional gradients across multiple views, preventing gradient collisions where positive and negative gradients cancel each other out [S3, unofficial]. This allows the method to identify "fence-sitting" Gaussians more accurately and sharpen edges [S2].

### [Densification Strategy](6234-densification-strategy.md)
Densification strategies manage the topology of the 3D Gaussian primitive set $\mathcal{G}$ during the training of 3D Gaussian Splatting (3DGS) by dynamically adding, splitting, cloning, or removing primitives based on optimization metrics [13466-densification-and-splitting.md]. This family solves the problem of under-representing complex scene details or over-representing empty space, which static initialization methods cannot address [13466-densification-and-splitting.md]. What sets this family apart from static initialization or feed-forward approaches is the iterative, feedback-driven adaptation of the primitive population throughout the training process, balancing reconstruction quality against computational cost [13466-densification-and-splitting.md].

### [Regularization and Geometry Constraints](6269-regularization-and-geometry-constraints.md)
Regularization and Geometry Constraints are training-time techniques that modify the optimization objective or Gaussian parameters to enforce geometric plausibility, prevent over-reconstruction, and stabilize convergence in 3D Gaussian Splatting. They solve the problem where standard photometric losses (e.g., L1, SSIM) fail to distinguish between well-reconstructed surfaces and redundant "floaters," leading to blurred details, geometric hollowness, or unstable camera poses. This family sets itself apart from [Base 3DGS Optimization](7549-base-3d-gaussian-splatting-optimization.md) by explicitly introducing auxiliary constraints beyond pixel-color matching, and from [Physical Parameter Optimization](4111-physical-parameter-optimization.md) by focusing on structural integrity rather than physical light transport properties.

### [Base 3DGS Optimization](7549-base-3dgs-optimization.md)
Base 3DGS Optimization initializes the geometric attributes of 3D Gaussian primitives to provide a robust geometric baseline for subsequent shading attribute optimization, such as in [iVR-GS Training](7578-ivr-gs-training.md). It differs from standard 3DGS by incorporating an explicit normal attribute $n$ for each Gaussian and optimizing it via a normal consistency loss, despite the normal not affecting the final rendered color or alpha in the base stage. This initialization accelerates the convergence of editable Gaussian optimization by ensuring accurate surface geometry representation before decomposing color and lighting.

### [GSP Training Loss](9156-gsp-training-loss.md)
The GSP Training Loss is the objective function used to optimize the parameters of 3D Gaussian kernels, including position, opacity, covariance, and material properties, within the Gaussian Splashing (GSP) framework. It extends the standard color reconstruction loss with normal consistency and anisotropy regularization to prevent artifacts caused by excessive kernel elongation during solid object simulation. Unlike standard 3D Gaussian Splatting which focuses on static geometry, this loss is designed to maintain geometric stability and physical plausibility in dynamic or physically-based rendering contexts.

### [GPS-Gaussian](9197-gps-gaussian.md)
GPS-Gaussian is a training initialization and acceleration method for 3D Gaussian Splatting that eliminates the need for iterative optimization by regressing Gaussian parameters directly from sparse source views. It solves the problem of slow convergence and high computational cost in standard 3DGS training by introducing Gaussian parameter maps on sparse input images, jointly regressing the 3D Gaussian attributes alongside a depth estimation module. It is distinguished from standard [Base 3DGS Optimization](7549-base-3d-gaussian-splatting-optimization.md) by its ability to produce a high-quality initial state without any fine-tuning or optimization steps, and from [FaCT-GS](10199-fact-gs.md) by its reliance on image-based regression rather than voxelization-based initialization.

### [Optimization and Training Efficiency](9264-optimization-and-training-efficiency.md)
This family encompasses the computational strategies, memory management techniques, and hyperparameter configurations required to efficiently train 3D Gaussian Splatting (3DGS) models. It solves the problem of high memory bandwidth consumption, VRAM bottlenecks, and slow convergence inherent in differentiable volume rendering with billions of primitives. It sets itself apart from standard differentiable rendering by explicitly managing the dynamic population of Gaussians (densification/pruning) and leveraging hardware-specific optimizations like half-precision rasterization, fused kernels, and out-of-core data streaming.

### [Dynamic and Temporal Modeling](9265-dynamic-and-temporal-modeling.md)
Dynamic and Temporal Modeling extends static radiance field representations (such as NeRF and 3D Gaussian Splatting) to handle scenes that change over time, enabling the synthesis of novel views at arbitrary timestamps. It solves the problem of modeling non-rigid geometry and appearance variations that static methods cannot capture. Unlike static methods, this family explicitly parameterizes time, either by warping coordinates to a canonical space, conditioning neural fields on temporal inputs, or parameterizing Gaussian attributes as time-dependent functions. It is distinguished from sibling methods like [Base 3DGS Optimization](7549-base-3d-gaussian-splatting.md) by the introduction of temporal degrees of freedom, and from [WildGaussians](6064-wildgaussians.md) by its focus on continuous or structured temporal modeling rather than just handling wild camera motion in static scenes.

### [Compression and Representation Efficiency](9266-compression-and-representation-efficiency.md)

### [SLAM and Mapping](9268-slam-and-mapping.md)
This family encompasses methods that integrate Simultaneous Localization and Mapping (SLAM) with 3D Gaussian Splatting (3DGS) or hybrid neural field representations to jointly estimate camera poses and reconstruct scene geometry/appearance. It solves the problem of real-time, high-fidelity 3D reconstruction and localization in dynamic or large-scale environments where traditional point-cloud or mesh-based SLAM struggles with textureless regions or complex lighting. It is distinguished from sibling methods like [Physical Parameter Optimization](4111-physical-parameter-optimization.md) by its focus on spatial-temporal consistency and pose estimation rather than material properties, and from [WildGaussians](6064-wildgaussians.md) by its explicit handling of camera trajectory optimization and loop closure.

### [Application-Specific Reconstructions](9269-application-specific-reconstructions.md)
Application-Specific Reconstructions modify the standard 3D Gaussian Splatting (3DGS) training loop to address domain-specific challenges such as dynamic human avatars, displaced objects, robotics simulation, and scientific inverse problems. Unlike [Base 3DGS Optimization](7549-base-3d-gaussian-splatting.md), which assumes a static, rigid scene, these methods introduce custom initialization strategies, deformation priors, and generative repair mechanisms to handle non-rigid motion, occlusion artifacts, and translucent materials. They are chosen when the standard densification and optimization pipeline fails to converge on complex geometries or dynamic scenes.

### [Geometry and Surface Priors](9270-geometry-and-surface-priors.md)
Geometry and Surface Priors address the inherent ambiguity and lack of explicit surface structure in standard 3D Gaussian Splatting (3DGS), which produces floating ellipsoids as an emergent property rather than a defined mesh. This family of methods enforces geometric consistency by coupling the discrete Gaussian representation with continuous geometric signals, such as Signed Distance Functions (SDFs) or monocular depth estimates. Unlike [Base 3DGS Optimization](7549-base-3d-gaussian-splatting-optimization.md), which relies solely on photometric rendering loss, these techniques explicitly constrain Gaussian positions, orientations, and scales to align with known or estimated surface geometry.

### [Neural Network and Feed-Forward Approaches](9526-neural-network-and-feed-forward-approaches.md)

### ClipGS
ClipGS is a Gaussian Splatting-based framework designed for the real-time interactive cinematic visualization of volumetric medical data with support for dynamic clipping planes [S1]. It solves the problem of maintaining rendering quality and continuity when a user interactsively clips a 3D volume, addressing the artifacts caused by hard visibility truncations and the high computational cost of learning full deformation fields for the additional clipping dimension [S1]. It sets itself apart from sibling methods like [WildGaussians](6064-wildgaussians.md) or [Dynamic and Temporal Modeling](9265-dynamic-and-temporal-modeling.md) by treating the clipping plane not as a temporal dimension requiring a dense deformation field, but as a geometric visibility constraint managed via a learnable truncation scheme and an Adaptive Adjustment Model (AAM) [S1].

### [Representation and Kernel Modifications](10145-representation-and-kernel-modifications.md)
Representation and Kernel Modifications extend the standard 3D Gaussian Splatting (3DGS) primitive definition and rendering kernel to address limitations in shape expressivity, geometric consistency, and surface reconstruction. While standard 3DGS relies on isotropic/anisotropic positive Gaussians with exponential kernels, this family introduces negative colors for shape cancellation, voxel-based scaffolds for structured priors, tangent-plane densification for geometric smoothness, non-exponential Radial Basis Functions (RBFs) for faster convergence, and oriented half-space primitives for watertight mesh extraction. These methods solve the issues of high-frequency detail loss, noisy normals, and non-watertight surfaces inherent in the base representation.

### [TideGS Trajectory Ordering](10176-tidegs-trajectory-ordering.md)
TideGS Trajectory Ordering is a training strategy for 3D Gaussian Splatting that replaces standard randomized view shuffling with a trajectory-aware ordered sequence of views to improve optimization stability and convergence for large-scale scenes. Unlike standard 3DGS which samples views with random shuffling, TideGS uses a clustered traveling-salesperson (TSP) ordering to preserve trajectory continuity, reducing gradient variation between consecutive steps. It builds on [Base 3DGS Optimization](7549-base-3d-gaussian-splatting-optimization.md) by modifying the view sampling schedule while retaining the standard masked update mechanism.

### [Gradient Computation and Back-Propagation](10202-gradient-computation-and-back-propagation.md)
Gradient Computation and Back-Propagation defines the mathematical and algorithmic mechanisms for propagating loss gradients from rendered pixels back to the parameters of 3D Gaussian primitives (position, scale, rotation, opacity, color) and camera intrinsics/extrinsics. It solves the problem of optimizing radiance fields by enabling end-to-end training via automatic differentiation or custom analytical backward passes. It sets itself apart from sibling methods by focusing on the core differentiability of the rendering equation, whereas siblings like [Physical Parameter Optimization](4111-physical-parameter-optimization.md) or [WildGaussians](6064-wildgaussians.md) extend or modify this core gradient path for specific domains (e.g., ToF, semantic features, or dynamic scenes).

### [VisibilitySplat](10927-visibilitysplat.md)
VisibilitySplat is a two-stage inverse rendering pipeline for 3D Gaussian Splatting (3DGS) that augments the standard rasterized pipeline with a sparse visibility oracle to estimate transmittance-weighted occlusion for Gaussian primitives from sampled lighting directions, enabling material and light decomposition. It addresses the limitation of standard 3DGS rasterization, which provides limited visibility information regarding cast shadows and mutual occlusion, by integrating a BVH-accelerated point-based ray tracer. Unlike NeRF-based methods that use implicit volumetric formulations, VisibilitySplat uses explicit point primitives via differentiable rasterization for real-time rendering while providing robust visibility signals for stabilizing material-light disentanglement under unknown illumination.

### [MCMC-based 3D Gaussian Splatting](11914-mcmc-based-3d-gaussian-splatting.md)
This method reformulates 3D Gaussian Splatting (3DGS) training as a Markov Chain Monte Carlo (MCMC) process, replacing the heuristic-based densification and pruning of [Base 3DGS Optimization](7549-base-3d-gaussian-splatting-optimization.md) with a stochastic exploration mechanism. It solves the problem of sensitivity to initialization and local minima by allowing Gaussians to explore the scene space via noise injection and relocation, rather than relying on fixed thresholds for cloning and splitting. Unlike [Base 3DGS Optimization](7549-base-3d-gaussian-splatting-optimization.md), which uses deterministic heuristics, this approach is robust to random initialization and prevents "stray" Gaussians from diverging into empty space through specific regularizers.

### [3D Gaussian Splatting as Markov Chain Monte Carlo (MCMC)](11915-3d-gaussian-splatting-as-markov-chain-monte-carlo-mcmc.md)
This method reinterprets the training of 3D Gaussian Splatting as a sampling process from a target distribution $\mathcal{G}$, rather than a deterministic minimization of a loss function via gradient descent. By framing the optimization as Markov Chain Monte Carlo (MCMC), it enables discrete structural changes (moving, cloning, pruning Gaussians) to be treated as valid state transitions that preserve probability mass, avoiding the discontinuities that break standard gradient-based assumptions. It differs from [Base 3DGS Optimization](7549-base-3d-gaussian-splatting-optimization.md) by injecting anisotropic noise into the mean updates and rewriting heuristics as MCMC moves, and from [Physical Parameter Optimization](4111-physical-parameter-optimization.md) by focusing on the probabilistic interpretation of the training dynamics rather than physical constraints.

### [Sliced Wasserstein-based Gaussian Cloning Update](11919-sliced-wasserstein-based-gaussian-cloning-update.md)
A method to determine the scale update for cloned Gaussians by minimizing the difference between the integrated color of the new and old Gaussians along arbitrary 1D slices passing through their centers, ensuring rasterization integral conservation. It sets itself apart from standard densification by enforcing a specific integral constraint on 1D projections rather than relying solely on position gradient magnitudes, leading to improved PSNR, SSIM, and lower LPIPS compared to base 3DGS.

### [Cross-Structure Collaboration Module](12071-cross-structure-collaboration-module.md)
The Cross-Structure Collaboration Module (CSCM) is a feature extraction mechanism for 3D Gaussian Splatting that integrates global tri-plane features with local context-aware grid features to generate Hierarchical Dense Embedded (HDE) features. It solves the problem of feature misalignment and insufficient geometric constraints in large-scale scenes by preserving global coherence while capturing local detail. It sets itself apart from standard tri-plane methods by adding a context grid interpolation step and from pure grid methods by maintaining a static global tri-plane backbone.

### HUGS
HUGS creates a coarse-to-fine deformation module using linear blend skinning and local learning-based refinement for constructing and animating virtual human avatars based on 3DGS [S1]. It utilizes the tri-plane representation to factorize the canonical space, allowing reconstruction of the person and scene from monocular video (50-100 frames) within 30 minutes [S1]. It achieves state-of-the-art human neural rendering performance at 20 FPS [S1].

### Wolf et al. Stereo Method
Wolf et al. train a 3DGS to render stereo-calibrated novel views [S1]. They apply stereo depth estimation on the rendered views and fuse the estimated dense depth maps together by the Truncated Signed Distance Function (TSDF) to form a triangular mesh [S1].

### Pseudo Ground Truth Acquisition for Gaussians
A labeling pipeline that generates Gaussian-specific pseudo ground truth labels from original point cloud labels, accounting for the densification and pruning processes of 3D-GS optimization [S8, unofficial].

### Cross-View Pruning Mechanism
A mechanism comprising a Cross-View Consistency Loss and adaptive pruning to reduce Gaussian redundancy and ensure geometric validity [S9, unofficial]. It selects adjacent view pairs from $M$ views based on SSIM > 0.6, computes Cross-View Consistent Loss ($L_{CVC}$) to penalize discrepancies between residuals of paired views, and prunes based on depth/distance consistency ($d^i_p < 0.01 \land d^j_p < 0.5 \lor d^i_p > 3\sigma$) [S9, unofficial].

### Structure-View Co-learning
A module combining structural and view-dependent gradients for robust joint optimization of Gaussian positions and appearance [S9, unofficial]. The loss is defined as $\mathcal{L} = \lambda (\lambda_V \mathcal{L}_V + \lambda_S \mathcal{L}_S)$, where $\mathcal{L}_V$ is view gradient loss and $\mathcal{L}_S$ is structure gradient loss [S9, unofficial].

### Single-view Mini-batch Strategy
A training data sampling strategy for 3D Gaussian Splatting where data is organized per viewpoint, and a mini-batch consists of data from only a single viewpoint [S10, unofficial]. For each viewpoint $i$ from 1 to $N$, the dataset $D_i$ containing pixel-color pairs $\{p_{ij}, C_{ij}\}$ is collected. The datasets are organized into a collection $D = \{D_i\}$ for all $N$ viewpoints. A mini-batch $B_{3DGS}$ is sampled containing data from count=1 viewpoint from $D$ [S10, unofficial].

## Operating characteristics
*   **Time Complexity:** Training time is significantly faster than NeRFs, typically completing in minutes to hours depending on scene complexity and resolution [S2][S5, unofficial]. The original 3DGS implementation takes approximately 2-3 hours for standard scenes [S5, unofficial].
*   **Memory Complexity:** Memory usage scales with the number of Gaussians. The default maximum number of Gaussians is 5,000,000 [S3, unofficial]. Memory consumption is higher than NeRFs due to the explicit storage of Gaussian parameters [S7].
*   **Measured Performance:** On the Replica dataset, SGS-SLAM achieves PSNR 34.15, SSIM 0.97, and LPIPS 0.096 [S1].
*   **Scaling:** Performance scales with the number of input views and resolution. Higher resolutions and more views increase training time and memory usage [S4, unofficial].
*   **Strengths:** Real-time rendering speeds, high visual fidelity, fast training compared to NeRFs [S2][S5, unofficial].
*   **Limitations:** Can produce "floaters" or artifacts in dynamic scenes or with insufficient views [S3, unofficial]. High memory usage for large scenes [S7].
*   **When to use:** When real-time rendering and fast training are required for static or slowly changing scenes [S2][S5, unofficial].
*   **When not to use:** For scenes with significant dynamic objects or transient effects without modifications (e.g., WildGaussians) [S3, unofficial]. For applications requiring explicit mesh extraction without post-processing [S1].

## Background
3D Gaussian Splatting was introduced by Kerbl et al. in 2023 [S2]. It builds on the concept of Gaussian Splatting, which was originally proposed for 2D image editing [S2]. The method was developed to address the slow training and rendering speeds of Neural Radiance Fields (NeRFs) [S5, unofficial]. It uses Structure from Motion (SfM) tools like COLMAP to initialize the 3D point cloud [S2][S6, unofficial]. The method has since been extended to various domains including SLAM, dynamic scenes, and human avatars [S1].

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
