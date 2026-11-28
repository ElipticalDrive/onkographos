# HO-Gaussian

Also called: Hybrid Optimization of 3D Gaussian Splatting

## Overview
HO-Gaussian is a point-free representation method for multi-camera urban scenes that combines a grid-based volume with the 3D Gaussian Splatting (3DGS) pipeline to eliminate dependency on Structure-from-Motion (SfM) point initialization [S1]. It solves the problem of rendering low-texture, sky, and distant areas in large-scale urban environments where standard 3DGS fails due to sparse or outlier-prone SfM data [S1]. Unlike standard 3DGS, which relies on well-initialized point clouds, HO-Gaussian learns Gaussian positions directly from a grid volume, uses Gaussian Directional Encoding to replace spherical harmonics for view-dependent color, and applies neural warping to ensure consistency across multiple camera viewpoints [S1].

## How it works
**Inputs:** Multi-view RGB images from a multi-camera setup (e.g., autonomous driving rigs); no SfM point cloud or LiDAR data required [S1].
**Outputs:** A set of 3D Gaussians with optimized positions, opacities, and view-dependent colors; rendered novel views [S1].

**Data Structures:**
1. **Grid-Based Volume:** A 3D voxel grid storing features used for position prediction and directional encoding [S1]. The specific resolution and channel dimensions are not given in the sources [S1].
2. **Gaussian Set:** A collection of $N$ Gaussians [S1]. Each Gaussian $i$ is defined by:
   * Position $\mathbf{p}_i$: Learned from the grid volume, not initialized from points [S1].
   * Opacity $\alpha_i$: Standard scalar [S1].
   * Color $\mathbf{c}_i$: Encoded via Gaussian Directional Encoding (GDE) from the grid volume, replacing spherical harmonics coefficients [S1].

**Numbered Steps:**
1. **Initialization:** Initialize the grid-based volume and the Gaussian set without using SfM points or LiDAR, relying solely on supervised information from images [S1].
2. **Position Learning:** Query the grid-based volume to predict the 3D position $\mathbf{p}_i$ for each Gaussian, facilitating the learning of geometric information in sky, distant, and low-texture areas [S1].
3. **Gaussian Directional Encoding (GDE):** Compute view-dependent color $\mathbf{c}_i$ using the grid volume features instead of storing high-order spherical harmonic functions, which reduces disk space requirements from gigabytes to megabytes [S1].
4. **Neural Warping:** Generate virtual viewpoints and warp features from observed cameras to these viewpoints to enhance consistent appearance and geometry across multiple cameras, reducing overfitting to specific viewpoints [S1].
5. **Hybrid Optimization:** Iteratively optimize the grid-based volume and the Gaussian parameters (positions, opacities, GDE features) using gradient descent on the rendering loss between rendered images and ground truth RGB images [S1].
6. **Point Densification:** At regular intervals during training, add new Gaussian positions to regions with high error, low texture, or distant areas, and fine-tune these positions via gradient descent in the Gaussian pipeline [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ho_gaussian_training(images, camera_poses, num_iterations, densification_interval):
    def initialize_grid_volume(): pass # Not given in sources
    def initialize_gaussians(): pass     # Not given in sources
    def sample_batch(imgs, poses): return imgs[0], poses[0] # Not given in sources
    def query_grid(vol, pos): return pos # Not given in sources
    def query_directional_encoding(vol, pos, cams): return [0.0, 0.0, 0.0] # Not given in sources
    def generate_virtual_views(cams): return cams # Not given in sources
    def warp_features(vol, cams, views): return [0.0] # Not given in sources
    def rasterize_gaussians(gs, cams): return [0.0] # Not given in sources
    def compute_rendering_loss(rendered, gt): return 0.0 # Not given in sources
    def compute_warping_loss(warped, gt): return 0.0 # Not given in sources
    def gradient_descent(vol, gs, loss): return vol, gs # Not given in sources
    def densify_gaussians(gs, vol, errors): return gs # Not given in sources

    grid_volume = initialize_grid_volume()
    gaussians = initialize_gaussians()

    for iteration in range(num_iterations):
        batch_images, batch_cameras = sample_batch(images, camera_poses)
        for g in gaussians:
            g.position = query_grid(grid_volume, g.initial_pos)
        for g in gaussians:
            g.color = query_directional_encoding(grid_volume, g.position, batch_cameras)
        virtual_views = generate_virtual_views(batch_cameras)
        warped_features = warp_features(grid_volume, batch_cameras, virtual_views)
        rendered_images = rasterize_gaussians(gaussians, batch_cameras)
        loss = compute_rendering_loss(rendered_images, batch_images)
        loss += compute_warping_loss(warped_features, batch_images)
        grid_volume, gaussians = gradient_descent(grid_volume, gaussians, loss)
        if iteration % densification_interval == 0:
            gaussians = densify_gaussians(gaussians, grid_volume, rendering_errors)
    return grid_volume, gaussians
```

**Parameters:**
* Densification Interval: Not given in the sources [S1].
* Grid Resolution: Not given in the sources [S1].
* Spherical Harmonics Degree: Replaced by GDE; not given in the sources [S1].

**Edge Cases and Pitfalls:**
* Low-Texture Regions: Standard 3DGS treats these as outliers; HO-Gaussian uses grid volume learning and densification to address this [S1].
* Distant/Sky Areas: SfM points are sparse here; HO-Gaussian learns positions from the grid volume to improve representation [S1].
* Multi-Camera Consistency: Without neural warping, views from different cameras may conflict; warping ensures mutual learning [S1].
* Disk Space: Standard 3DGS with spherical harmonics requires gigabytes for large scenes; HO-Gaussian reduces this to ~123MB for the evaluated scene [S1].

## Subtopics

### Gaussian Directional Encoding
A technique replacing spherical harmonics with grid-based directional encoding to reduce disk space and model view-dependent color [S1]. It sets itself apart by eliminating the need to store high-order coefficients [S1]. Choose it when disk space is a constraint in large-scale scenes [S1]. [Link](9436-textured-gs.md) is a sibling but focuses on textured primitives, not directional encoding.

### Neural Warping
A scheme for multi-camera consistency by warping features to virtual viewpoints [S1]. It sets itself apart by enabling mutual learning between camera perspectives [S1]. Choose it for multi-camera urban scenes [S1]. [Link](10060-deformable-3d-gaussian-splatting.md) is a sibling but focuses on deformation, not warping for consistency.

## Operating characteristics
* Time Complexity: Training time is 76 minutes on the evaluated urban scene (Argoverse dataset) [S1].
* Memory Complexity: Model size is 123MB, significantly smaller than baseline 3DGS (557MB) and NeRF-based methods (e.g., LocalRF 3762MB) [S1].
* Measured Performance: On the Argoverse dataset, HO-Gaussian achieves PSNR 30.98, SSIM 0.9043, LPIPS 0.2287 with RGB input only [S1].
* Rendering Speed: 71 FPS on the evaluated urban scene [S1].
* Scaling: The method is designed for large-scale urban scenes and multi-camera systems [S1]. It scales by using grid-based volume for efficient position and color encoding, avoiding the $O(N)$ storage of spherical harmonics per Gaussian [S1].
* Strengths: Point-free initialization (no SfM/LiDAR needed), reduced disk space, real-time rendering, good performance in low-texture/distant areas, multi-camera consistency [S1].
* Limitations: Relies on grid volume learning [S1].
* When to use: Large-scale urban scene reconstruction from multi-camera RGB data where SfM points are sparse or unreliable [S1].

## Background
HO-Gaussian was proposed by Li et al. in the paper "HO-Gaussian: Hybrid Optimization of 3D Gaussian Splatting for Urban Scenes" [S1]. It addresses limitations of previous 3DGS-based methods in urban scenes, specifically their reliance on SfM points and difficulties in rendering distant, sky, and low-texture areas [S1]. The method was evaluated on the Argoverse and Waymo datasets for autonomous driving scenarios [S1]. It builds on the 3D Gaussian Splatting pipeline [S1] but introduces hybrid optimization with a grid-based volume, Gaussian Directional Encoding, and neural warping [S1].

## Sources
[S1] HO-Gaussian: Hybrid Optimization of 3D Gaussian Splatting for Urban Scenes, https://doi.org/10.1007/978-3-031-73027-6_2, fetched 2026-10-26
