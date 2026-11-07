# SfMNeRF

Also called: SfM-NeRF

## Overview
SfMNeRF is a neural radiance field optimization method designed to improve novel view synthesis and 3D geometry reconstruction under sparse input conditions by integrating Structure-from-Motion (SfM) constraints into the training loop [S1]. Unlike standard NeRF which relies solely on photometric consistency, SfMNeRF explicitly constrains the 3D scene structure using epipolar geometry, matched feature positions, and multi-view photometric consistency, thereby reducing geometric ambiguity and preventing local minima [S1]. It distinguishes itself from sibling methods like [AdaNeRF](6195-adanerf.md) and [AligNeRF](8695-alignerf.md) by leveraging explicit cross-view geometric priors (SIFT features and epipolar lines) rather than implicit or learned alignment mechanisms [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $m$ training images $\{I_i\}$ with known camera intrinsics $K$ and global poses $[\mathbf{r}_i, \mathbf{t}_i]$. Matched Scale-Invariant Feature Transform (SIFT) correspondences between image pairs.
*   **Outputs:** A trained Neural Radiance Field (MLP) parameterized by $\Theta$, mapping 5D coordinates $(\mathbf{x}, \mathbf{d})$ to density $\sigma$ and color $\mathbf{c}$, and synthesized novel views.

**Data Structures:**
*   **Camera Intrinsic Matrix $K$:** $3 \times 3$ matrix.
*   **Global Pose:** For image $i$, rotation $\mathbf{r}_i$ and translation $\mathbf{t}_i$.
*   **Relative Pose $T_{r \to t}$:** Derived from global poses of reference ($r$) and target ($t$) images.
*   **SIFT Features:** Set of matched 2D pixel coordinates $\{p^k\}$ between reference and target images.
*   **Mask Map $M$:** Binary mask where $M(p(k))=1$ if point $p(k)$ lies within the minimum bounding rectangle of all matched SIFT features in the reference image, else $0$ [S1].
*   **Depth Map $D$:** Estimated depth for each pixel, derived from volume rendering integration.

**Numbered Steps:**

1.  **Pyramidal Construction:** Construct an $L$-level image pyramid for the reference image $I_r$ and target image $I_t$. Let $I^n_r$ and $I^n_t$ be the images at level $n$, where $I^0$ is the original resolution and $I^n$ is downsampled by half from $I^{n-1}$ [S1].
2.  **Initialization:** Start optimization at the lowest pyramid level $L-1$.
3.  **Sub-pixel Sampling:** For a reference patch $\Omega_p$ (size $32 \times 32$ [S1]), generate sub-pixel coordinates $p = (x + x_{off}, y + y_{off})$ where $x_{off}, y_{off} \in (0, 1)$ [S1].
4.  **Ground Truth Interpolation:** Compute ground-truth color $\hat{I}(p)$ for sub-pixels via bilinear interpolation of the four nearest integer pixels in the reference image [S1]:
    $$ \hat{I}(p) = \sum_{i \in \{top, bottom\}, j \in \{left, right\}} w^{ij} I(p^{ij}) $$
    where $w^{ij}$ is proportional to the inverse Euclidean distance between $p$ and neighbor $p^{ij}$ [S1].
5.  **NeRF Forward Pass:** Input sub-pixel coordinates and viewing directions into the NeRF MLP to obtain density $\sigma$ and color $\mathbf{c}$ for $N_c$ samples along the ray [S1].
6.  **3D Coordinate Estimation:** Calculate the 3D coordinate $\mathbf{x}_s$ of SIFT features and other points using weighted volume rendering integration [S1]:
    $$ \mathbf{x}_s = \sum_{j=1}^{N_c} w_j (\mathbf{o} + t_j \mathbf{d}), \quad w_j = T_j (1 - \exp(-\sigma_j \delta_j)) $$
    where $\mathbf{o}$ is the camera center, $\mathbf{d}$ is the viewing direction, $\delta_j = t_{j-1} - t_j$, and $T_j = \exp(-\sum_{k=1}^{j-1} \sigma_k \delta_k)$ [S1].
7.  **Loss Computation:** Compute the following loss components:
    *   **Rendering Loss ($L_{ren}$):** Mean squared error between rendered color $\tilde{I}(p_i)$ and interpolated ground truth $\hat{I}(p_i)$ [S1]:
        $$ L_{ren} = \frac{1}{m} \sum_{p_i \in \Omega_p} \left\| \hat{I}(p_i) - \tilde{I}(p_i) \right\|_2^2 $$
    *   **Position-of-Matched Features Loss ($L_{3D}$):** Enforces that matched SIFT features have consistent 3D coordinates across views [S1]:
        $$ L_{3D} = \frac{1}{n} \sum_{k=1}^n \|\mathbf{x}_s^k - \mathbf{x}_{s'}^k\|_2^2 $$
        where $\mathbf{x}_s^k$ and $\mathbf{x}_{s'}^k$ are the 3D coordinates of matched feature $k$ in the reference and target images, respectively [S1].
    *   **Epipolar Constraint Loss ($L_{epi}$):** Minimizes the distance between the estimated 3D point $d_{p_r}$ and the epipolar line in the target view. Implausible points on the epipolar line are filtered by color similarity threshold [S1]:
        $$ L_{epi} = \min_t \left\{ \left\| d_{p_r} - d_{p_t} \right\|_2^2 \right\} $$
        where $d_{p_t}$ are points on the epipolar line [S1].
    *   **Photometric Consistency Loss ($L_{pr}$):** Warps pixels from the reference patch to the target view and compares colors. The warping uses the relative pose $T_{r \to t} = [\mathbf{r}, \mathbf{t}]$ where $\mathbf{r} = \mathbf{r}_t^{-1}$ and $\mathbf{t} = -\mathbf{r}_t^{-1}\mathbf{t}_t$ [S1]:
        $$ \hat{p}_t = K T_{r \to t} \mathbf{x}_p $$
        $$ L_{pr} = \sum_{p(k) \in \Omega_p} M(p(k)) \left| \hat{I}_r(p(k)) - \hat{I}_t(\hat{p}_t(k)) \right| $$
        where $\hat{I}_t(\hat{p}_t(k))$ is the color at the projected location in the target image, obtained via bilinear interpolation [S1].
    *   **SSIM Loss ($L_{SSIM}$):** Structural Similarity Index between reference and warped target patches [S1]:
        $$ L_{SSIM} = M \left( \frac{1 - SSIM_{r \to t}}{2} \right) $$
    *   **Depth Smooth Loss ($L_{ds}$):** Encourages smooth depth variations, weighted by image gradient to preserve edges [S1]:
        $$ L_{ds} = \sum_{p(k) \in \Omega_p} \left| \nabla D ( p(k) ) \cdot \left( e^{-|\hat{I}(p(k))|} \right) \right| $$
    *   **Depth Prior Loss ($L_{depth}$):** (Used in pyramidal strategy) Constrains current depth estimate to match upsampled depth from the previous pyramid level [S1]:
        $$ L_{depth} = \sum_{p(k) \in \Omega_p} |D(p(k)) - D_r^{L-1}(p(k)/2)| $$
8.  **Total Loss:** Combine losses with weights [S1]:
    $$ L_{total} = \lambda_{ren} L_{ren} + \lambda_{3D} L_{3D} + \lambda_{pr} L_{pr} + \lambda_{epi} L_{epi} + \lambda_{SSIM} L_{SSIM} + \lambda_{ds} L_{ds} + \lambda_{depth} L_{depth} $$
9.  **Optimization:** Update NeRF parameters using Adam optimizer ($\beta_1=0.9, \beta_2=0.999$, learning rate $10^{-3}$) [S1].
10. **Pyramidal Progression:** After convergence at level $n$, move to level $n-1$. Use the depth map from level $n$ (upsampled via bilinear interpolation) as a prior via $L_{depth}$ [S1]. Repeat until level 0.

**Parameters:**
*   $\lambda_{ren} = 1.0$
*   $\lambda_{3D} = 1.0$
*   $\lambda_{pr} = 0.001$
*   $\lambda_{epi} = 0.0001$
*   $\lambda_{SSIM} = 0.008$
*   $\lambda_{ds} = 0.01$
*   Patch size: $32 \times 32$
*   Samples per ray: 64 (coarse) + 64 (importance) [S1]

**Edge Cases and Pitfalls:**
*   **Epipolar Points:** Large numbers of epipolar points cause computational inefficiency. Occluded points may lack correspondences on the epipolar line. Mitigation: Filter epipolar points by color similarity threshold [S1].
*   **Local Minima:** Large pixel motions can cause optimization to get stuck. Mitigation: Pyramidal strategy starting from low resolution [S1].
*   **Masking:** Pixels outside the minimum bounding rectangle of SIFT matches are masked out in photometric losses to avoid unreliable correspondences [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def compute_subpixel_ground_truth(image, patch_coords):
    """
    Compute ground truth color for sub-pixel coordinates via bilinear interpolation.
    patch_coords: (N, 2) tensor of float coordinates (x, y) within a patch.
    image: (H, W, 3) tensor.
    """
    # Extract 4 neighbors
    x, y = patch_coords[:, 0], patch_coords[:, 1]
    x0, y0 = torch.floor(x).long(), torch.floor(y).long()
    x1, y1 = x0 + 1, y0 + 1
    
    # Clamp to image boundaries
    x0 = torch.clamp(x0, 0, image.shape[1] - 1)
    y0 = torch.clamp(y0, 0, image.shape[0] - 1)
    x1 = torch.clamp(x1, 0, image.shape[1] - 1)
    y1 = torch.clamp(y1, 0, image.shape[0] - 1)
    
    # Weights based on distance
    dx0, dx1 = x - x0.float(), x1.float() - x
    dy0, dy1 = y - y0.float(), y1.float() - y
    
    # Bilinear interpolation
    # Top-left, Top-right, Bottom-left, Bottom-right
    val_tl = image[y0, x0]
    val_tr = image[y0, x1]
    val_bl = image[y1, x0]
    val_br = image[y1, x1]
    
    gt_color = (dx1 * dy1 * val_tl + 
                dx0 * dy1 * val_tr + 
                dx1 * dy0 * val_bl + 
                dx0 * dy0 * val_br)
    return gt_color

def compute_3d_coords_from_samples(o, d, sigma, t_vals):
    """
    Compute 3D coordinate of a feature point using volume rendering weights.
    o: (3,) camera center.
    d: (3,) viewing direction.
    sigma: (N,) density samples.
    t_vals: (N,) sample distances along ray.
    """
    delta = t_vals[1:] - t_vals[:-1]
    # Add a small epsilon to avoid division by zero if needed, though exp handles it
    alpha = 1.0 - torch.exp(-sigma * delta)
    T = torch.cumprod(torch.cat([torch.ones_like(alpha[:1]), 1.0 - alpha[:-1] + 1e-10]), dim=0)
    weights = T * alpha
    
    # 3D points on ray
    points = o.unsqueeze(0) + t_vals.unsqueeze(1) * d.unsqueeze(0)
    # Note: t_vals should be relative to o, or o is origin. 
    # Formula in source: x_s = sum w_j (o + t_j d). Assuming t_j is distance from origin.
    
    x_s = torch.sum(weights.unsqueeze(1) * points, dim=0)
    return x_s

def compute_epipolar_loss(d_pr, epipolar_points):
    """
    Compute minimum distance from point to epipolar line.
    d_pr: (3,) 3D coordinate of reference point.
    epipolar_points: (M, 3) candidate points on epipolar line.
    """
    dists = torch.norm(d_pr.unsqueeze(0) - epipolar_points, dim=1)
    return torch.min(dists)

def warp_to_target(x_p, K, T_r_to_t):
    """
    Warp 3D point to target image 2D coordinates.
    x_p: (N, 3) 3D points.
    K: (3, 3) intrinsic matrix.
    T_r_to_t: (3, 4) relative pose matrix [R|t].
    """
    # Project to homogeneous
    x_h = torch.cat([x_p, torch.ones_like(x_p[:, :1])], dim=1) # (N, 4)
    x_target_h = torch.matmul(x_h, T_r_to_t.t()) # (N, 4)
    x_target = x_target_h[:, :3] / x_target_h[:, 3:4] # (N, 3)
    x_img = torch.matmul(K, x_target.t()).t() # (N, 3)
    return x_img[:, :2] / x_img[:, 2:3] # (N, 2) 2D coordinates

def train_step(images, poses, sift_matches, nerf_model, level, pyramid_images):
    """
    Single training step for a specific pyramid level.
    """
    # 1. Select reference and target images
    # In practice, iterate over pairs or select a reference/target pair
    ref_img = images[0]
    target_img = images[1]
    
    # 2. Get relative pose
    r_t, t_t = poses[1] # Global pose of target
    r_rel = torch.inverse(r_t)
    t_rel = -torch.matmul(r_rel, t_t.unsqueeze(1)).squeeze(1)
    T_r_to_t = torch.eye(4)
    T_r_to_t[:3, :3] = r_rel
    T_r_to_t[:3, 3] = t_rel
    
    # 3. Sample patch and sub-pixels
    # Assume patch_coords is (N, 2) float coordinates in reference image space
    # Add random offset (0,1)
    offsets = torch.rand_like(patch_coords)
    sub_pixel_coords = patch_coords + offsets
    
    # 4. Compute Ground Truth
    gt_color = compute_subpixel_ground_truth(ref_img, sub_pixel_coords)
    
    # 5. NeRF Forward
    # Generate rays from sub_pixel_coords and camera center
    # ... (Ray generation logic omitted for brevity, standard NeRF)
    # Get sigma, color, t_vals for samples
    sigma, color, t_vals, o, d = nerf_model.forward_rays(sub_pixel_coords, ref_img.shape)
    
    # 6. Compute Rendering Loss
    rendered_color = volume_render(color, sigma, t_vals)
    l_ren = F.mse_loss(rendered_color, gt_color)
    
    # 7. Compute 3D Feature Loss
    # For each SIFT match k
    l_3d = 0
    for k in sift_matches:
        # Get 3D coords in ref and target
        x_s_ref = compute_3d_coords_from_samples(o_ref, d_ref, sigma_ref, t_vals_ref)
        x_s_tgt = compute_3d_coords_from_samples(o_tgt, d_tgt, sigma_tgt, t_vals_tgt)
        l_3d += torch.norm(x_s_ref - x_s_tgt)
    l_3d /= len(sift_matches)
    
    # 8. Compute Epipolar Loss
    # Filter epipolar points by color similarity
    # ... (Filtering logic)
    l_epi = 0
    for p in points:
        d_pr = compute_3d_coords_from_samples(...)
        # Find epipolar line points in target
        # ...
        l_epi += compute_epipolar_loss(d_pr, epipolar_points)
        
    # 9. Compute Photometric Consistency Loss
    # Warp reference patch points to target
    x_p_3d = compute_3d_coords_from_samples(...) # 3D points for patch pixels
    p_t_2d = warp_to_target(x_p_3d, K, T_r_to_t)
    
    # Get color at projected points in target image
    target_color_warped = bilinear_interpolate(target_img, p_t_2d)
    
    # Masking
    mask = compute_mask(patch_coords, sift_matches) # 1 if in min rect, else 0
    
    l_pr = torch.sum(mask * torch.abs(ref_img_colors - target_color_warped))
    
    # SSIM Loss
    l_ssim = compute_ssim_loss(ref_patch, target_warped_patch) * mask.mean()
    
    # 10. Depth Smooth Loss
    depth_map = compute_depth_map(sigma, t_vals)
    l_ds = compute_depth_smooth_loss(depth_map, ref_img)
    
    # 11. Depth Prior Loss (if level > 0)
    l_depth = 0
    if level > 0:
        prev_depth = pyramid_images[level-1] # Upsampled depth from previous level
        # Sample prev_depth at current coordinates / 2
        l_depth = torch.abs(depth_map - prev_depth_sampled).mean()
        
    # 12. Total Loss
    total_loss = (1.0 * l_ren + 
                  1.0 * l_3d + 
                  0.001 * l_pr + 
                  0.0001 * l_epi + 
                  0.008 * l_ssim + 
                  0.01 * l_ds + 
                  lambda_depth * l_depth)
                  
    return total_loss
```

## Operating characteristics
*   **Time Complexity:** Higher than standard NeRF due to additional loss computations (SIFT matching, epipolar geometry, warping). The pyramidal strategy requires multiple passes over the data at different resolutions.
*   **Memory Complexity:** Requires storing SIFT features, masks, and intermediate depth maps for the pyramid.
*   **Performance:** Evaluated on LLFF-NeRF, ScanNet, and DTU MVS datasets [S1]. Outperforms state-of-the-art approaches under sparse input conditions [S1].
*   **Strengths:** Explicit geometric constraints improve 3D structure accuracy and novel view synthesis quality when input views are sparse. Sub-pixel rendering improves generalization.
*   **Limitations:** Computational inefficiency due to large number of epipolar points and SIFT processing. Requires accurate initial camera poses and SIFT matches.
*   **When to use:** Sparse view synthesis, scenes with significant geometric ambiguity.
*   **When not to use:** Dense view scenarios where standard NeRF suffices, or when computational resources are severely limited.

## Background
SfMNeRF was proposed to address the failure of existing NeRF-based methods to obtain moderate 3D structures under sparse input conditions [S1]. It integrates knowledge from self-supervised depth estimation and Structure-from-Motion (SfM) to constrain the 3D-scene geometry during view synthesis training [S1]. The method was evaluated on public datasets including LLFF-NeRF, ScanNet, and DTU MVS [S1].

## Sources
[S1] Improving neural radiance fields with depth-aware optimization for novel view synthesis, https://doi.org/10.1016/j.imavis.2026.106052, fetched 2026-10-26
