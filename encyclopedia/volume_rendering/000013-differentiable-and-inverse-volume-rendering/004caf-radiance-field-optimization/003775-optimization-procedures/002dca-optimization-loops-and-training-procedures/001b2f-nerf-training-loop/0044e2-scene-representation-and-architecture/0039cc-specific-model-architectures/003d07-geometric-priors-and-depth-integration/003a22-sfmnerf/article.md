# SfMNeRF

Also called: SfM-NeRF

## Overview
SfMNeRF is a neural radiance field optimization method designed to improve novel view synthesis and 3D geometry reconstruction under sparse input conditions by integrating Structure-from-Motion (SfM) constraints into the training loop [S1]. Unlike standard NeRF which relies solely on photometric error, SfMNeRF explicitly constrains the 3D scene structure using matched feature positions, epipolar geometry, and multi-view photometric consistency [S1]. It distinguishes itself from sibling methods like DASNeRF and GeoNeRF by utilizing sparse 2D feature matches (SIFT) and epipolar lines rather than dense depth priors or signed distance functions, combined with a sub-pixel rendering strategy to increase sample density [S1].

## How it works
### Inputs and Outputs
**Inputs:**
*   A set of $m$ training images with known camera intrinsics $K$ and global poses $[\mathbf{r}_i, \mathbf{t}_i]$.
*   Matched Scale-Invariant Feature Transform (SIFT) correspondences between image pairs.
*   A neural radiance field (MLP) parameterized by $\theta$, mapping 6D coordinates $(\mathbf{x}, \mathbf{d})$ to volume density $\sigma$ and color $\mathbf{c}$ [S1].

**Outputs:**
*   Optimized NeRF parameters $\theta$.
*   Synthesized novel views and depth maps.

### Data Structures
*   **Camera Intrinsic Matrix ($K$):** $3 \times 3$ matrix for each view.
*   **Global Pose:** $[\mathbf{r}_i, \mathbf{t}_i]$ where $\mathbf{r}_i$ is rotation and $\mathbf{t}_i$ is translation for image $i$.
*   **Relative Pose ($T_{r \to t}$):** Derived from global poses of reference ($r$) and target ($t$) images:
    $$ \mathbf{r} = \mathbf{r}_t^{-1}, \quad \mathbf{t} = -\mathbf{r}_t^{-1}\mathbf{t}_t $$
    $$ T_{r \to t} = [\mathbf{r}, \mathbf{t}] \quad [S1] $$
*   **Mask Map ($M$):** Binary mask where $M(p(k))=1$ if point $p(k)$ is inside the minimum rectangle enclosing all matched SIFT features in the reference image, else $0$ [S1].
*   **Depth Map ($D$):** Estimated depth for each pixel, derived from volume rendering integration.
*   **Pyramid Levels:** $L$-level pyramid representations of images $I^0, \dots, I^{L-1}$, where $I^n$ is half the resolution of $I^{n-1}$ [S1].

### Numbered Steps

1.  **Pyramid Construction:** Construct $L$-level image pyramids for reference and target images. Start optimization at the lowest resolution level $L-1$ [S1].
2.  **Feature Extraction:** Extract SIFT features from the reference and target images at the current pyramid level and obtain matched correspondences [S1].
3.  **Sub-pixel Sampling:**
    *   Randomly select a patch $\Omega_p$ (size $32 \times 32$) from the reference image [S1].
    *   For each pixel $p=(x,y)$ in the patch, generate sub-pixel coordinates:
        $$ p = (x + x_{off}, y + y_{off}) $$
        where $x_{off}, y_{off} \in (0, 1)$ [S1].
    *   Compute ground-truth sub-pixel color $\hat{I}(p)$ via bilinear interpolation of the four nearest integer pixels:
        $$ \hat{I}(p) = \sum_{i \in \{top, bottom\}, j \in \{left, right\}} w^{ij} I(p^{ij}) $$
        where $w^{ij}$ is proportional to the inverse Euclidean distance between $p$ and neighbor $p^{ij}$ [S1].
4.  **Volume Rendering:**
    *   Cast rays through the sub-pixel coordinates and directions $\mathbf{d}$.
    *   Sample points along the ray. Compute volume density $\sigma_j$ and color $\mathbf{c}_j$.
    *   Compute rendered color $\tilde{I}(p)$ and depth $D(p)$ using standard volume rendering integration [S1].
    *   Compute 3D coordinates of SIFT features $\mathbf{x}_s$ using the weighted sum of samples:
        $$ \mathbf{x}_s = \sum_{j=1}^{N_c} w_j (\mathbf{o} + t_j \mathbf{d}) $$
        $$ w_j = T_j (1 - \exp(-\sigma_j \delta_j)), \quad \delta_j = t_{j-1} - t_j $$
        where $N_c$ is the number of samples, $\mathbf{o}$ is the camera origin, and $t_j$ are sample depths [S1].
5.  **Loss Computation:** Calculate the following loss terms:
    *   **Rendering Loss ($L_{ren}$):** Mean squared error between rendered color and interpolated ground-truth color.
        $$ L_{ren} = \frac{1}{m} \sum_{p_i \in \Omega_p} \left\| \hat{I}(p_i) - \tilde{I}(p_i) \right\|_2^2 \quad [S1] $$
    *   **Positions-of-Matched Features Loss ($L_{3D}$):** Enforce that matched SIFT features have identical 3D coordinates in reference and target views.
        $$ L_{3D} = \frac{1}{n} \sum_{k=1}^n \|\mathbf{x}_s^k - \mathbf{x}_{s'}^k\|_2^2 $$
        where $\mathbf{x}_s^k$ and $\mathbf{x}_{s'}^k$ are 3D coordinates of matched feature $k$ in reference and target images [S1].
    *   **Epipolar Constraint Loss ($L_{epi}$):**
        *   For a point in the reference image, compute its 3D coordinate $\mathbf{d}_{p_r}$.
        *   Identify points on the epipolar line in the target image.
        *   Filter implausible points: keep only points where the color difference between the reference point and the epipolar point is below a threshold [S1].
        *   Minimize the distance to the closest valid epipolar point:
            $$ L_{epi} = \min_t \left\{ \left\| \mathbf{d}_{p_r} - \mathbf{d}_{p_t} \right\|_2^2 \right\} $$
            where $\mathbf{d}_{p_t}$ is the 3D coordinate of the epipolar point [S1].
    *   **Photometric Consistency Loss ($L_{pr}$):**
        *   Warp pixels from the reference patch to the target view:
            $$ \hat{p}_t = K T_{r \to t} \mathbf{x}_p $$
            where $\mathbf{x}_p$ is the 3D coordinate of pixel $p$ [S1].
        *   Interpolate color at $\hat{p}_t$ in the target image to get $\hat{I}_t(\hat{p}_t)$.
        *   Compute L1 difference masked by $M$:
            $$ L_{pr} = \sum_{p(k) \in \Omega_p} M(p(k)) \left| \hat{I}_r(p(k)) - \hat{I}_t(\hat{p}_t(k)) \right| \quad [S1] $$
    *   **SSIM Loss ($L_{SSIM}$):**
        $$ L_{SSIM} = M \left( \frac{1 - SSIM_{r \to t}}{2} \right) $$
        where $SSIM_{r \to t}$ is the structural similarity index between the reference patch and the warped target patch [S1].
    *   **Depth Smooth Loss ($L_{ds}$):**
        $$ L_{ds} = \sum_{p(k) \in \Omega_p} \left| \nabla D(p(k)) \cdot \left( e^{-|\hat{I}(p(k))|} \right) \right| $$
        where $\nabla$ is the 2D differential operator [S1].
    *   **Depth Prior Loss ($L_{depth}$):** (Used only at levels $n < L-1$)
        $$ L_{depth} = \sum_{p(k) \in \Omega_p} |D(p(k)) - D_r^{L-1}(p(k)/2)| $$
        where $D_r^{L-1}$ is the depth map from the previous (lower resolution) level, sampled via bilinear interpolation at half coordinates [S1].
6.  **Total Loss:**
    $$ L_{total} = \lambda_{ren} L_{ren} + \lambda_{3D} L_{3D} + \lambda_{pr} L_{pr} + \lambda_{epi} L_{epi} + \lambda_{SSIM} L_{SSIM} + \lambda_{ds} L_{ds} + \lambda_{depth} L_{depth} \quad [S1] $$
7.  **Optimization:** Update NeRF parameters $\theta$ using Adam optimizer ($\beta_1=0.9, \beta_2=0.999$, learning rate $1e-3$) [S1].
8.  **Pyramidal Progression:** After optimizing level $L-1$, use the resulting depth map as a prior for level $L-2$, and repeat recursively up to level $0$ [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F
import numpy as np

def compute_volume_rendering(ray_o, ray_d, samples, sigma_fn, color_fn):
    """
    Standard NeRF volume rendering.
    Returns: rendered_color, rendered_depth, 3d_coords_of_samples
    """
    # samples: (N_samples, 3) positions along ray
    # ray_o: (3,) camera origin
    # ray_d: (3,) camera direction
    
    # Compute distances between samples
    deltas = samples[1:] - samples[:-1]
    deltas = torch.cat([deltas, torch.tensor([1e10], device=samples.device).expand(-1, 3)], dim=0)
    
    # Compute weights
    # T_j = exp(-sum_{s<j} sigma_s * delta_s)
    # w_j = T_j * (1 - exp(-sigma_j * delta_j))
    # Note: Simplified implementation assuming standard NeRF sampling
    alphas = 1.0 - torch.exp(-F.relu(sigma_fn(samples)) * deltas.norm(dim=1))
    transmittance = torch.cumprod(1.0 - alphas + 1e-10, dim=0)
    weights = alphas * transmittance
    weights = weights / (weights.sum() + 1e-10) # Normalize
    
    colors = color_fn(samples, ray_d)
    rendered_color = (weights.unsqueeze(-1) * colors).sum(dim=0)
    
    # Depth is weighted average of sample depths (z-coords along ray)
    # Assuming samples are ordered by depth
    rendered_depth = (weights * samples[:, 2]).sum() # Simplified depth
    
    return rendered_color, rendered_depth, samples

def bilinear_interpolate(image, points):
    """
    Image: (H, W, C)
    Points: (N, 2) float coordinates (x, y)
    Returns: (N, C) interpolated colors
    """
    # Implementation of bilinear interpolation
    # ...
    pass

def get_subpixel_ground_truth(image, patch_coords):
    """
    image: (H, W, C)
    patch_coords: (N, 2) integer pixel coordinates
    Returns: (N, C) ground truth colors with sub-pixel offsets
    """
    # Add random offset (0, 1)
    offsets = torch.rand_like(patch_coords)
    subpixel_coords = patch_coords + offsets
    
    # Interpolate
    gt_colors = bilinear_interpolate(image, subpixel_coords)
    return gt_colors, subpixel_coords

def compute_sift_3d_coords(samples, sigma_fn, ray_o, ray_d):
    """
    Compute 3D coordinates of a feature point using volume rendering weights.
    """
    # Compute weights w_j as in Eq. 7
    # ...
    # x_s = sum(w_j * (o + t_j * d))
    pass

def compute_epipolar_loss(point_3d_ref, epipolar_points_3d):
    """
    point_3d_ref: (3,)
    epipolar_points_3d: (N, 3)
    Returns: min distance to epipolar line
    """
    # Filter implausible points by color similarity (not shown here)
    # Compute min distance
    dists = torch.norm(epipolar_points_3d - point_3d_ref, dim=1)
    return dists.min()

def train_step(images, poses, intrinsics, sift_matches, pyramid_level, model):
    """
    Single training step for a specific pyramid level.
    """
    # 1. Select reference and target images
    ref_img = images[pyramid_level][0]
    tgt_img = images[pyramid_level][1]
    
    # 2. Get SIFT matches
    matches = sift_matches[pyramid_level]
    
    # 3. Sub-pixel rendering setup
    patch_size = 32
    patch_coords = random_patch(ref_img.shape, patch_size)
    gt_colors, subpixel_coords = get_subpixel_ground_truth(ref_img, patch_coords)
    
    # 4. Render
    # Cast rays through subpixel_coords
    # ...
    rendered_colors, rendered_depths, samples = render_rays(subpixel_coords, model)
    
    # 5. Compute Losses
    L_ren = F.mse_loss(rendered_colors, gt_colors)
    
    # L_3D: Positions of matched features
    # Compute 3D coords for SIFT matches in ref and tgt
    # ...
    L_3d = compute_3d_loss(sift_matches, model, poses)
    
    # L_epi: Epipolar constraint
    # ...
    L_epi = compute_epipolar_loss(sift_matches, model, poses, intrinsics)
    
    # L_pr: Photometric consistency
    # Warp ref patch to target
    # ...
    L_pr = compute_photometric_loss(ref_img, tgt_img, model, poses, intrinsics)
    
    # L_SSIM
    L_ssim = compute_ssim_loss(ref_img, tgt_img, model, poses, intrinsics)
    
    # L_ds: Depth smooth
    L_ds = compute_depth_smooth_loss(rendered_depths, gt_colors)
    
    # L_depth: Prior from lower level (if not lowest level)
    L_depth = 0
    if pyramid_level > 0:
        prev_depth_map = get_depth_map_from_prev_level(pyramid_level - 1)
        L_depth = compute_depth_prior_loss(rendered_depths, prev_depth_map)
        
    # Total Loss
    total_loss = (1.0 * L_ren + 
                  1.0 * L_3d + 
                  0.001 * L_pr + 
                  0.0001 * L_epi + 
                  0.008 * L_ssim + 
                  0.01 * L_ds + 
                  L_depth) # Lambda for depth prior is implicitly 1 or handled in loop
    
    return total_loss
```

### Parameters
*   **Loss Weights:** $\lambda_{ren} = 1.0$, $\lambda_{3D} = 1.0$, $\lambda_{pr} = 0.001$, $\lambda_{epi} = 0.0001$, $\lambda_{SSIM} = 0.008$, $\lambda_{ds} = 0.01$ [S1].
*   **Patch Size:** $32 \times 32$ [S1].
*   **Optimizer:** Adam, $\beta_1 = 0.9$, $\beta_2 = 0.999$, learning rate $1e-3$ [S1].
*   **Sampling:** Hierarchical sampling with 64 coarse samples and 64 importance samples [S1].
*   **Pyramid Levels:** $L$ levels, where resolution halves at each level [S1].

### Edge Cases and Pitfalls
*   **Epipolar Point Filtering:** A large number of epipolar points causes computational inefficiency. Occluded points may not have corresponding points on the epipolar line. The method filters implausible points using color similarity (thresholding color difference) [S1].
*   **Local Minima:** Large pixel motions can cause the photometric consistency loss to get stuck in local minima. This is mitigated by the pyramidal strategy, starting from low resolution [S1].
*   **Masking:** The mask $M$ is defined by the minimum rectangle enclosing all matched SIFT features. Points outside this rectangle are ignored in photometric losses to avoid erroneous constraints in non-overlapping regions [S1].

## Operating characteristics
*   **Time Complexity:** Depends on the number of pyramid levels $L$, the number of samples per ray, and the number of SIFT matches. The epipolar constraint requires searching along epipolar lines, which can be expensive if not filtered [S1].
*   **Memory Complexity:** Requires storing the NeRF MLP, intermediate depth maps for the pyramid, and SIFT matches. The pyramidal strategy allows processing high-resolution images by starting with low-resolution representations [S1].
*   **Performance:** Evaluated on LLFF-NeRF, ScanNet, and DTU MVS datasets. It outperforms state-of-the-art approaches under sparse input conditions [S1].
*   **Strengths:** Explicit geometric constraints improve 3D structure accuracy and novel view synthesis quality with sparse inputs. Sub-pixel rendering increases sample density and generalization [S1].
*   **Limitations:** Relies on the quality of SIFT feature matching. Computationally intensive due to multiple loss terms and epipolar searches.

## Background
SfMNeRF was proposed to address the failure of existing NeRF-based methods to obtain moderate 3D structures under sparse input conditions, where novel view synthesis quality drops dramatically due to implicitly reconstructed inaccurate 3D-scene structures [S1]. It leverages knowledge from self-supervised depth estimation methods to constrain the 3D-scene geometry during view synthesis training [S1]. The method was evaluated on LLFF-NeRF, ScanNet, and DTU MVS datasets [S1].

## Sources
[S1] Improving neural radiance fields with depth-aware optimization for novel view synthesis, https://doi.org/10.1016/j.imavis.2026.106052, fetched 2026-10-26
