# 3D Gaussian Splatting Optimization

Also called: 3DGS Training Procedure

## Overview
This article details the specific optimization schedule, initialization strategies, and hyperparameter configurations required to train a 3D Gaussian Splatting model from scratch. It addresses the problem of converging anisotropic Gaussian primitives to represent a scene's geometry and appearance while maintaining training stability and rendering speed. It differs from general differentiable rendering by employing a staged introduction of Spherical Harmonics (SH) and a resolution warm-up to mitigate artifacts caused by missing angular information and gradient instability [S1, unofficial].

## How it works

### Inputs and Outputs
**Inputs:**
*   A set of $M$ input images with known camera poses (intrinsics and extrinsics) [S1, unofficial].
*   Scene bounds (for synthetic bounded scenes) or a bounding volume [S1, unofficial].

**Outputs:**
*   A set of $N$ optimized 3D Gaussians $\mathcal{G} = \{g_i\}_{i=1}^N$, where each $g_i$ contains:
    *   Position $\mathbf{p}_i \in \mathbb{R}^3$
    *   Covariance $\Sigma_i$ (parameterized by rotation quaternion $\mathbf{q}_i \in \mathbb{R}^4$ and scale vector $\mathbf{s}_i \in \mathbb{R}^3$)
    *   Opacity $\alpha_i \in [0, 1]$
    *   Spherical Harmonics coefficients $\mathbf{c}_{i, l}$ for view-dependent color [S1, unofficial].

### Data Structures
The implementation uses Python/PyTorch with custom CUDA kernels for rasterization [S1, unofficial].
*   **Gaussian Attributes:** Stored in contiguous arrays or PyTorch tensors for batched operations.
*   **Sorting:** Uses NVIDIA CUB sorting routines for fast Radix sort to order Gaussians by depth for alpha blending [S1, unofficial].
*   **Rasterization:** Custom CUDA kernels extended from Kopanas et al. [2021] [S1, unofficial].

### Optimization Steps

1.  **Initialization:**
    *   **Synthetic Bounded Scenes:** Initialize with 100,000 Gaussians uniformly distributed within a volume enclosing the scene bounds [S1, unofficial]. These are automatically pruned to 6,000–10,000 meaningful Gaussians during early training [S1, unofficial].
    *   **Real-World/Unbounded Scenes:** Typically initialized from Structure-from-Motion (SfM) points, though the specific initialization count for real-world scenes is not explicitly detailed in the provided notes beyond the synthetic case [S1, unofficial].

2.  **Resolution Warm-up:**
    *   Start optimization at an image resolution 4 times smaller than the target resolution [S1, unofficial].
    *   Upsample the resolution twice: first after 250 iterations, and second after 500 iterations [S1, unofficial].

3.  **Spherical Harmonics (SH) Band Introduction:**
    *   To prevent incorrect diffuse color values due to missing angular regions in captures, optimize **only** the zero-order (DC) SH component initially [S1, unofficial].
    *   Introduce one additional SH band every 1,000 iterations [S1, unofficial].
    *   Continue until all 4 bands of SH are represented [S1, unofficial].

4.  **Gradient Computation:**
    *   Compute gradients for **all** visible Gaussians contributing to the pixel color.
    *   **Pitfall:** Limiting gradient computation to only the front-most $N$ points (e.g., $N=10$) causes unstable optimization and severe quality degradation (e.g., an 11dB PSNR drop in the Truck scene) [S1, unofficial].

5.  **Densification (Implicit in Optimization Loop):**
    *   **Cloning:** Small Gaussians are cloned to better/faster converge on thin structures [S1, unofficial].
    *   **Splitting:** Large Gaussians are split to improve background reconstruction [S1, unofficial].
    *   *Note: The specific thresholds and frequency for cloning/splitting are part of the broader "Gaussian Primitive Training Schedule" [18977-gaussian-primitive-training-schedule.md] and are not detailed in the optimization-specific notes.*

6.  **Pruning:**
    *   Gaussians with low opacity are pruned during the training process to maintain compactness [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_3d_gaussian_splatting(input_images, camera_poses, target_resolution, total_iterations=30000):
    """
    Optimizes 3D Gaussian Splatting parameters.
    """
    # 1. Initialization
    # For synthetic bounded scenes: 100K random points in bounding volume
    # For real-world: Typically SfM initialization (details in parent article)
    gaussians = initialize_gaussians(scene_bounds="bounded") 
    
    # 2. Setup Optimizer
    # Adam optimizer with beta1=0.9, beta2=0.99 (from Shared Principles)
    optimizer = Adam(gaussians.parameters(), lr=initial_lr)
    
    # 3. Setup Rendering Pipeline
    # Custom CUDA rasterization extended from Kopanas et al. [2021]
    # Radix sort via NVIDIA CUB
    rasterizer = CustomCUDARasterizer()
    
    # 4. Training Loop
    current_sh_band = 0 # Start with 0-order only
    current_resolution = target_resolution / 4 # 4x smaller
    
    for iteration in range(1, total_iterations + 1):
        
        # Resolution Warm-up
        if iteration == 250:
            current_resolution = current_resolution * 2
        elif iteration == 500:
            current_resolution = current_resolution * 2
            
        # Sample a batch of images and cameras
        image, camera = sample_batch(input_images, camera_poses)
        
        # Render image using current resolution
        # All visible Gaussians contribute to gradients (do NOT limit to N front-most)
        rendered_image = rasterizer.render(gaussians, camera, current_resolution)
        
        # Compute Loss
        # Composite loss: (1-lambda)*L1 + lambda*D-SSIM (from Shared Principles)
        loss = compute_composite_loss(rendered_image, image)
        
        # Backpropagation
        loss.backward()
        
        # Update SH Bands
        # Introduce one band every 1000 iterations
        if iteration % 1000 == 0 and current_sh_band < 4:
            current_sh_band += 1
            enable_sh_band(gaussians, current_sh_band)
            
        # Step Optimizer
        optimizer.step()
        optimizer.zero_grad()
        
        # Densification and Pruning
        # (Specific logic delegated to Gaussian Primitive Training Schedule)
        densify_and_prune(gaussians, iteration)
        
    return gaussians
```

### Parameters and Typical Values
*   **Total Iterations:** 30,000 for standard scenes .
*   **SH Band Interval:** 1,000 iterations per band [S1, unofficial].
*   **Total SH Bands:** 4 [S1, unofficial].
*   **Initial Resolution Factor:** 4x smaller than target [S1, unofficial].
*   **Resolution Upsample Steps:** At iteration 250 and 500 [S1, unofficial].
*   **Gradient Depth Limit:** None (all visible points) [S1, unofficial].

### Edge Cases and Pitfalls
*   **Missing Angular Information:** In captures with missing regions (e.g., corners, inside-out), optimizing all SH bands simultaneously produces incorrect zero-order (diffuse) color values. Mitigated by optimizing only zero-order initially [S1, unofficial].
*   **Gradient Limiting:** Limiting gradients to the front-most $N$ points (e.g., $N=10$) leads to unstable optimization and significant quality loss (11dB PSNR drop in Truck scene) [S1, unofficial].
*   **Popping Artifacts:** Large Gaussians can cause popping artifacts, partly due to trivial rejection via guard bands in the rasterizer and depth order switches [S1, unofficial].

## Operating characteristics

### Time and Memory Complexity
*   **Training Time:**
    *   Synthetic NeRF scenes: ~35-45 minutes on a single A6000 GPU [S1, unofficial].
    *   Mip-NeRF360 scenes: 48 hours on a single GPU (equivalent to 12 hours on a 4-GPU A100 node) [S1, unofficial].
    *   Tanks & Temples: ~7 minutes on a single A6000 GPU [S1, unofficial].
*   **Rendering Speed:**
    *   Trained synthetic scenes: 180-300 FPS on A6000 GPU [S1, unofficial].
    *   Mip-NeRF360: 10 seconds per frame [S1, unofficial].
*   **Model Size:**
    *   Synthetic scenes: 200,000–500,000 Gaussians per scene after 30,000 iterations [S1, unofficial].
    *   Compactness: Uses ~1/4 the point count of Zhang et al. [2022] with 3.8 MB model size vs 9 MB [S1, unofficial].

### Strengths and Limitations
*   **Strengths:**
    *   Anisotropic covariance significantly improves surface alignment and rendering quality compared to isotropic models [S1, unofficial].
    *   Achieves comparable quality to InstantNGP/Plenoxels in 5-10 minutes, surpassing them with additional time [S1, unofficial].
    *   Fast convergence for thin structures via cloning small Gaussians [S1, unofficial].
*   **Limitations:**
    *   Artifacts in poorly observed regions (e.g., elongated "splotchy" Gaussians) [S1, unofficial].
    *   Popping artifacts in regions with view-dependent appearance [S1, unofficial].
    *   Training time for large unbounded scenes (e.g., Mip-NeRF360) is significantly longer than for synthetic bounded scenes [S1, unofficial].

### When to Use
*   Use when real-time rendering speed is required after training.
*   Use when high-quality view-dependent effects are needed (SH bands).
*   Avoid if training time is strictly constrained for very large unbounded scenes without access to multi-GPU setups.

## Background

### Origin
The method was introduced in the paper "3D Gaussian Splatting for Real-Time Radiance Field Rendering" [S1, unofficial]. The rasterization kernels are extended versions of methods by Kopanas et al. [2021], and the sorting uses routines from Merrill and Grimshaw [2010] [S1, unofficial].

### History and Development
The optimization procedure was refined to address specific issues in Gaussian Splatting, such as the instability caused by limited gradient depth and the sensitivity of SH coefficients to missing angular data. The staged introduction of SH bands and resolution warm-up were key innovations to stabilize training [S1, unofficial].

### Usage
Evaluated on 13 real scenes (Mip-NeRF360, Tanks&Temples, Hedman et al.) and synthetic Blender dataset [S1, unofficial]. Hardware used: A6000 GPU for most results, 4-GPU A100 node for Mip-NeRF360 [S1, unofficial].

## Sources

[S1] 3D Gaussian Splatting for Real-Time Radiance Field Rendering [pdf], https://repo-sam.inria.fr/fungraph/3d-gaussian-splatting/3d_gaussian_splatting_low.pdf, fetched 2026-10-26 (unofficial)
