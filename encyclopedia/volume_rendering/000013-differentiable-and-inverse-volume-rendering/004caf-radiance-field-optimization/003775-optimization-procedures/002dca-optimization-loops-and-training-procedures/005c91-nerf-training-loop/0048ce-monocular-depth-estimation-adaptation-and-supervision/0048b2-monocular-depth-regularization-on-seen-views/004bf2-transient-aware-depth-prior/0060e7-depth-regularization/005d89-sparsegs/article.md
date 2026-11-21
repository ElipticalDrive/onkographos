# SparseGS

Also called: Sparse View Synthesis Using 3D Gaussian Splatting

## Overview
SparseGS is a method for sparse-view 3D reconstruction using 3D Gaussian Splatting (3DGS) to address issues such as misplaced Gaussians ("floaters"), geometry collapse, and texture noise [S1]. It differs from sibling methods like FSGS, which relies on Gaussian unpooling, and DNGaussian, which uses depth-normal regularizers, by combining monocular depth priors with diffusion-based Score Distillation Sampling (SDS) to stabilize training under extreme sparsity (~3 to 12 views) [S2, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:** Sparse set of input images (typically ~3 to 12 views), camera poses, and intrinsics. A pre-trained monocular depth estimation model provides initial depth priors. A pre-trained Stable Diffusion model (version 2.1) provides generative priors [S1].
- **Outputs:** A set of 3D Gaussian primitives (position, covariance, opacity, spherical harmonics coefficients) and a rendered image/depth map.

**Data Structures:**
- **3D Gaussian Primitives:** Standard 3DGS representation. Each Gaussian $i$ is defined by:
  - Mean position $\mu_i \in \mathbb{R}^3$.
  - Covariance matrix $\Sigma_i \in \mathbb{R}^{3 \times 3}$, parameterized by a rotation matrix $R_i$ and a scaling vector $s_i$.
  - Opacity $\alpha_i \in [0, 1]$.
  - Spherical Harmonics (SH) coefficients for view-dependent color.
- **Depth Map:** Rendered depth is computed per pixel.

**Numbered Steps:**

1. **Initialization:** Initialize Gaussians from the input point cloud or sparse reconstruction. [G] Standard 3DGS initialization is assumed.
2. **Softmax-Scaling Depth Rendering:** To reduce misplaced Gaussians caused by alpha-blending depth averaging, replace standard alpha-blending depth with a softmax-scaled depth rendering. This constrains depth convergence [S1].
   - Let $d_i$ be the depth of the $i$-th Gaussian along the ray.
   - Let $\alpha_i$ be the opacity.
   - The rendered depth $D$ is computed using a softmax-weighted average rather than a linear alpha-composite average, biasing towards the mode of the depth distribution [S1].
3. **Depth Correlation Loss:** Apply a Pearson Correlation Loss between the rendered depth and the monocular depth estimate.
   - Compute local and global Pearson correlation coefficients between the rendered depth map and the reference monocular depth map.
   - Loss weights: Local Pearson loss weight = 0.15; Global Pearson loss weight = 0.15 [S1].
4. **Unseen Viewpoints Regularization (UVR):** To prevent overfitting and reduce high-frequency artifacts in sparse settings, render images from viewpoints away from the training set and apply a consistency loss. This improves PSNR by an average of 0.33 dB [S1].
5. **Score Distillation Sampling (SDS) and Depth Warping:** For regions with insufficient input coverage, leverage SDS and depth warping to reduce geometry collapse and texture noise while preserving fine details.
   - Enable SDS after two-thirds of the total training iterations, once the scene structure is stable [S1].
   - SDS Loss weight: $5 \times 10^{-4}$ [S1].
   - Depth Warping Loss weight: 0.05 [S1].
6. **Floater Pruning:** Identify and remove low-opacity Gaussians that are misplaced (floaters). This heuristic cleans the scene, making the image and depth map sharper. This boosts PSNR by 0.28 dB [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_sparsegs(input_images, camera_poses, num_iterations):
    # Initialize Gaussians
    gaussians = initialize_gaussians(input_images, camera_poses)
    
    # Load pre-trained models
    depth_estimator = load_monocular_depth_model()
    diffusion_model = load_stable_diffusion_v2_1()
    
    # Hyperparameters
    weight_depth_pearson_local = 0.15
    weight_depth_pearson_global = 0.15
    weight_sds = 5e-4
    weight_depth_warping = 0.05
    sds_start_iter = int(num_iterations * 2/3)
    
    for iter in range(num_iterations):
        # 1. Forward Pass: Render Image and Depth
        # Use softmax-scaling for depth to reduce floaters
        rendered_image, rendered_depth = render_gaussians(
            gaussians, camera_poses, use_softmax_depth=True
        )
        
        # 2. Compute Losses
        loss = 0.0
        
        # Image Reconstruction Loss (e.g., L1 or LPIPS)
        # loss += image_loss(rendered_image, input_image)
        
        # Depth Correlation Loss (Pearson)
        ref_depth = depth_estimator(input_image)
        local_corr = pearson_correlation(rendered_depth, ref_depth, local_window=True)
        global_corr = pearson_correlation(rendered_depth, ref_depth, local_window=False)
        loss += weight_depth_pearson_local * (1.0 - local_corr)
        loss += weight_depth_pearson_global * (1.0 - global_corr)
        
        # Unseen Viewpoints Regularization (UVR)
        # Render from random unseen viewpoints and compare
        unseen_view = sample_unseen_view(camera_poses)
        rendered_unseen = render_gaussians(gaussians, unseen_view, use_softmax_depth=True)
        # loss += unseen_view_loss(rendered_unseen)
        
        # SDS and Depth Warping (after 2/3 iterations)
        if iter > sds_start_iter:
            # Score Distillation Sampling
            # Sample noise, predict clean image from diffusion model
            # Compute gradient w.r.t. rendered image
            sds_grad = compute_sds_gradient(rendered_image, diffusion_model)
            loss += weight_sds * sds_grad
            
            # Depth Warping
            # Warp reference depth from input view to current view
            warped_depth = warp_depth(ref_depth, camera_poses)
            depth_warp_loss = depth_consistency_loss(rendered_depth, warped_depth)
            loss += weight_depth_warping * depth_warp_loss
            
        # 3. Backpropagate and Update
        gradients = compute_gradients(loss, gaussians)
        update_gaussians(gaussians, gradients)
        
        # 4. Floater Pruning (Periodic)
        # Remove low-opacity Gaussians
        gaussians = prune_floaters(gaussians, threshold=0.01)
        
    return gaussians
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
Uses Gaussian unpooling for few-shot reconstruction. Excels in fine details but fails on background geometry. [Link](6420-fsgs-training-and-densification.md)

### DNGaussian Training and Depth Regularization
Uses depth-normal regularizers. Renders coherent backgrounds but may prune foreground objects incorrectly. [Link](7399-dngaussian.md)

### Edge-Aware Disparity Regularization
Focuses on edge-aware constraints for disparity. [Link](9140-edge-aware-disparity-regularization.md)

## Operating characteristics
- **Time and Memory Complexity:** Standard 3DGS complexity, plus overhead for monocular depth estimation and SDS gradient computation. SDS is enabled only after 2/3 of iterations, reducing early training cost [S1].
- **Measured Performance:** On Mip-NeRF360 (12-view setting):
  - Baseline (Alpha-blending): PSNR 17.49, LPIPS 0.431, SSIM 0.499.
  - + Softmax-scaling: PSNR 18.45.
  - + UVR: PSNR 18.86.
  - + Pruning: PSNR 19.09.
  - Full SparseGS: PSNR 19.37, LPIPS 0.398, SSIM 0.577 [S1].
- **Scaling:** Outperforms NeRF-based methods and concurrent works (FSGS, DNGaussian) in both 12-view and 24-view settings [S1].
- **Strengths:** Effectively reconstructs high-frequency geometry and preserves sharp boundaries. Cleaner and more coherent scene structure than DNGaussian. Less background collapse than FSGS [S1].
- **Limitations:** Relies on pre-trained monocular depth and diffusion models, which may introduce biases if the priors are inaccurate. SDS computation is expensive.

## Background
SparseGS was proposed to tackle sparse-view 3D reconstruction using 3D Gaussian Splatting. It addresses the limitations of original 3DGS, where alpha-blending depth rendering leads to misplaced Gaussians (floaters), and the tendency of 3DGS to place isolated Gaussians in under-covered regions, resulting in high-frequency noise [S1]. The method builds on the observation that NeRF-based methods often produce overly smooth appearances in sparse settings, while 3DGS-based representations gravitate towards isolated, noisy Gaussians [S1]. It utilizes Stable Diffusion v2.1 for generative priors [S1].

## Sources
[S1] Sparsegs: Sparse View Synthesis Using 3D Gaussian Splatting, https://doi.org/10.1109/3dv66043.2025.00100, fetched 2026-10-26
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
