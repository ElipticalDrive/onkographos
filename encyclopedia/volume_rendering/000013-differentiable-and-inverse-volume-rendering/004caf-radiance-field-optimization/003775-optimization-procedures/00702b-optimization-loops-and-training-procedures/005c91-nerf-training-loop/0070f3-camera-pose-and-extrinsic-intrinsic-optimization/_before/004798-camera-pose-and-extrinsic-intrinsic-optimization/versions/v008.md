# Camera Pose and Extrinsic/Intrinsic Optimization

Also called: Joint Pose and NeRF Optimization, Camera Pose Optimization, Inverse Rendering for Pose

## Overview
This family of methods addresses the problem of optimizing camera parameters (extrinsics and intrinsics) simultaneously with Neural Radiance Field (NeRF) weights, eliminating the reliance on pre-computed Structure-from-Motion (SfM) pipelines like COLMAP [G]. By treating camera poses as learnable parameters, these methods enable 3D reconstruction from sparse or uncalibrated image sets [G]. What sets this family apart from siblings like [Time-dependent NeRF with Continuous Camera Motion](12399-time-dependent-nerf-with-continuous-camera-motion.md) is the explicit optimization of discrete camera poses per image rather than continuous motion functions, and from [Camera Preconditioning](12545-camera-preconditioning.md) by focusing on the joint gradient-based update of poses and scene geometry rather than preconditioning the optimizer for faster convergence [G].

## Shared principles
All methods in this family share the core mechanism of differentiable rendering, where the rendered pixel color $\hat{C}(r)$ is a differentiable function of the camera pose $P$ and the NeRF parameters $\Theta$ [G]. The optimization problem is typically formulated as minimizing a loss function $\mathcal{L}$ with respect to both $\Theta$ and $P$:
$$ \min_{\Theta, P} \mathcal{L}(\hat{C}(r; \Theta, P), C(r)) $$
where $C(r)$ is the ground-truth pixel color [G]. The gradient $\nabla_P \mathcal{L}$ allows the camera pose to be updated via an optimizer (e.g., Adam, SGD) in lockstep with the NeRF weights [G]. Key shared components include:
1.  **Ray Casting:** Computing ray origins $\hat{o}$ and directions $\hat{d}$ from pixel coordinates using current camera estimates [G]. For a pinhole camera model with focal length $f$ and rotation $R$ and translation $t$:
    $$ \hat{o} = t $$
    $$ \hat{d} = R \begin{pmatrix} (u - W/2)/f \\ -(v - H/2)/f \\ -1 \end{pmatrix} $$
2.  **Differentiable Rendering:** The volume rendering integral is computed via discrete summation, allowing backpropagation through the ray marching process to the pose parameters [G].
3.  **Joint Optimization:** Separate optimizers or a combined optimizer are used to update NeRF weights and pose parameters, often with different learning rates to stabilize convergence [G].

## Subtopics

### [Camera Pose and Parameter Optimization](7611-camera-pose-and-parameter-optimization.md)
This family of methods addresses the problem of optimizing camera parameters (extrinsics and intrinsics) simultaneously with Neural Radiance Field (NeRF) weights, eliminating the reliance on pre-computed Structure-from-Motion (SfM) pipelines like COLMAP [G]. By treating camera poses as learnable parameters, these methods enable 3D reconstruction from sparse or uncalibrated image sets [G]. What sets this family apart from siblings like [Time-dependent NeRF with Continuous Camera Motion](12399-time-dependent-nerf-with-continuous-camera-motion.md) is the explicit optimization of discrete camera poses per image rather than continuous motion functions, and from [Camera Preconditioning](12545-camera-preconditioning.md) by focusing on the joint gradient-based update of poses and scene geometry rather than preconditioning the optimizer for faster convergence [G].

### [MELON](11909-melon.md)
MELON is a differentiable inverse rendering method that jointly optimizes a Neural Radiance Field (NeRF) and camera poses from a set of unposed images. It solves the problem of local minima in the low-dimensional $SO(3)$ latent space by introducing a "modulo loss" that maps the encoder's output to a quotient space, effectively replicating the latent variables to match the object's rotational symmetries. Unlike adversarial approaches such as GNeRF, MELON uses an encoder-decoder architecture with amortized inference, allowing it to perform ab-initio reconstruction without pose initialization or pre-trained CNNs. Does not require a discriminator.

### [Time-dependent NeRF with Continuous Camera Motion](12399-time-dependent-nerf-with-continuous-camera-motion.md)
This method jointly optimizes a time-dependent Neural Radiance Field (NeRF) and continuous camera motion from monocular video, representing the scene as local Signed Distance Field (SDF) and appearance fields at each time step. Unlike methods that train a single global NeRF mapped via noisy initial transformations, this approach uses local time-dependent representations constrained by consistency losses to avoid local minima. It builds on [Camera Pose and Extrinsic/Intrinsic Optimization](12051-camera-pose-and-extrinsic-intrinsic-optimization.md) by modeling camera motion as continuous velocities rather than discrete pose parameters.

### [Camera Preconditioning](12545-camera-preconditioning.md)
Camera Preconditioning (CamP) is a technique to normalize and decorrelate camera parameters during Neural Radiance Field (NeRF) optimization by transforming the optimization problem from direct camera parameters $\boldsymbol{\phi}_i$ to latent preconditioned parameters $\tilde{\boldsymbol{\phi}}_i$, using a preconditioner matrix $P_i^{-1}$ derived from the covariance of projection derivatives. It addresses issues where parameters are correlated or have different units (e.g., translation in world units vs. focal length in pixels), which causes ill-conditioning in naive optimization. Unlike per-parameter scaling or adaptive optimizers like Adam, CamP decouples highly correlated coefficients by using the full covariance matrix, ensuring that parameter updates are scaled according to their sensitivity to the rendered image.

### [SPARF](12705-sparf.md)
SPARF is a joint pose-NeRF training strategy designed for novel view synthesis from sparse input views (as few as 3) with noisy camera pose estimates. It solves the problems of overfitting and geometric inconsistency inherent in sparse-view regimes by enforcing multi-view geometry constraints, specifically multi-view correspondence and depth consistency losses, rather than relying solely on photometric loss. Unlike sibling methods that optimize poses and NeRF weights using independent per-image photometric supervision, SPARF explicitly connects training views through pixel correspondences to drive convergence to a globally accurate pose and scene solution.

### [Test-Time Two-Stage Optimization](13119-test-time-two-stage-optimization.md)
Test-Time Two-Stage Optimization is a post-training evaluation procedure that first optimizes camera poses and then optimizes appearance embeddings to handle unconstrained image collections with varying lighting and transient objects. It distinguishes itself from sibling methods by decoupling pose refinement and appearance adaptation into two sequential stages, addressing the limitations of methods that optimize only poses (e.g., BARF) or only appearance (e.g., NeRF-W).

### [Joint Pose and Radiance Field Optimization](17928-joint-pose-and-radiance-field-optimization.md)
Joint Pose and Radiance Field Optimization solves the problem of reconstructing a 3D scene from an unposed image collection by simultaneously optimizing the neural radiance field parameters and the camera extrinsic (and optionally intrinsic) parameters. It distinguishes itself from methods that assume known poses by treating camera poses as learnable variables in the optimization loop, thereby addressing the "chicken-and-egg" problem where pose estimation requires a good scene model and vice versa. This approach builds on [Camera Pose and Extrinsic/Intrinsic Optimization](12051-camera-pose-and-extrinsic-intrinsic-optimization.md) by extending the optimization domain to include the scene representation itself.

### [Sync-NeRF Offset Optimization](17949-sync-nerf-offset-optimization.md)
Sync-NeRF Offset Optimization addresses the problem of reconstructing dynamic scenes from multi-view videos that are not temporally synchronized. Unlike standard dynamic NeRFs that assume all cameras capture the same scene moment at a given frame index, this method introduces learnable per-camera time offsets to align the temporal axes of unsynchronized inputs. It sets itself apart by treating synchronization as a differentiable optimization problem, jointly refining camera-specific time shifts alongside the neural radiance field parameters.

#### NeRF Pose Optimization
- **Definition:** Formulates pose reconstruction as an inverse problem by optimizing the pose instead of network parameters given a pre-trained NeRF [S1].
- **Steps:** Use the photo-metric loss Eq. 6 to optimize the pose [S1]. Use an interest-point detector and perform interest region-based sampling [S1].
- **Context:** iNeRF (December 2020) [166] formulated pose reconstruction as an inverse problem. Given a pre-trained NeRF, K. Gao, et al. using the photo-metric loss Eq. 6, Yen-Chen et al. optimized the pose instead of the network parameters. The authors used an interest-point detector and performed interest region-based sampling. The authors also performed semi-supervised experiments, where they used iNeRF pose estimation on unposed training images to augment the NeRF training set, and further train the forward NeRF. This semi-supervision was shown by the author to reduce the requirement of posed photos from the forward NeRF by 25 % [S1].

#### Camera Pose Optimization and Perturbation
- **Definition:** Two CameraOptModule instances: one for random pose perturbation during training, and one for learnable pose adjustment, both applied per image [S2, unofficial].
- **Steps:** Initialize perturbation module with random noise scaled by `cfg.pose_noise` [S2, unofficial]. Initialize adjustment module with zero parameters to ensure neutral initial pose [S2, unofficial]. During training, apply perturbation to `camtoworlds` if `cfg.pose_noise > 0`: `camtoworlds = self.pose_perturb(camtoworlds, image_ids)` [S2, unofficial]. Apply learnable adjustment to `camtoworlds` if `cfg.pose_opt` is enabled: `camtoworlds = self.pose_adjust(camtoworlds, image_ids)` [S2, unofficial].
- **Parameters:** `cfg.pose_opt_lr`: Base learning rate for pose adjustment; actual LR is `cfg.pose_opt_lr * sqrt(batch_size)` [S2, unofficial]. `cfg.pose_opt_reg`: Weight decay for the pose optimizer [S2, unofficial]. `cfg.pose_noise`: Magnitude of random pose perturbation applied during training [S2, unofficial].

## Background
The problem of joint pose and NeRF optimization emerged as a solution to the dependency on accurate Structure-from-Motion (SfM) pipelines like COLMAP, which can fail on sparse or uncalibrated image sets [G]. Early work, such as iNeRF (December 2020) [166], formulated pose reconstruction as an inverse problem given a pre-trained NeRF, optimizing poses using photo-metric loss and interest-point-based sampling [S1]. Subsequently, NeRF– (February 2021) [101] jointly estimated NeRF model parameters and camera parameters, allowing for end-to-end construction of radiance fields and novel view synthesis, achieving results comparable to using COLMAP with the 2020 NeRF model [S1]. These methods laid the groundwork for more complex joint optimization strategies that handle sparse views, continuous motion, and unsynchronized inputs.

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] llazzaroni/AugSplat, https://github.com/llazzaroni/AugSplat, fetched 2026-10-27 (unofficial)
