# SPARF

Also called: Neural Radiance Fields from Sparse and Noisy Poses, SPARF Joint Pose-NeRF Training Strategy, joint pose-NeRF optimization, pose-NeRF training

## Overview
SPARF is a joint pose-NeRF training strategy designed for novel view synthesis from sparse input views (as few as 3) with noisy camera pose estimates [S1]. It solves the problems of overfitting and geometric inconsistency inherent in sparse-view regimes by enforcing multi-view geometry constraints, specifically multi-view correspondence and depth consistency losses, rather than relying solely on photometric loss [S1]. Unlike sibling methods that optimize poses and NeRF weights using independent per-image photometric supervision, SPARF explicitly connects training views through pixel correspondences to drive convergence to a globally accurate pose and scene solution [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** $n$ RGB images (typically $n=3$) with large viewpoint variations and noisy camera pose estimates. No prior on scene shape is assumed [S1].
*   **Outputs:** Optimized camera poses $\hat{P}$ and NeRF parameters $\Theta$ defining the radiance field [S1].

**Data Structures and Layouts:**
*   **Camera Pose $P$:** Represented as a world-to-camera transformation matrix.
    *   **Translation:** A 3D embedding vector $t \in \mathbb{R}^3$ [S1].
    *   **Rotation:** A continuous 6D embedding vector $r \in \mathbb{R}^6$. The rotation matrix $R$ is recovered from $r$ using a Gram-Schmidt-like process [S1]. Given $r = [a_1^T, a_2^T]$ where $a_1, a_2 \in \mathbb{R}^3$, the columns of $R = [b_1, b_2, b_3]$ are computed as:
        $$ b_1 = \frac{a_1}{\|a_1\|} $$
        $$ b_2 = \frac{a_2 - (a_2 \cdot b_1)b_1}{\|a_2 - (a_2 \cdot b_1)b_1\|} $$
        $$ b_3 = b_1 \times b_2 $$
        [S1]
*   **NeRF Model:** A multi-layer perceptron (MLP) mapping 3D coordinates and viewing directions to color and density. The sources note that for LLFF joint optimization, a single coarse MLP suffices, reducing training time compared to hierarchical sampling [S1].
*   **Correspondences:** Dense matches $q$ for each pixel $p$ and confidence $w_p$ inferred from a pre-trained matching model (e.g., PDC-Net) [S1].

**Numbered Steps:**

1.  **Pre-compute Correspondences:** Extract dense matches between all pairs of training views using a pre-trained dense correspondence regression network (e.g., PDC-Net) [S1]. Filter matches by confidence threshold $\gamma = 0.95$ (PR operator) and optionally by cyclic consistency < 1.5 pixels [S1].
2.  **Initialize:** Initialize NeRF weights and camera poses (noisy estimates). Set depth sampling bounds. For LLFF, use inverse depth space with lower bound $1/t_n = 1$ and upper bound $1/t_f = 0.05$. For DTU/Replica, use metric space $[t_n, t_f]$. If poses are noisy, expand depth range by factor $\epsilon = 0.3$, i.e., $[(1-\epsilon)t_n, (1+\epsilon)t_f]$ [S1].
3.  **Staged Training Loop:** Iterate for total $N$ iterations (typically 100K for joint optimization) [S1].
    *   **Stage 1 (Iterations $0$ to $0.3N$):** Jointly optimize camera poses $\hat{P}$ and the coarse NeRF MLP $\Theta_c$.
    *   **Stage 2 (Iterations $0.3N$ to $N$):** Freeze camera poses $\hat{P}$. Optimize both coarse and fine NeRF MLPs ($\Theta_c, \Theta_f$) if hierarchical sampling is used [S1].
    *   **Per-Iteration Procedure:**
        1.  **Photometric Loss:** Sample $x$ random pixels from all training images. Compute rendered color $\hat{I}$ and photometric loss $\mathcal{L}_{photo}$ against ground truth $I$ [S1].
        2.  **Multi-View Correspondence Loss:** Sample an image pair $(I_i, I_j)$ and a subset of 1024 matches. For each match $(p, q)$ with confidence $w_p$:
            *   Backproject pixel $p$ in $I_i$ to 3D using rendered depth $\hat{z}(p; \Theta, \hat{P}_i)$ and pose $\hat{P}_i$.
            *   Project the 3D point into view $I_j$ using $\hat{P}_j$ to get predicted pixel $\hat{q}$.
            *   Compute re-projection error $\rho(\|q - \hat{q}\|^2)$ using Huber loss $\rho$ [S1].
            *   Accumulate $\mathcal{L}_{MVCorr} = \sum w_p \rho(\dots)$ [S1].
        3.  **Depth Consistency Loss:**
            *   Sample a training view $I_i$ with pose $\hat{P}_i$.
            *   Find the closest other training view and compute a virtual "unknown" pose $P_{un}$ by interpolating the poses of two close-by training views [S1].
            *   Sample 1024 pixels $p$ in $I_i$.
            *   Backproject $p$ to 3D point $r_{un,y}$ using $\hat{P}_i$ and rendered depth $\hat{z}(p; \Theta, \hat{P}_i)$ [S1].
            *   Project $r_{un,y}$ into the virtual view $P_{un}$ to get pixel $y$ and depth $z_y$ [S1].
            *   Compute rendered depth $\hat{z}(y; \Theta, P_{un})$ at $y$ in the virtual view [S1].
            *   Calculate visibility mask $\gamma_y = T_{un, z_y} = \exp \left( -\sum_{m'=1}^{m} \sigma_{m'} \delta_{m'} \right)$ evaluated at the virtual view ray, where $T$ is accumulated transmittance. $\gamma_y \approx 1$ if unoccluded, $\approx 0$ if occluded [S1].
            *   Compute $\mathcal{L}_{DCons} = \sum \gamma_y \rho(z_y - \hat{z}(y; \Theta, P_{un}))$ [S1].
            *   *Note:* Backpropagate through the pseudo-supervision $z_y$ and $\hat{z}$, but not through $\hat{P}_i$ [S1].
        4.  **Total Loss:** $\mathcal{L} = \mathcal{L}_{photo} + \lambda_c \mathcal{L}_{MVCorr} + \lambda_d \mathcal{L}_{DCons}$ [S1].
        5.  **Update:** Update parameters using Adam optimizer [S1].
4.  **Loss Weight Decay:** After Stage 1 (poses frozen) or from start if poses are fixed, halve $\lambda_c$ every 10,000 iterations to gradually increase photometric loss importance and correct geometry errors [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim
import numpy as np

def gram_schmidt_6d(r):
    """
    Recover rotation matrix R from 6D vector r = [a1, a2].
    r: Tensor of shape (6,) or (batch, 6)
    Returns: R of shape (3, 3) or (batch, 3, 3)
    """
    a1 = r[..., :3]
    a2 = r[..., 3:]
    
    # Normalize a1
    b1 = a1 / torch.norm(a1, dim=-1, keepdim=True)
    
    # Orthogonalize a2 against b1
    proj = torch.sum(a2 * b1, dim=-1, keepdim=True) * b1
    b2_raw = a2 - proj
    b2 = b2_raw / torch.norm(b2_raw, dim=-1, keepdim=True)
    
    # Cross product for b3
    b3 = torch.cross(b1, b2, dim=-1)
    
    R = torch.stack([b1, b2, b3], dim=-2)
    return R

def volume_rendering(nerf_model, ray_origins, ray_directions, near, far, num_samples):
    """
    Standard volume rendering.
    nerf_model: MLP taking (xyz, direction) -> (sigma, color)
    ray_origins: (batch, 3)
    ray_directions: (batch, 3)
    near, far: scalars
    num_samples: int
    Returns: color, depth, weights
    """
    t_vals = torch.linspace(0., 1., num_samples)
    z_vals = near * (1. - t_vals) + far * t_vals
    
    # Expand dims for broadcasting
    ray_origins = ray_origins.unsqueeze(1) # (batch, 1, 3)
    ray_directions = ray_directions.unsqueeze(1) # (batch, 1, 3)
    
    pts = ray_origins + ray_directions * z_vals.unsqueeze(-1) # (batch, num_samples, 3)
    
    # Query NeRF
    pts_flat = pts.reshape(-1, 3)
    dirs_flat = ray_directions.repeat_interleave(num_samples, dim=1).reshape(-1, 3)
    
    # Assuming nerf_model returns (sigma, rgb)
    # Note: In SPARF, softplus is used on sigma
    sigma_flat, rgb_flat = nerf_model(pts_flat, dirs_flat)
    sigma = sigma_flat.reshape(pts.shape[:-1])
    rgb = rgb_flat.reshape(pts.shape[:-1, -1])
    
    # Apply softplus to sigma for stability
    sigma = torch.nn.functional.softplus(sigma)
    
    # Compute deltas
    deltas = z_vals[..., 1:] - z_vals[..., :-1]
    delta_inf = 1e10
    deltas = torch.cat([deltas, delta_inf], dim=-1) # (batch, num_samples)
    
    # Compute alpha
    alpha = 1.0 - torch.exp(-sigma * deltas) # (batch, num_samples)
    
    # Compute transmittance
    T = torch.cumprod(1.0 - alpha + 1e-10, dim=-1) # (batch, num_samples)
    T = torch.cat([torch.ones_like(T[..., :1]), T[..., :-1]], dim=-1)
    
    # Weights
    weights = alpha * T # (batch, num_samples)
    
    # Composite
    color = torch.sum(weights.unsqueeze(-1) * rgb, dim=-2)
    depth = torch.sum(weights * z_vals, dim=-2)
    
    return color, depth, weights, T

def sparf_step(nerf_model, poses, images, matches, config):
    """
    Single optimization step for SPARF.
    """
    # 1. Photometric Loss
    # Sample random pixels
    batch_size = images.shape[0]
    num_pixels = config['num_photometric_pixels'] # 1024
    indices = torch.randint(0, images.shape[1] * images.shape[2], (batch_size, num_pixels))
    u = (indices % images.shape[2]).float()
    v = (indices // images.shape[2]).float()
    
    # Construct rays
    # Assuming pinhole model: o = t, d = R * K^-1 * p_bar
    # This part depends on specific camera intrinsics K, assumed known or fixed
    ray_origins, ray_directions = construct_rays(poses, u, v, config['intrinsics'])
    
    # Render
    rendered_colors, _, _, _ = volume_rendering(
        nerf_model, ray_origins, ray_directions, 
        config['near'], config['far'], config['num_samples']
    )
    
    # Gather ground truth colors
    gt_colors = gather_pixels(images, u, v)
    
    loss_photo = torch.mean((rendered_colors - gt_colors) ** 2)
    
    # 2. Multi-View Correspondence Loss
    # Sample image pair (i, j) and 1024 matches
    # matches[i] contains list of (q, w) for pixel p in image i
    # Simplified: assume we have a batch of matches (p_i, q_j, w)
    loss_mvc = 0.0
    num_mvc_matches = 1024
    
    # Sample matches from pre-computed set
    selected_matches = sample_matches(matches, num_mvc_matches)
    
    for p_i, q_j, w in selected_matches:
        # Backproject p_i in image i to 3D
        ray_o_i, ray_d_i = construct_rays(poses[i:i+1], p_i.u, p_i.v, config['intrinsics'])
        _, depth_i, _, _ = volume_rendering(nerf_model, ray_o_i, ray_d_i, config['near'], config['far'], config['num_samples'])
        
        # Get 3D point
        # r_3d = o_i + depth_i * d_i
        r_3d = ray_o_i + ray_d_i * depth_i.unsqueeze(-1)
        
        # Project r_3d to image j
        # p_j = K * R_j * (R_i^-1 * (r_3d - t_i) + t_j) ? 
        # Actually, r_3d is in world coords. 
        # proj_j = K * (R_j^-1 * (r_3d - t_j))
        # But poses are world-to-camera. 
        # Let's assume poses[i] is W2C matrix.
        # r_cam_j = poses[j].R @ r_3d + poses[j].t
        # p_j = K @ r_cam_j
        
        # Compute reprojection error
        pred_q_j = project_point(r_3d, poses[j], config['intrinsics'])
        error = torch.norm(pred_q_j - q_j, dim=-1)
        loss_mvc += w * huber_loss(error)
        
    loss_mvc /= num_mvc_matches
    
    # 3. Depth Consistency Loss
    loss_dc = 0.0
    num_dc_pixels = 1024
    
    # Sample training view I_i
    i = torch.randint(0, batch_size, (1,)).item()
    # Find closest other view for virtual pose
    # Simplified: pick random other view k and interpolate
    k = (i + 1) % batch_size
    # Interpolate poses to get P_un
    # Linear interpolation of translation and 6D rotation
    t_un = 0.5 * poses[i].t + 0.5 * poses[k].t
    r_un = 0.5 * poses[i].r + 0.5 * poses[k].r
    R_un = gram_schmidt_6d(r_un)
    P_un = Pose(t=t_un, R=R_un)
    
    # Sample pixels in I_i
    u_dc, v_dc = sample_random_pixels(images[i].shape[1], images[i].shape[2], num_dc_pixels)
    
    # Backproject to 3D using I_i's pose
    ray_o_i, ray_d_i = construct_rays(poses[i:i+1], u_dc, v_dc, config['intrinsics'])
    _, depth_i, _, _ = volume_rendering(nerf_model, ray_o_i, ray_d_i, config['near'], config['far'], config['num_samples'])
    r_3d = ray_o_i + ray_d_i * depth_i.unsqueeze(-1)
    
    # Project to virtual view P_un
    y, z_y = project_point_to_view(r_3d, P_un, config['intrinsics'])
    
    # Render depth at y in virtual view
    # Need to construct rays for virtual view at y
    ray_o_un, ray_d_un = construct_rays_from_pixel(P_un, y, config['intrinsics'])
    _, depth_un, _, transmittance_un = volume_rendering(nerf_model, ray_o_un, ray_d_un, config['near'], config['far'], config['num_samples'])
    
    # Visibility mask gamma_y
    # gamma_y = T_un at depth z_y
    # This requires finding the transmittance at the specific depth z_y along the virtual ray
    # Simplified: use the transmittance at the sample closest to z_y or interpolate
    # In SPARF, gamma_y = exp(-sum sigma * delta) up to z_y
    # We approximate this by checking if the ray is occluded before z_y
    # The source says gamma_y is close to 1 if no point with large density exists between camera center and 3D point
    # We can compute this by rendering the virtual ray and checking accumulated transmittance at z_y
    # However, volume_rendering returns T at each sample. We need T at z_y.
    # Let's assume we can get T at z_y from the rendering process
    gamma_y = get_transmittance_at_depth(transmittance_un, z_y, ray_o_un, ray_d_un)
    
    # Compute loss
    # rho(z_y - depth_un)
    error_dc = z_y - depth_un
    loss_dc += gamma_y * huber_loss(error_dc)
    
    loss_dc /= num_dc_pixels
    
    # Total Loss
    lambda_c = config['lambda_c']
    lambda_d = config['lambda_d']
    
    total_loss = loss_photo + lambda_c * loss_mvc + lambda_d * loss_dc
    
    return total_loss, loss_photo, loss_mvc, loss_dc

def sparf_train(nerf_model, poses, images, matches, config):
    optimizer_nerf = optim.Adam(nerf_model.parameters(), lr=config['lr_nerf_init'])
    optimizer_poses = optim.Adam(poses.parameters(), lr=config['lr_pose_init'])
    
    total_iters = config['total_iters'] # 100K
    stage1_iters = int(0.3 * total_iters)
    
    for step in range(total_iters):
        # Decay LR
        optimizer_nerf.param_groups[0]['lr'] = decay_lr(config['lr_nerf_init'], config['lr_nerf_final'], step, total_iters)
        optimizer_poses.param_groups[0]['lr'] = decay_lr(config['lr_pose_init'], config['lr_pose_final'], step, total_iters)
        
        # Decay lambda_c
        if step > stage1_iters or config['poses_fixed']:
            if step % 10000 == 0:
                config['lambda_c'] /= 2.0
        
        # Forward pass
        loss, l_photo, l_mvc, l_dc = sparf_step(nerf_model, poses, images, matches, config)
        
        # Backward pass
        optimizer_nerf.zero_grad()
        optimizer_poses.zero_grad()
        
        loss.backward()
        
        optimizer_nerf.step()
        optimizer_poses.step()
        
        # Freeze poses in Stage 2
        if step >= stage1_iters:
            # Detach poses from graph or stop gradient
            # In PyTorch, this is done by not passing them to the optimizer or using .detach()
            # Here we assume optimizer_poses is not called or poses are detached
            pass
            
    return nerf_model, poses
```

**Parameters:**
*   **Training Duration:** 100K iterations for joint pose-NeRF refinement [S1].
*   **Hardware:** ~10 hours on a single A100 GPU [S1].
*   **Image Resolution:** Resized to 360×600 [S1].
*   **Samples per Step:** 1024 pixel rays for photometric loss, 1024 matches for correspondence loss, 1024 pixels for depth consistency loss [S1].
*   **Loss Weights:** $\lambda_c = 10^{-3}$, $\lambda_d = 10^{-3}$ [S1]. On DTU with fixed GT poses, $\lambda_c = 10^{-4}$, $\lambda_d = 10^{-3}$ [S1].
*   **Optimizer:** Adam. NeRF LR: $5 \times 10^{-4}$ decaying to $1 \times 10^{-4}$. Pose LR: $1 \times 10^{-3}$ decaying to $1 \times 10^{-4}$ [S1].
*   **Positional Encoding:** Coarse-to-fine schedule. Joint optimization: frequency width linearly adjusted from 40% to 70% of iterations. Fixed poses: 10% to 50% [S1].
*   **Depth Bounds:** LLFF: inverse depth $[1, 0.05]$. DTU/Replica: metric $[t_n, t_f]$. Noisy poses: expand by $\epsilon=0.3$ [S1].
*   **Correspondence Filter:** Confidence $\gamma = 0.95$, cyclic consistency < 1.5 pixels [S1].

**Edge Cases and Pitfalls:**
*   **Overfitting:** Direct photometric loss causes overfitting to training views, especially with noisy poses. Solved by using multi-view correspondence as the main signal [S1].
*   **Occlusion:** Depth consistency loss is invalid for occluded pixels or those projected outside the virtual view. Handled by visibility mask $\gamma_y$ [S1].
*   **Training Length:** Training longer than necessary with fixed GT poses leads to overfitting. Recommended limits: 50K/70K iterations for 3 views (DTU/Replica/LLFF respectively), 100K/140K for 6 views, 150K/200K for 9 views [S1].
*   **Degenerate Geometry:** High-frequency positional encodings with standard NeRF on few views lead to degenerate geometry. Coarse-to-fine PE prevents this [S1].

## Subtopics

### Camera Pose and Parameter Optimization
General framework for optimizing poses and NeRF parameters jointly. SPARF is a specific instance tailored for sparse views. [Camera Pose and Parameter Optimization](7611-camera-pose-and-parameter-optimization.md)

### MELON
Uses local optimization and multi-scale features for pose refinement. SPARF uses global multi-view geometry constraints. [MELON](11909-melon.md)

### Time-dependent NeRF with Continuous Camera Motion
Models continuous camera motion over time. SPARF assumes static scene with discrete poses. [Time-dependent NeRF with Continuous Camera Motion](12399-time-dependent-nerf-with-continuous-camera-motion.md)

### Camera Preconditioning
Uses preconditioning to stabilize pose optimization. SPARF uses staged training and specific loss functions. [Camera Preconditioning](12545-camera-preconditioning.md)

### VMRF (View Matching Neural Radiance Fields)
Uses view matching losses. SPARF uses multi-view correspondence and depth consistency. [VMRF (View Matching Neural Radiance Fields)](12630-vmrf-view-matching-neural-radiance-fields.md)

### NeRF Pose and Appearance Optimization
Optimizes poses and appearance factors. SPARF focuses on geometry and pose. [NeRF Pose and Appearance Optimization](13113-nerf-pose-and-appearance-optimization.md)

### Test-Time Two-Stage Optimization
Optimizes poses at test time. SPARF optimizes during training. [Test-Time Two-Stage Optimization](13119-test-time-two-stage-optimization.md)

## Operating characteristics
*   **Time Complexity:** 100K iterations, ~10 hours on A100 GPU [S1].
*   **Memory Complexity:** Standard NeRF memory usage plus storage for pre-computed correspondences [S1].
*   **Performance:**
    *   **DTU (3 views, noisy poses):** PSNR 17.74, SSIM 0.71, LPIPS 0.26, Rot 1.81°, Trans 5.0×100. Outperforms BARF (PSNR 10.71) and SCNeRF (PSNR 12.04) [S1].
    *   **LLFF (3 views, identity poses):** PSNR 19.58, SSIM 0.61, LPIPS 0.31, Rot 0.53°, Trans 2.8×100. Outperforms BARF (PSNR 17.47) and SCNeRF (PSNR 17.10) [S1].
    *   **Replica (3 views, COLMAP poses):** PSNR 26.98, SSIM 0.88, LPIPS 0.13, Rot 0.15°, Trans 0.36×100. Outperforms DS-NeRF (PSNR 24.75) and SCNeRF (PSNR 22.54) [S1].
*   **Strengths:** Handles sparse views (3 images) and noisy poses effectively. Prevents overfitting through multi-view constraints.
*   **Limitations:** Requires pre-computed dense correspondences. Performance degrades if correspondences are inaccurate. Limited to static scenes.
*   **When to Use:** Sparse-view novel view synthesis with noisy initial poses.
*   **When Not to Use:** Dense-view settings (other methods may be more efficient), dynamic scenes, or when high-quality initial poses are available (fixed pose optimization may suffice).

## Background
SPARF was introduced in the paper "SPARF: Neural Radiance Fields from Sparse and Noisy Poses" [S1]. It addresses the challenge of novel view synthesis in the sparse-view regime, where traditional NeRF methods overfit and fail to converge to a globally consistent geometry. The method builds on principles of multi-view geometry, specifically using pixel correspondences and depth consistency to constrain the optimization. It was evaluated on DTU, LLFF, and Replica datasets, setting new state-of-the-art results for few-view synthesis with noisy poses [S1].

## Sources
[S1] SPARF: Neural Radiance Fields from Sparse and Noisy Poses, https://doi.org/10.1109/cvpr52729.2023.00408, fetched 2026-10-26
