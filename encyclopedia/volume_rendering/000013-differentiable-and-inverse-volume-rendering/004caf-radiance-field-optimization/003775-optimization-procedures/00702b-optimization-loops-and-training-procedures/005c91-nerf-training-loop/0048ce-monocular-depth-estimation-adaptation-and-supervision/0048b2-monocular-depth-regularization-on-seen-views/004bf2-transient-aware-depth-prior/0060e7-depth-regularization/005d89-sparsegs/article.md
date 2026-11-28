# SparseGS

Also called: Sparse View Synthesis Using 3D Gaussian Splatting

## Overview
SparseGS is a method for sparse-view 3D reconstruction using 3D Gaussian Splatting (3DGS) to address misplaced Gaussians ("floaters"), geometry collapse, and texture noise [S1]. It differs from siblings like FSGS, which relies on Gaussian unpooling, and DNGaussian, which uses depth-normal regularizers, by combining monocular depth priors with diffusion-based Score Distillation Sampling (SDS) to stabilize training under extreme sparsity (~3 to 12 views) [S2, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:** Sparse set of input images (typically ~3 to 12 views), camera poses, and intrinsics [S1]. A pre-trained monocular depth estimation model provides initial depth priors [S2, unofficial]. A pre-trained Stable Diffusion model (version 2.1) provides generative priors [S1].
- **Outputs:** A set of 3D Gaussian primitives (position, covariance, opacity, spherical harmonics coefficients) and a rendered image/depth map [G].

**Data Structures:**
- **3D Gaussian Primitives:** Standard 3DGS representation [G]. Each Gaussian $i$ is defined by:
  - Mean position $\mu_i \in \mathbb{R}^3$ [G].
  - Covariance matrix $\Sigma_i \in \mathbb{R}^{3 \times 3}$, parameterized by a rotation matrix $R_i$ and a scaling vector $s_i$ [G].
  - Opacity $\alpha_i \in [0, 1]$ [G].
  - Spherical Harmonics (SH) coefficients for view-dependent color [G].
- **Depth Map:** Rendered depth is computed per pixel [G].

**Numbered Steps:**

1. **Initialization:** Initialize Gaussians from the input point cloud or sparse reconstruction [G]. Standard 3DGS initialization is assumed.
2. **Softmax-Scaling Depth Rendering:** To reduce misplaced Gaussians caused by alpha-blending depth averaging, replace standard alpha-blending depth with a softmax-scaled depth rendering [S1]. This constrains depth convergence [S1].
   - Let $d_i$ be the depth of the $i$-th Gaussian along the ray.
   - Let $\alpha_i$ be the opacity.
   - The rendered depth $D$ is computed using a softmax-weighted average rather than a linear alpha-composite average, biasing towards the mode of the depth distribution [S1].
3. **Depth Correlation Loss:** Apply a Pearson Correlation Loss between the rendered depth and the monocular depth estimate [S1].
   - Compute local and global Pearson correlation coefficients between the rendered depth map and the reference monocular depth map [S1].
   - Loss weights: Local Pearson loss weight = 0.15; Global Pearson loss weight = 0.15 [S1].
4. **Unseen Viewpoints Regularization (UVR):** To prevent overfitting and reduce high-frequency artifacts in sparse settings, render images from viewpoints away from the training set and apply a consistency loss [S1]. This improves PSNR by an average of 0.33 dB [S1].
5. **Score Distillation Sampling (SDS) and Depth Warping:** For regions with insufficient input coverage, leverage SDS and depth warping to reduce geometry collapse and texture noise while preserving fine details [S1].
   - Enable SDS after two-thirds of the total training iterations, once the scene structure is stable [S1].
   - SDS Loss weight: $5 \times 10^{-4}$ [S1].
   - Depth Warping Loss weight: 0.05 [S1].
6. **Floater Pruning:** Identify and remove low-opacity Gaussians that are misplaced (floaters) [S1]. This heuristic cleans the scene, making the image and depth map sharper [S1]. This boosts PSNR by 0.28 dB [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_sparsegs(input_images, camera_poses, num_iterations):
    # Standard 3DGS initialization from sparse point cloud [G]
    gaussians = {
        'positions': torch.tensor(initialize_point_cloud(input_images, camera_poses)),
        'covariances': compute_covariances_from_point_cloud(),
        'opacities': torch.ones(num_gaussians),
        'sh_coeffs': torch.zeros(num_gaussians, num_sh_bases, 3)
    }

    # Pre-trained models [G]
    depth_estimator = MonocularDepthModel()
    diffusion_model = StableDiffusionV2_1()

    # Hyperparameters [S1]
    w_pearson_local = 0.15
    w_pearson_global = 0.15
    w_sds = 5e-4
    w_depth_warp = 0.05
    sds_start_iter = int(num_iterations * 2 / 3)

    for iter in range(num_iterations):
        # 1. Forward Pass: Render Image and Depth [G]
        rendered_image, rendered_depth = render_gaussians(
            gaussians, camera_poses, use_softmax_depth=True
        )

        # 2. Compute Losses
        loss = torch.tensor(0.0, requires_grad=True)

        # Depth Correlation Loss (Pearson) [S1]
        ref_depth = depth_estimator(input_images[0])
        local_corr = pearson_correlation(rendered_depth, ref_depth, window_size=7)
        global_corr = pearson_correlation(rendered_depth, ref_depth)
        loss = loss + w_pearson_local * (1.0 - local_corr)
        loss = loss + w_pearson_global * (1.0 - global_corr)

        # Unseen Viewpoints Regularization (UVR) [S1]
        unseen_pose = sample_random_pose(camera_poses)
        rendered_unseen, _ = render_gaussians(gaussians, unseen_pose, use_softmax_depth=True)
        loss = loss + 0.1 * torch.mean(torch.abs(rendered_unseen - input_images[0]))

        # SDS and Depth Warping [S1]
        if iter > sds_start_iter:
            # Score Distillation Sampling [G]
            noise = torch.randn_like(rendered_image)
            t = torch.randint(0, 1000, (1,)).item()
            noisy_image = diffusion_model.add_noise(rendered_image, noise, t)
            noise_pred = diffusion_model.predict_noise(noisy_image, t)
            sds_grad = compute_sds_gradient(rendered_image, noise_pred, noise)
            loss = loss + w_sds * sds_grad

            # Depth Warping [G]
            warped_depth = warp_depth(rendered_depth, camera_poses, unseen_pose)
            loss = loss + w_depth_warp * torch.mean(torch.abs(rendered_depth - warped_depth))

        # 3. Backpropagate and Update [G]
        gradients = torch.autograd.grad(loss, list(gaussians.values()))
        update_gaussians(gaussians, gradients)

        # 4. Floater Pruning [S1]
        gaussians = prune_floaters(gaussians, opacity_threshold=0.01)

    return gaussians

# Helper functions [G]
def initialize_point_cloud(images, poses):
    # Standard 3DGS initialization from sparse point cloud
    return standard_3dgs_init(images, poses)

def compute_covariances_from_point_cloud():
    # Standard covariance computation from point cloud
    return standard_covariance_calc()

def render_gaussians(gaussians, poses, use_softmax_depth):
    # Standard 3DGS rasterization; if use_softmax_depth, depth is weighted by softmax(alpha) instead of linear alpha
    return standard_rasterize(gaussians, poses, use_softmax_depth)

def pearson_correlation(x, y, window_size=None):
    # Standard Pearson correlation coefficient computation
    return standard_pearson_corr(x, y, window_size)

def sample_random_pose(poses):
    # Randomly sample a pose outside the training set
    return random.choice(poses)

def compute_sds_gradient(rendered, noise_pred, noise):
    # Standard SDS gradient: w(t) * (noise_pred - noise) * d(rendered)/d(rendered)
    return (noise_pred - noise)

def warp_depth(depth, poses, target_pose):
    # Standard depth warping using camera poses and inverse projection
    return standard_depth_warp(depth, poses, target_pose)

def prune_floaters(gaussians, opacity_threshold):
    # Filter Gaussians with opacity below threshold
    mask = gaussians['opacities'] > opacity_threshold
    return {k: v[mask] for k, v in gaussians.items()}

def update_gaussians(gaussians, gradients):
    # Standard Adam optimizer step
    standard_optimizer_step(gaussians, gradients)
```

**Parameters:**
- SDS loss weight: $5 \times 10^{-4}$ [S1].
- Depth warping loss weight: 0.05 [S1].
- Local and Global Pearson loss weights: 0.15 [S1].
- SDS activation: After 2/3 of total iterations [S1].

**Edge Cases and Pitfalls:**
- **Precision:** Softmax-scaling depth rendering is critical to avoid the averaging artifacts of standard alpha-blending which cause floaters [S1].
- **Boundaries:** The method is designed for forward-facing datasets (LLFF, DTU) and unbounded scenes (Mip-NeRF360) [S1].
- **Degenerate Input:** In regions with very little coverage, standard 3DGS fails; SDS and depth warping are specifically enabled to handle this [S1].

## Subtopics

### FSGS Training and Densification
Uses Gaussian unpooling for few-shot reconstruction [S2, unofficial]. Excels in fine details but fails on background geometry [S1]. [Link](6420-fsgs-training-and-densification.md)

### DNGaussian Training and Depth Regularization
Uses depth-normal regularizers [S2, unofficial]. Renders coherent backgrounds but may prune foreground objects incorrectly [S1]. [Link](7399-dngaussian.md)

### Edge-Aware Disparity Regularization
Focuses on edge-aware constraints for disparity [S2, unofficial]. [Link](9140-edge-aware-disparity-regularization.md)

## Operating characteristics
- **Time and Memory Complexity:** Standard 3DGS complexity, plus overhead for monocular depth estimation and SDS gradient computation [G]. SDS is enabled only after 2/3 of iterations, reducing early training cost [S1].
- **Measured Performance:** On Mip-NeRF360 (12-view setting):
  - Baseline (Alpha-blending): PSNR 17.49, LPIPS 0.431, SSIM 0.499.
  - + Softmax-scaling: PSNR 18.45.
  - + UVR: PSNR 18.86.
  - + Pruning: PSNR 19.09.
  - Full SparseGS: PSNR 19.37, LPIPS 0.398, SSIM 0.577 [S1].
- **Scaling:** Outperforms NeRF-based methods and concurrent works (FSGS, DNGaussian) in both 12-view and 24-view settings [S1].
- **Strengths:** Effectively reconstructs high-frequency geometry and preserves sharp boundaries. Cleaner and more coherent scene structure than DNGaussian. Less background collapse than FSGS [S1].
- **Limitations:** Relies on pre-trained monocular depth and diffusion models, which may introduce biases if the priors are inaccurate. SDS computation is expensive [G].

## Background
SparseGS was proposed to tackle sparse-view 3D reconstruction using 3D Gaussian Splatting [S1]. It addresses the limitations of original 3DGS, where alpha-blending depth rendering leads to misplaced Gaussians (floaters), and the tendency of 3DGS to place isolated Gaussians in under-covered regions, resulting in high-frequency noise [S1]. The method builds on the observation that NeRF-based methods often produce overly smooth appearances in sparse settings, while 3DGS-based representations gravitate towards isolated, noisy Gaussians [S1]. It utilizes Stable Diffusion v2.1 for generative priors [S1].

## Sources
[S1] Sparsegs: Sparse View Synthesis Using 3D Gaussian Splatting, https://doi.org/10.1109/3dv66043.2025.00100, fetched 2026-10-26
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
