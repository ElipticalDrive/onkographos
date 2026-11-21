# SfMNeRF

Also called: SfM-NeRF

## Overview
SfMNeRF is a method for novel view synthesis and 3D-scene geometry reconstruction using Neural Radiance Fields (NeRF) under sparse input conditions [S1]. It improves geometric accuracy and avoids local minima by integrating Structure-from-Motion (SfM) constraints—specifically matched feature positions, epipolar geometry, and multi-view photometric consistency—into the view synthesis training loop [S1]. Unlike standard NeRF, which relies solely on implicit photometric consistency, SfMNeRF explicitly constrains the 3D-scene structure using sparse feature matches and epipolar lines, while employing sub-pixel rendering to increase sample density and generalization [S1].

## How it works
SfMNeRF builds on [Specific Model Architectures](8585-specific-model-architectures.md) by augmenting the standard NeRF optimization with explicit geometric losses derived from sparse features and multi-view geometry.

### Inputs and Outputs
**Inputs:**
*   A set of $m$ training images with known camera intrinsics $K$ and global poses $[\mathbf{r}_i, \mathbf{t}_i]$.
*   Matched Scale-Invariant Feature Transform (SIFT) correspondences between image pairs.
*   A pyramidal representation of the input images for coarse-to-fine optimization.

**Outputs:**
*   A trained NeRF model (MLP parameters $\theta$) representing volume density $\sigma_\theta$ and color $c_\theta$.
*   Synthesized novel views and depth maps.

### Data Structures
*   **Camera Intrinsic Matrix ($K$):** $3 \times 3$ matrix mapping 3D world coordinates to 2D image coordinates.
*   **Relative Pose ($T_{r \to t}$):** $4 \times 4$ transformation matrix from reference image $r$ to target image $t$, composed of rotation $\mathbf{r}$ and translation $\mathbf{t}$.
*   **SIFT Features:** A set of 2D pixel coordinates in reference and target images, matched across views.
*   **Mask Map ($M$):** A binary mask where $M(p(k))=1$ if point $p(k)$ lies within the minimum bounding rectangle of all matched SIFT features in the reference image, else $0$.
*   **Depth Map ($D$):** Estimated depth for each pixel, computed via volume rendering integration.

### Numbered Steps

1.  **Pyramidal Construction:** Construct an $L$-level pyramid for the reference image $I_r$ and target image $I_t$. Let $I^0_r = I_r$ and $I^0_t = I_t$. For level $n > 0$, $I^n_r$ and $I^n_t$ are downsampled versions of $I^{n-1}_r$ and $I^{n-1}_t$ with half the resolution.
2.  **Initialization:** Start optimization at the lowest pyramid level $L-1$.
3.  **Sub-pixel Sampling:** For a selected patch $\Omega_p$ in the reference image, generate sub-pixel coordinates $p = (x + x_{off}, y + y_{off})$ where $x_{off}, y_{off} \in (0, 1)$.
4.  **Ground Truth Interpolation:** Compute the ground-truth color $\hat{I}(p)$ for each sub-pixel using bilinear interpolation of the four nearest integer-pixel neighbors in the reference image:
    $$ \hat{I}(p) = \sum_{i \in \{top, bottom\}, j \in \{left, right\}} w^{ij} I(p^{ij}) $$
    where $w^{ij}$ is proportional to the inverse Euclidean distance between $p$ and neighbor $p^{ij}$.
5.  **Volume Rendering:** Input sub-pixel coordinates and viewing directions into the NeRF to render color $\tilde{I}(p)$ and depth $D(p)$.
6.  **Feature Extraction:** Extract SIFT features from the reference and target images and obtain matched correspondences.
7.  **3D Coordinate Estimation:** For each matched SIFT feature $k$, compute its 3D coordinate $\mathbf{x}_s^k$ in the reference and target views using weighted volume rendering integration:
    $$ \mathbf{x}_s = \sum_{j=1}^{N_c} w_j (\mathbf{o} + t_j \mathbf{d}) $$
    where $w_j = T_j (1 - \exp(-\sigma_j \delta_j))$, $\delta_j = t_{j-1} - t_j$, and $N_c$ is the number of samples along the ray.
8.  **Loss Computation:** Calculate the following loss terms:
    *   **Rendering Loss ($L_{ren}$):** Mean squared error between rendered sub-pixel color and interpolated ground truth.
    *   **Position-of-Matches Loss ($L_{3D}$):** Squared Euclidean distance between 3D coordinates of matched features in reference and target views.
    *   **Epipolar Loss ($L_{epi}$):** Minimum squared distance between the 3D coordinate of a reference point and points on the epipolar line in the target view.
    *   **Photometric Consistency Loss ($L_{pr}$):** L1 difference between the reference patch and the warped target patch, masked by $M$.
    *   **SSIM Loss ($L_{SSIM}$):** Structural Similarity index between reference and warped target patches, masked by $M$.
    *   **Depth Smoothness Loss ($L_{ds}$):** Weighted L1 norm of depth gradients, where weights are determined by image intensity.
    *   **Depth Prior Loss ($L_{depth}$):** (Used in levels $n < L-1$) L1 difference between current depth and bilinearly interpolated depth from the previous lower-resolution level.
9.  **Optimization:** Minimize the total loss $L_{total}$ using the Adam optimizer.
10. **Pyramidal Progression:** Recursively move to level $n-2$, using the depth map from level $n-1$ as a prior via $L_{depth}$, until the "zeroth" level is reached.

### Formulas

**Rendering Loss:**
$$ L_{ren} = \frac{1}{m} \sum_{p_i \in \Omega_p} \left\| \hat{I}(p_i) - \tilde{I}(p_i) \right\|_2^2 $$
where $\hat{I}(p_i)$ is the ground-truth color from interpolation and $\tilde{I}(p_i)$ is the synthesized color from NeRF.

**Position-of-Matched Features Loss:**
$$ L_{3D} = \frac{1}{n} \sum_{k=1}^n \|\mathbf{x}_s^k - \mathbf{x}_{s'}^k\|_2^2 $$
where $\mathbf{x}_s^k$ and $\mathbf{x}_{s'}^k$ are the estimated 3D coordinates of matched SIFT feature $k$ in the reference and target images, respectively.

**Epipolar Constraint Loss:**
$$ L_{epi} = \min_t \left\{ \left\| \mathbf{d}_{p_r} - \mathbf{d}_{p_t} \right\|_2^2 \right\} $$
where $\mathbf{d}_{p_r}$ is the 3D coordinate of the point in the reference image, and $\mathbf{d}_{p_t}$ are the 3D coordinates of points on the epipolar line in the target image. Implausible points on the epipolar line are filtered by color similarity threshold.

**Warped Pixel Coordinate:**
$$ \hat{p}_t = K T_{r \to t} \mathbf{x}_p $$
where $\mathbf{x}_p$ is the 3D coordinate of pixel $p$ in the reference image, $K$ is the intrinsic matrix, and $T_{r \to t} = [\mathbf{r}, \mathbf{t}]$ is the relative pose.

**Relative Pose Definition:**
$$ \mathbf{r} = \mathbf{r}_t^{-1}, \quad \mathbf{t} = -\mathbf{r}_t^{-1}\mathbf{t}_t $$
where $[\mathbf{r}_t, \mathbf{t}_t]$ is the global pose of the target image.

**Photometric Reconstruction Loss:**
$$ L_{pr} = \sum_{p(k) \in \Omega_p} M(p(k)) \left| \hat{I}_r(p(k)) - \hat{I}_t(\hat{p}_t(k)) \right| $$
where $\hat{I}_r$ is the reference patch color, $\hat{I}_t(\hat{p}_t(k))$ is the interpolated color of the warped point in the target image, and $M$ is the binary mask.

**SSIM Loss:**
$$ L_{SSIM} = M \left( \frac{1 - SSIM_{r \to t}}{2} \right) $$
where $SSIM_{r \to t}$ is the structural similarity between the reference and target patches.

**Depth Smooth Loss:**
$$ L_{ds} = \sum_{p(k) \in \Omega_p} \left| \nabla D ( p(k) ) \cdot \left( e^{-|\hat{I}(p(k))|} \right) \right| $$
where $\nabla$ is the 2D differential operator and $D(p(k))$ is the estimated depth.

**Depth Prior Loss (Pyramidal):**
$$ L_{depth} = \sum_{p(k) \in \Omega_p} |D(p(k)) - D_r^{L-1}(p(k)/2)| $$
where $D_r^{L-1}(p(k)/2)$ is the depth value sampled via bilinear interpolation from the lower-resolution depth map $D_r^{L-1}$.

**Total Loss:**
$$ L_{total} = \lambda_{ren} L_{ren} + \lambda_{3D} L_{3D} + \lambda_{pr} L_{pr} + \lambda_{epi} L_{epi} + \lambda_{SSIM} L_{SSIM} + \lambda_{ds} L_{ds} + \lambda_{depth} L_{depth} $$

### Parameters
*   **Loss Weights:** $\lambda_{ren} = 1.0$, $\lambda_{3D} = 1.0$, $\lambda_{pr} = 0.001$, $\lambda_{epi} = 0.0001$, $\lambda_{SSIM} = 0.008$, $\lambda_{ds} = 0.01$. [S1]
*   **Patch Size:** $32 \times 32$ pixels. [S1]
*   **Optimizer:** Adam with $\beta_1 = 0.9$, $\beta_2 = 0.999$, learning rate $1e-3$. [S1]
*   **Sampling:** Hierarchical sampling with 64 coarse samples and 64 importance samples. [S1]

### Edge Cases and Pitfalls
*   **Epipolar Point Filtering:** The large number of points on an epipolar line causes computational inefficiency. Occluded points may not have corresponding points on the epipolar line. SfMNeRF filters implausible epipolar points by checking if the color difference between the point on the epipolar line and the reference point is below a threshold [S1].
*   **Local Minima:** Large pixel motions can cause the photometric consistency loss to get stuck in local minima. This is mitigated by the pyramidal strategy, starting from low-resolution images and progressively moving to higher resolutions [S1].
*   **Masking:** The mask $M(p(k))$ ensures that only pixels within the minimum bounding rectangle of matched SIFT features are used for photometric consistency, filtering out regions without reliable feature correspondences [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F
import numpy as np

def compute_weighted_depth(samples_z, samples_sigma, origin, direction):
    """
    Compute 3D coordinate of a feature using volume rendering integration.
    samples_z: [N] depth values along ray
    samples_sigma: [N] volume densities
    origin: [3] ray origin
    direction: [3] ray direction
    """
    delta = torch.diff(torch.cat([torch.zeros(1, device=samples_z.device), samples_z]))
    # T_j is transmittance up to sample j
    transmittance = torch.exp(-torch.cumsum(samples_sigma[:-1] * delta[:-1], dim=0))
    weights = transmittance * (1 - torch.exp(-samples_sigma * delta))
    
    # 3D position of each sample
    sample_positions = origin.unsqueeze(0) + samples_z.unsqueeze(1) * direction.unsqueeze(0)
    
    # Weighted sum
    weighted_pos = torch.sum(weights.unsqueeze(1) * sample_positions, dim=0)
    return weighted_pos

def bilinear_interpolate(image, coords):
    """
    coords: [H, W, 2] float coordinates (x, y) in image space
    image: [C, H, W] tensor
    """
    # Normalize coords to [-1, 1] for grid_sample
    # Assuming coords are in [0, H-1], [0, W-1]
    h, w = image.shape[-2], image.shape[-1]
    x_norm = 2 * (coords[..., 0] / (w - 1)) - 1
    y_norm = 2 * (coords[..., 1] / (h - 1)) - 1
    grid = torch.stack([x_norm, y_norm], dim=-1).unsqueeze(0).unsqueeze(0)
    # grid_sample expects [N, C, H, W] and grid [N, H_out, W_out, 2]
    # We need to interpolate at specific float coordinates
    # Using F.grid_sample with align_corners=False
    # Note: This is a simplified representation. In practice, one might use
    # custom CUDA kernels or careful indexing for sub-pixel interpolation.
    # Here we assume a helper that performs exact bilinear interpolation.
    pass

def sfmnerf_step(reference_image, target_image, nerf_model, k_matrix, pose_r, pose_t, 
                 sift_matches, pyramid_level, prev_depth_map=None):
    """
    One step of SfMNeRF optimization for a specific pyramid level.
    """
    # 1. Sub-pixel Sampling
    patch_size = 32
    # Randomly select a patch Omega_p from reference image
    # Generate sub-pixel offsets
    x_off = torch.rand(patch_size, patch_size)
    y_off = torch.rand(patch_size, patch_size)
    
    # Base coordinates of the patch
    base_x, base_y = get_patch_coordinates(reference_image, patch_size)
    
    # Sub-pixel coordinates
    sub_pixel_coords = torch.stack([base_x + x_off, base_y + y_off], dim=-1) # [H, W, 2]
    
    # Ground truth via bilinear interpolation
    gt_colors = bilinear_interpolate(reference_image, sub_pixel_coords)
    
    # 2. Volume Rendering
    # Cast rays through sub-pixel coordinates
    rays_o, rays_d = get_rays_from_coords(sub_pixel_coords, k_matrix, pose_r)
    
    # Sample along rays (Hierarchical sampling: 64 coarse, 64 fine)
    z_coarse, sigma_coarse = nerf_model.sample(rays_o, rays_d, num_samples=64)
    z_fine, sigma_fine = nerf_model.importance_sample(rays_o, rays_d, z_coarse, sigma_coarse, num_samples=64)
    
    # Render color and depth
    rendered_colors, rendered_depths = nerf_model.render(rays_o, rays_d, z_fine, sigma_fine)
    
    # 3. Loss Computation
    
    # L_ren
    l_ren = torch.mean((gt_colors - rendered_colors) ** 2)
    
    # L_3D: Position of matched features
    l_3d = 0.0
    for match in sift_matches:
        # Get 3D coords for reference and target
        ray_ref = get_ray_for_feature(match.ref_coord, k_matrix, pose_r)
        ray_tgt = get_ray_for_feature(match.tgt_coord, k_matrix, pose_t)
        
        z_ref, sigma_ref = nerf_model.sample(ray_ref.o, ray_ref.d, 64)
        x_s_ref = compute_weighted_depth(z_ref, sigma_ref, ray_ref.o, ray_ref.d)
        
        z_tgt, sigma_tgt = nerf_model.sample(ray_tgt.o, ray_tgt.d, 64)
        x_s_tgt = compute_weighted_depth(z_tgt, sigma_tgt, ray_tgt.o, ray_tgt.d)
        
        l_3d += torch.norm(x_s_ref - x_s_tgt) ** 2
    l_3d /= len(sift_matches)
    
    # L_epi: Epipolar constraint
    l_epi = 0.0
    for match in sift_matches:
        # Get 3D coord of reference point
        ray_ref = get_ray_for_feature(match.ref_coord, k_matrix, pose_r)
        z_ref, sigma_ref = nerf_model.sample(ray_ref.o, ray_ref.d, 64)
        d_pr = compute_weighted_depth(z_ref, sigma_ref, ray_ref.o, ray_ref.d)
        
        # Compute epipolar line in target image
        # Filter implausible points by color similarity
        epipolar_points = get_epipolar_line_points(match.ref_coord, k_matrix, pose_r, pose_t, reference_image)
        
        min_dist = float('inf')
        for pt in epipolar_points:
            # Get 3D coord of epipolar point
            ray_tgt_pt = get_ray_for_feature(pt, k_matrix, pose_t)
            z_pt, sigma_pt = nerf_model.sample(ray_tgt_pt.o, ray_tgt_pt.d, 64)
            d_pt = compute_weighted_depth(z_pt, sigma_pt, ray_tgt_pt.o, ray_tgt_pt.d)
            
            dist = torch.norm(d_pr - d_pt) ** 2
            if dist < min_dist:
                min_dist = dist
        l_epi += min_dist
    l_epi /= len(sift_matches)
    
    # L_pr and L_SSIM: Photometric consistency
    # Warp reference patch to target view
    # Get 3D coords of all pixels in patch
    patch_rays_o, patch_rays_d = get_rays_from_coords(sub_pixel_coords, k_matrix, pose_r)
    # Render depth to get 3D positions
    _, _, z_patch, sigma_patch = nerf_model.render_full(patch_rays_o, patch_rays_d)
    patch_3d_coords = compute_3d_coords_from_depth(z_patch, patch_rays_o, patch_rays_d)
    
    # Project to target image
    relative_pose = compute_relative_pose(pose_r, pose_t)
    warped_coords_2d = project_to_image(patch_3d_coords, k_matrix, relative_pose)
    
    # Interpolate target image at warped coords
    warped_colors = bilinear_interpolate(target_image, warped_coords_2d)
    
    # Mask M
    mask = compute_sift_mask(sub_pixel_coords, sift_matches, reference_image.shape)
    
    l_pr = torch.mean(mask * torch.abs(reference_image - warped_colors))
    
    # SSIM loss (simplified)
    l_ssim = torch.mean(mask * (1 - ssim_index(reference_image, warped_colors)) / 2)
    
    # L_ds: Depth smoothness
    depth_grad = compute_depth_gradient(rendered_depths)
    intensity_weight = torch.exp(-torch.abs(rendered_colors))
    l_ds = torch.mean(depth_grad * intensity_weight)
    
    # L_depth: Pyramidal prior
    l_depth = 0.0
    if prev_depth_map is not None:
        # Downsample current coords to match prev_depth_map resolution
        low_res_coords = sub_pixel_coords / 2
        prev_depth_vals = bilinear_interpolate(prev_depth_map, low_res_coords)
        l_depth = torch.mean(torch.abs(rendered_depths - prev_depth_vals))
        
    # Total Loss
    total_loss = (1.0 * l_ren + 
                  1.0 * l_3d + 
                  0.001 * l_pr + 
                  0.0001 * l_epi + 
                  0.008 * l_ssim + 
                  0.01 * l_ds + 
                  1.0 * l_depth) # lambda_depth assumed 1.0 if used, or scaled appropriately
    
    return total_loss

def train_sfmnerf(images, poses, k_matrix, sift_matches):
    """
    Main training loop with pyramidal strategy.
    """
    # Construct pyramid
    levels = construct_pyramid(images)
    
    model = NeRFModel()
    prev_depth_map = None
    
    # Start from lowest level
    for level in reversed(range(len(levels))):
        ref_img = levels[level][0]
        tgt_img = levels[level][1]
        
        # Adjust SIFT matches for current resolution if necessary
        current_matches = adjust_sift_matches_for_resolution(sift_matches, level)
        
        optimizer = torch.optim.Adam(model.parameters(), lr=1e-3)
        
        for iteration in range(num_iterations):
            optimizer.zero_grad()
            loss = sfmnerf_step(ref_img, tgt_img, model, k_matrix, poses[0], poses[1], 
                                current_matches, level, prev_depth_map)
            loss.backward()
            optimizer.step()
            
        # Save depth map for next level (higher resolution)
        # Render depth for the reference image at this level
        prev_depth_map = render_depth_map(model, ref_img, k_matrix, poses[0])
        
    return model
```

## Operating characteristics
*   **Time Complexity:** The method is computationally intensive due to the additional geometric losses. The epipolar constraint requires searching for the minimum distance along the epipolar line for each feature match, which can be optimized by filtering implausible points via color similarity [S1]. The pyramidal strategy reduces the computational burden at higher resolutions by initializing from coarse estimates [S1].
*   **Memory Complexity:** Requires storing the NeRF model, intermediate depth maps for the pyramidal strategy, and SIFT feature matches. The use of sub-pixel rendering increases the number of samples per pixel, slightly increasing memory usage during rendering [S1].
*   **Performance:** Evaluated on LLFF-NeRF, ScanNet, and DTU MVS datasets, SfMNeRF surpasses state-of-the-art approaches given sparse inputs [S1]. It improves novel view synthesis quality and 3D-scene geometry reconstruction compared to standard NeRF, which suffers from inaccurate 3D structures under sparse conditions [S1].
*   **Strengths:** Explicitly constrains geometry using SfM priors, reducing shape-radiance ambiguity. Sub-pixel rendering improves generalization. Pyramidal strategy prevents local minima.
*   **Limitations:** Relies on the quality of SIFT feature matches. Large occlusions or textureless regions may lack sufficient features. The epipolar constraint can be computationally expensive if not filtered properly [S1].

## Background
SfMNeRF was proposed to address the limitations of existing NeRF-based methods in obtaining moderate 3D structures under sparse input conditions, where novel view synthesis quality drops dramatically due to implicitly reconstructed inaccurate 3D-scene structures [S1]. The method leverages knowledge from self-supervised depth estimation and Structure-from-Motion (SfM) to explicitly constrain the 3D-scene geometry during view synthesis training [S1]. It was evaluated on public datasets including LLFF-NeRF, ScanNet, and DTU MVS [S1]. The code is available at https://github.com/XTU-PR-LAB/SfMNeRF [S1].

## Sources
[S1] Improving neural radiance fields with depth-aware optimization for novel view synthesis, https://doi.org/10.1016/j.imavis.2026.106052, fetched 2026-10-26
