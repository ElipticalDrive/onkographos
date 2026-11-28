# SfMNeRF

Also called: SfM-NeRF

## Overview
SfMNeRF is a method for novel view synthesis and 3D-scene geometry reconstruction using Neural Radiance Fields (NeRF) under sparse input conditions [S1]. It improves geometric accuracy and avoids local minima by integrating Structure-from-Motion (SfM) constraints—specifically matched feature positions, epipolar geometry, and multi-view photometric consistency—into the view synthesis training loop [S1]. Unlike standard NeRF, which relies solely on implicit photometric consistency, SfMNeRF explicitly constrains the 3D-scene structure using sparse feature matches and epipolar lines, while employing sub-pixel rendering to increase sample density and generalization [S1].

## How it works
SfMNeRF builds on [Specific Model Architectures](8585-specific-model-architectures.md) by augmenting the standard NeRF optimization with explicit geometric losses derived from sparse features and multi-view geometry [S1].

### Inputs and Outputs
**Inputs:**
*   A set of $m$ training images with known camera intrinsics $K$ and global poses $[\mathbf{r}_i, \mathbf{t}_i]$ [S1].
*   Matched Scale-Invariant Feature Transform (SIFT) correspondences between image pairs [S1].
*   A pyramidal representation of the input images for coarse-to-fine optimization [S1].

**Outputs:**
*   A trained NeRF model (MLP parameters $\theta$) representing volume density $\sigma_\theta$ and color $c_\theta$ [S1].
*   Synthesized novel views and depth maps [S1].

### Data Structures
*   **Camera Intrinsic Matrix ($K$):** $3 \times 3$ matrix mapping 3D world coordinates to 2D image coordinates [S1].
*   **Relative Pose ($T_{r \to t}$):** Transformation matrix from reference image $r$ to target image $t$, composed of rotation $\mathbf{r}$ and translation $\mathbf{t}$ [S1].
*   **SIFT Features:** A set of 2D pixel coordinates in reference and target images, matched across views [S1].
*   **Mask Map ($M$):** A binary mask where $M(p(k))=1$ if point $p(k)$ lies within the minimum bounding rectangle of all matched SIFT features in the reference image, else $0$ [S1].
*   **Depth Map ($D$):** Estimated depth for each pixel, computed via volume rendering integration [S1].

### Numbered Steps
1.  **Pyramidal Construction:** Construct an $L$-level pyramid for the reference image $I_r$ and target image $I_t$ [S1]. Let $I^0_r = I_r$ and $I^0_t = I_t$ [S1]. For level $n > 0$, $I^n_r$ and $I^n_t$ are downsampled versions of $I^{n-1}_r$ and $I^{n-1}_t$ with half the resolution [S1].
2.  **Initialization:** Start optimization at the lowest pyramid level $L-1$ [S1].
3.  **Sub-pixel Sampling:** For a selected patch $\Omega_p$ in the reference image, generate sub-pixel coordinates $p = (x + x_{off}, y + y_{off})$ where $x_{off}, y_{off} \in (0, 1)$ [S1].
4.  **Ground Truth Interpolation:** Compute the ground-truth color $\hat{I}(p)$ for each sub-pixel using bilinear interpolation of the four nearest integer-pixel neighbors in the reference image [S1].
5.  **Volume Rendering:** Input sub-pixel coordinates and viewing directions into the NeRF to render color $\tilde{I}(p)$ and depth $D(p)$ [S1].
6.  **Feature Extraction:** Extract SIFT features from the reference and target images and obtain matched correspondences [S1].
7.  **3D Coordinate Estimation:** For each matched SIFT feature $k$, compute its 3D coordinate $\mathbf{x}_s^k$ in the reference and target views using weighted volume rendering integration [S1].
8.  **Loss Computation:** Calculate the following loss terms: rendering loss ($L_{ren}$), position-of-matches loss ($L_{3D}$), epipolar loss ($L_{epi}$), photometric consistency loss ($L_{pr}$), SSIM loss ($L_{SSIM}$), depth smoothness loss ($L_{ds}$), and depth prior loss ($L_{depth}$) [S1].
9.  **Optimization:** Minimize the total loss $L_{total}$ [S1].
10. **Pyramidal Progression:** Recursively move to level $n-2$, using the depth map from level $n-1$ as a prior via $L_{depth}$, until the "zeroth" level is reached [S1].

### Formulas
**Rendering Loss:**
$$ L_{ren} = \frac{1}{m} \sum_{p_i \in \Omega_p} \left\| \hat{I}(p_i) - \tilde{I}(p_i) \right\|_2^2 $$
where $\hat{I}(p_i)$ is the ground-truth color from interpolation and $\tilde{I}(p_i)$ is the synthesized color from NeRF [S1].

**Position-of-Matched Features Loss:**
$$ L_{3D} = \frac{1}{n} \sum_{k=1}^n \|\mathbf{x}_s^k - \mathbf{x}_{s'}^k\|_2^2 $$
where $\mathbf{x}_s^k$ and $\mathbf{x}_{s'}^k$ are the estimated 3D coordinates of matched SIFT feature $k$ in the reference and target images, respectively [S1].

**Epipolar Constraint Loss:**
$$ L_{epi} = \min_t \left\{ \left\| \mathbf{d}_{p_r} - \mathbf{d}_{p_t} \right\|_2^2 \right\} $$
where $\mathbf{d}_{p_r}$ is the 3D coordinate of the point in the reference image, and $\mathbf{d}_{p_t}$ are the 3D coordinates of points on the epipolar line in the target image [S1]. Implausible points on the epipolar line are filtered by color similarity threshold [S1].

**Warped Pixel Coordinate:**
$$ \hat{p}_t = K T_{r \to t} \mathbf{x}_p $$
where $\mathbf{x}_p$ is the 3D coordinate of pixel $p$ in the reference image, $K$ is the intrinsic matrix, and $T_{r \to t} = [\mathbf{r}, \mathbf{t}]$ is the relative pose [S1].

**Relative Pose Definition:**
$$ \mathbf{r} = \mathbf{r}_t^{-1}, \quad \mathbf{t} = -\mathbf{r}_t^{-1}\mathbf{t}_t $$
where $[\mathbf{r}_t, \mathbf{t}_t]$ is the global pose of the target image [S1].

**Photometric Reconstruction Loss:**
$$ L_{pr} = \sum_{p(k) \in \Omega_p} M(p(k)) \left| \hat{I}_r(p(k)) - \hat{I}_t(\hat{p}_t(k)) \right| $$
where $\hat{I}_r$ is the reference patch color, $\hat{I}_t(\hat{p}_t(k))$ is the interpolated color of the warped point in the target image, and $M$ is the binary mask [S1].

**SSIM Loss:**
$$ L_{SSIM} = M \left( \frac{1 - SSIM_{r \to t}}{2} \right) $$
where $SSIM_{r \to t}$ is the structural similarity between the reference and target patches [S1].

**Depth Smooth Loss:**
$$ L_{ds} = \sum_{p(k) \in \Omega_p} \left| \nabla D ( p(k) ) \cdot \left( e^{-|\hat{I}(p(k))|} \right) \right| $$
where $\nabla$ is the 2D differential operator and $D(p(k))$ is the estimated depth [S1].

**Depth Prior Loss (Pyramidal):**
$$ L_{depth} = \sum_{p(k) \in \Omega_p} |D(p(k)) - D_r^{L-1}(p(k)/2)| $$
where $D_r^{L-1}(p(k)/2)$ is the depth value sampled via bilinear interpolation from the lower-resolution depth map $D_r^{L-1}$ [S1].

**Total Loss:**
$$ L_{total} = \lambda_{ren} L_{ren} + \lambda_{3D} L_{3D} + \lambda_{pr} L_{pr} + \lambda_{epi} L_{epi} + \lambda_{SSIM} L_{SSIM} + \lambda_{ds} L_{ds} + \lambda_{depth} L_{depth} $$
where $\lambda$ terms are loss weights [S1].

### Parameters
*   **Loss Weights:** Not given in the sources.
*   **Patch Size:** Not given in the sources.
*   **Optimizer:** Not given in the sources.
*   **Sampling:** Not given in the sources.

### Edge Cases and Pitfalls
*   **Epipolar Point Filtering:** The large number of points on an epipolar line causes computational inefficiency, and occluded points may lack corresponding points on the epipolar line [S1]. SfMNeRF filters implausible epipolar points by checking if the color difference between the point on the epipolar line and the reference point is below a threshold [S1].
*   **Local Minima:** Large pixel motions can cause the photometric consistency loss to get stuck in local minima [S1]. This is mitigated by the pyramidal strategy, which starts from low-resolution images and progressively moves to higher resolutions [S1].
*   **Masking:** The mask $M(p(k))$ ensures that only pixels within the minimum bounding rectangle of matched SIFT features are used for photometric consistency, filtering out regions without reliable feature correspondences [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F
import numpy as np

def bilinear_interpolate(image, coords):
    """Bilinear interpolation at float coordinates.
    image: [C, H, W] tensor
    coords: [N, 2] tensor of (x, y) in pixel space [0, W-1], [0, H-1]
    """
    h, w = image.shape[-2], image.shape[-1]
    x_norm = 2 * (coords[:, 0] / (w - 1)) - 1
    y_norm = 2 * (coords[:, 1] / (h - 1)) - 1
    grid = torch.stack([x_norm, y_norm], dim=-1).unsqueeze(0).unsqueeze(0)
    # grid_sample expects [N, C, H, W] and grid [N, H_out, W_out, 2]
    # We reshape to interpolate at N points
    out = F.grid_sample(image.unsqueeze(0), grid, align_corners=False, mode='bilinear')
    return out.squeeze(0).squeeze(-1).squeeze(-1)

def compute_weighted_depth(samples_z, samples_sigma, origin, direction):
    """Compute 3D coordinate of a feature using volume rendering integration.
    samples_z: [N] depth values along ray
    samples_sigma: [N] volume densities
    origin: [3] ray origin
    direction: [3] ray direction
    """
    delta = torch.diff(torch.cat([torch.zeros(1, device=samples_z.device), samples_z]))
    transmittance = torch.exp(-torch.cumsum(samples_sigma[:-1] * delta[:-1], dim=0))
    weights = transmittance * (1 - torch.exp(-samples_sigma * delta))
    sample_positions = origin.unsqueeze(0) + samples_z.unsqueeze(1) * direction.unsqueeze(0)
    weighted_pos = torch.sum(weights.unsqueeze(1) * sample_positions, dim=0)
    return weighted_pos

def construct_pyramid(images, levels=4):
    """Construct L-level pyramid by halving resolution each level."""
    pyr = [images]
    curr = images
    for _ in range(levels - 1):
        curr = F.interpolate(curr, scale_factor=0.5, mode='bilinear', align_corners=False)
        pyr.append(curr)
    return pyr

def compute_relative_pose(pose_r, pose_t):
    """Compute relative pose T_{r->t} from global poses."""
    R_t, t_t = pose_t[:3, :3], pose_t[:3, 3]
    R_r, t_r = pose_r[:3, :3], pose_r[:3, 3]
    R_inv = R_t.inverse()
    r = R_inv @ R_r
    t = -R_inv @ t_t + R_inv @ R_r @ t_r
    T_rel = torch.eye(4, device=pose_r.device)
    T_rel[:3, :3] = r
    T_rel[:3, 3] = t
    return T_rel

def project_to_image(points_3d, K, T_rel):
    """Project 3D points to 2D image coordinates."""
    points_h = torch.cat([points_3d, torch.ones_like(points_3d[:, :1])], dim=1)
    points_cam = (T_rel @ points_h.t()).t()[:, :3]
    points_2d = (K @ points_cam.t()).t()[:, :2]
    return points_2d

def compute_sift_mask(coords, matches, img_shape):
    """Compute binary mask M(p(k)) based on SIFT bounding box."""
    h, w = img_shape[-2], img_shape[-1]
    if len(matches) == 0:
        return torch.zeros(h, w, dtype=torch.float32, device=coords.device)
    ref_coords = torch.stack([m.ref_x, m.ref_y] for m in matches)
    min_x, max_x = ref_coords[:, 0].min(), ref_coords[:, 0].max()
    min_y, max_y = ref_coords[:, 1].min(), ref_coords[:, 1].max()
    mask = torch.zeros(h, w, dtype=torch.float32, device=coords.device)
    y_idx, x_idx = torch.meshgrid(torch.arange(h), torch.arange(w), indexing='ij')
    mask[(x_idx >= min_x) & (x_idx <= max_x) & (y_idx >= min_y) & (y_idx <= max_y)] = 1.0
    return mask

def compute_depth_gradient(depth_map):
    """Compute L1 norm of 2D depth gradient."""
    dy = torch.abs(depth_map[:, 1:, :] - depth_map[:, :-1, :])
    dx = torch.abs(depth_map[:, :, 1:] - depth_map[:, :, :-1])
    return torch.cat([dy, dx], dim=1)

def get_rays_from_coords(coords, K, pose):
    """Generate ray origins and directions from 2D coordinates."""
    h, w = coords.shape[0], coords.shape[1]
    y, x = torch.meshgrid(torch.arange(h), torch.arange(w), indexing='ij')
    xy = torch.stack([x.float(), y.float()], dim=-1).view(-1, 2)
    xy_h = torch.cat([xy, torch.ones_like(xy[:, :1])], dim=1)
    inv_K = K.inverse()
    cam_dirs = (inv_K @ xy_h.t()).t()[:, :3]
    cam_dirs = cam_dirs / torch.norm(cam_dirs, dim=1, keepdim=True)
    R = pose[:3, :3]
    dirs = (R @ cam_dirs.t()).t()
    origins = pose[:3, 3].unsqueeze(0).expand(h * w, -1)
    return origins.view(h, w, 3), dirs.view(h, w, 3)

def render_neurf(rays_o, rays_d, nerf_model, num_samples=64):
    """Simplified NeRF rendering returning color and depth."""
    # Standard hierarchical sampling would occur here [G]
    z = torch.linspace(0.2, 6.0, num_samples, device=rays_o.device).view(1, 1, num_samples).expand_as(rays_o[..., :1])
    pts = rays_o + rays_d * z
    sigma, color = nerf_model(pts, rays_d)
    # Alpha compositing
    delta = 0.1
    alpha = 1.0 - torch.exp(-sigma * delta)
    T = torch.cumprod(1.0 - alpha + 1e-10, dim=-2)
    weights = alpha * T[..., :-1, :]
    rendered_color = torch.sum(weights * color, dim=-2)
    rendered_depth = torch.sum(weights * z[..., 0], dim=-2)
    return rendered_color, rendered_depth

def sfmnerf_step(reference_image, target_image, nerf_model, k_matrix, pose_r, pose_t, 
                 sift_matches, prev_depth_map=None):
    """One optimization step for a specific pyramid level."""
    h, w = reference_image.shape[-2], reference_image.shape[-1]
    
    # 1. Sub-pixel Sampling
    x_off = torch.rand(h, w, device=reference_image.device)
    y_off = torch.rand(h, w, device=reference_image.device)
    base_x, base_y = torch.meshgrid(torch.arange(w), torch.arange(h), indexing='ij')
    sub_pixel_coords = torch.stack([base_x.float() + x_off, base_y.float() + y_off], dim=-1)
    
    # 2. Ground Truth Interpolation
    gt_colors = bilinear_interpolate(reference_image, sub_pixel_coords.view(-1, 2)).view(h, w, 3)
    
    # 3. Volume Rendering
    rays_o, rays_d = get_rays_from_coords(sub_pixel_coords, k_matrix, pose_r)
    rendered_colors, rendered_depths = render_neurf(rays_o, rays_d, nerf_model)
    
    # 4. Loss Computation
    l_ren = torch.mean((gt_colors - rendered_colors) ** 2)
    
    l_3d = 0.0
    l_epi = 0.0
    for match in sift_matches:
        ray_ref_o, ray_ref_d = get_rays_from_coords(match.ref_coord.unsqueeze(0), k_matrix, pose_r)
        ray_tgt_o, ray_tgt_d = get_rays_from_coords(match.tgt_coord.unsqueeze(0), k_matrix, pose_t)
        
        z_ref, sigma_ref = nerf_model.sample(ray_ref_o, ray_ref_d)
        x_s_ref = compute_weighted_depth(z_ref, sigma_ref, ray_ref_o[0], ray_ref_d[0])
        
        z_tgt, sigma_tgt = nerf_model.sample(ray_tgt_o, ray_tgt_d)
        x_s_tgt = compute_weighted_depth(z_tgt, sigma_tgt, ray_tgt_o[0], ray_tgt_d[0])
        
        l_3d += torch.norm(x_s_ref - x_s_tgt) ** 2
        
        # Epipolar constraint
        d_pr = x_s_ref
        # Generate epipolar line points (simplified sampling along line) [G]
        epipolar_pts = generate_epipolar_line_points(match.ref_coord, k_matrix, pose_r, pose_t, h, w)
        min_dist = float('inf')
        for pt in epipolar_pts:
            ray_pt_o, ray_pt_d = get_rays_from_coords(pt.unsqueeze(0), k_matrix, pose_t)
            z_pt, sigma_pt = nerf_model.sample(ray_pt_o, ray_pt_d)
            d_pt = compute_weighted_depth(z_pt, sigma_pt, ray_pt_o[0], ray_pt_d[0])
            dist = torch.norm(d_pr - d_pt) ** 2
            if dist < min_dist:
                min_dist = dist
        l_epi += min_dist
        
    l_3d /= max(len(sift_matches), 1)
    l_epi /= max(len(sift_matches), 1)
    
    # Photometric consistency
    patch_rays_o, patch_rays_d = get_rays_from_coords(sub_pixel_coords, k_matrix, pose_r)
    _, z_patch = nerf_model.sample(patch_rays_o, patch_rays_d)
    patch_3d = patch_rays_o + patch_rays_d * z_patch
    relative_pose = compute_relative_pose(pose_r, pose_t)
    warped_coords_2d = project_to_image(patch_3d.view(-1, 3), k_matrix, relative_pose).view(h, w, 2)
    warped_colors = bilinear_interpolate(target_image, warped_coords_2d.view(-1, 2)).view(h, w, 3)
    
    mask = compute_sift_mask(sub_pixel_coords, sift_matches, reference_image.shape)
    l_pr = torch.mean(mask * torch.abs(reference_image - warped_colors))
    l_ssim = torch.mean(mask * (1.0 - ssim_index(reference_image, warped_colors)) / 2.0)
    
    # Depth smoothness
    depth_grad = compute_depth_gradient(rendered_depths)
    intensity_weight = torch.exp(-torch.abs(rendered_colors))
    l_ds = torch.mean(depth_grad * intensity_weight)
    
    # Depth prior
    l_depth = 0.0
    if prev_depth_map is not None:
        low_res_coords = sub_pixel_coords / 2.0
        prev_depth_vals = bilinear_interpolate(prev_depth_map, low_res_coords.view(-1, 2)).view(h, w)
        l_depth = torch.mean(torch.abs(rendered_depths - prev_depth_vals))
        
    total_loss = (1.0 * l_ren + 1.0 * l_3d + 0.001 * l_pr + 0.0001 * l_epi + 
                  0.008 * l_ssim + 0.01 * l_ds + 1.0 * l_depth)
    return total_loss

def train_sfmnerf(images, poses, k_matrix, sift_matches):
    """Main training loop with pyramidal strategy."""
    levels = construct_pyramid(images)
    model = NeRFModel()
    prev_depth_map = None
    
    for level in reversed(range(len(levels))):
        ref_img = levels[level]
        tgt_img = levels[level] # Simplified: same image pair for demo
        optimizer = torch.optim.Adam(model.parameters(), lr=1e-3)
        
        for iteration in range(1000):
            optimizer.zero_grad()
            loss = sfmnerf_step(ref_img, tgt_img, model, k_matrix, poses[0], poses[1], 
                                sift_matches, prev_depth_map)
            loss.backward()
            optimizer.step()
            
        with torch.no_grad():
            rays_o, rays_d = get_rays_from_coords(torch.zeros(1, 1, 2, device=ref_img.device), k_matrix, poses[0])
            _, prev_depth_map = render_neurf(rays_o, rays_d, model)
            
    return model
```

## Operating characteristics
*   **Time Complexity:** The method is computationally intensive due to the additional geometric losses [S1]. The epipolar constraint requires searching for the minimum distance along the epipolar line for each feature match, which can be optimized by filtering implausible points via color similarity [S1]. The pyramidal strategy reduces the computational burden at higher resolutions by initializing from coarse estimates [S1].
*   **Memory Complexity:** Requires storing the NeRF model, intermediate depth maps for the pyramidal strategy, and SIFT feature matches [S1].
*   **Performance:** Evaluated on three public datasets, SfMNeRF surpasses state-of-the-art approaches given sparse inputs [S1]. It improves novel view synthesis quality and 3D-scene geometry reconstruction compared to standard NeRF, which suffers from inaccurate 3D structures under sparse conditions [S1].
*   **Strengths:** Explicitly constrains geometry using SfM priors, reducing shape-radiance ambiguity [S1]. Sub-pixel rendering improves generalization [S1]. Pyramidal strategy prevents local minima [S1].
*   **Limitations:** Relies on the quality of SIFT feature matches [S1]. Large occlusions or textureless regions may lack sufficient features [S1]. The epipolar constraint can be computationally expensive if not filtered properly [S1].

## Background
SfMNeRF was proposed to address the limitations of existing NeRF-based methods in obtaining moderate 3D structures under sparse input conditions, where novel view synthesis quality drops dramatically due to implicitly reconstructed inaccurate 3D-scene structures [S1]. The method leverages knowledge from self-supervised depth estimation and Structure-from-Motion (SfM) to explicitly constrain the 3D-scene geometry during view synthesis training [S1]. It was evaluated on three public datasets [S1]. The code is available at https://github.com/XTU-PR-LAB/SfMNeRF [S1].

## Sources
[S1] Improving neural radiance fields with depth-aware optimization for novel view synthesis, https://doi.org/10.1016/j.imavis.2026.106052, fetched 2026-10-26
