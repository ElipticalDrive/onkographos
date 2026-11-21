# SPARF

Also called: Neural Radiance Fields from Sparse and Noisy Poses, SPARF Joint Pose-NeRF Training Strategy, joint pose-NeRF optimization, pose-NeRF training

## Overview
SPARF is a joint pose-NeRF training strategy designed for novel view synthesis from sparse input views (as few as 3) with noisy camera pose estimates [S1]. It solves the problems of overfitting and geometric inconsistency inherent in sparse-view regimes by enforcing multi-view geometry constraints, specifically multi-view correspondence and depth consistency losses, rather than relying solely on photometric loss [S1]. Unlike sibling methods that optimize poses and NeRF weights using independent per-image photometric supervision, SPARF explicitly connects training views through pixel correspondences to drive convergence to a globally accurate pose and scene solution [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** $n$ RGB images (typically $n=3$) with large viewpoint variations and noisy camera pose estimates. No prior on scene shape is assumed [S1].
*   **Outputs:** Optimized camera poses $\hat{P}$ and NeRF parameters $\Theta$ defining the radiance field [S1].

**Data Structures and Layouts:**
*   **Camera Pose $P$:** Represented as a world-to-camera transformation matrix [S1].
    *   **Translation:** A 3D embedding vector $t \in \mathbb{R}^3$ optimized directly [S1].
    *   **Rotation:** A continuous 6D embedding vector $r \in \mathbb{R}^6$ optimized directly [S1]. The rotation matrix $R$ is recovered from $r$ using a Gram-Schmidt-like process [S1]. Given $r = [a_1^T, a_2^T]$ where $a_1, a_2 \in \mathbb{R}^3$, the columns of $R = [b_1, b_2, b_3]$ are computed as:
        $$ b_1 = \frac{a_1}{\|a_1\|} $$
        $$ b_2 = \frac{a_2 - (a_2 \cdot b_1)b_1}{\|a_2 - (a_2 \cdot b_1)b_1\|} $$
        $$ b_3 = b_1 \times b_2 $$
        [S1]
*   **NeRF Model:** A multi-layer perceptron (MLP) mapping 3D coordinates and viewing directions to color and density [S1]. The sources note that for LLFF joint optimization, a single coarse MLP suffices, reducing training time compared to hierarchical sampling [S1].
*   **Correspondences:** Dense matches $q$ for each pixel $p$ and confidence $w_p$ inferred from a pre-trained matching model (e.g., PDC-Net) [S1].

**Numbered Steps:**
1.  **Pre-compute Correspondences:** Extract dense matches between all pairs of training views using a pre-trained dense correspondence regression network (e.g., PDC-Net) [S1]. Filter matches by confidence threshold $\gamma = 0.95$ (PR operator) and optionally by cyclic consistency < 1.5 pixels [S1].
2.  **Initialize:** Initialize NeRF weights and camera poses (noisy estimates) [S1]. Set depth sampling bounds. For LLFF, use inverse depth space with lower bound $1/t_n = 1$ and upper bound $1/t_f = 0.05$ [S1]. For DTU/Replica, use metric space $[t_n, t_f]$ [S1]. If poses are noisy, expand depth range by factor $\epsilon = 0.3$, i.e., $[(1-\epsilon)t_n, (1+\epsilon)t_f]$ [S1].
3.  **Staged Training Loop:** Iterate for total $N$ iterations (typically 100K for joint optimization) [S1].
    *   **Stage 1 (Iterations $0$ to $0.3N$):** Jointly optimize camera poses $\hat{P}$ and the coarse NeRF MLP $\Theta_c$ [S1].
    *   **Stage 2 (Iterations $0.3N$ to $N$):** Freeze camera poses $\hat{P}$ [S1]. Optimize both coarse and fine NeRF MLPs ($\Theta_c, \Theta_f$) if hierarchical sampling is used [S1].
    *   **Per-Iteration Procedure:**
        1.  **Photometric Loss:** Sample $x$ random pixels from all training images. Compute rendered color $\hat{I}$ and photometric loss $\mathcal{L}_{photo}$ against ground truth $I$ [S1].
        2.  **Multi-View Correspondence Loss:** Sample an image pair $(I_i, I_j)$ and a subset of 1024 matches [S1]. For each match $(p, q)$ with confidence $w_p$:
            *   Backproject pixel $p$ in $I_i$ to 3D using rendered depth $\hat{z}(p; \Theta, \hat{P}_i)$ and pose $\hat{P}_i$ [S1].
            *   Project the 3D point into view $I_j$ using $\hat{P}_j$ to get predicted pixel $\hat{q}$ [S1].
            *   Compute re-projection error $\rho(\|q - \hat{q}\|^2)$ using Huber loss $\rho$ [S1].
            *   Accumulate $\mathcal{L}_{MVCorr} = \sum w_p \rho(\dots)$ [S1].
        3.  **Depth Consistency Loss:**
            *   Sample a training view $I_i$ with pose $\hat{P}_i$ [S1].
            *   Find the closest other training view and compute a virtual "unknown" pose $P_{un}$ by interpolating the poses of two close-by training views [S1].
            *   Sample 1024 pixels $p$ in $I_i$ [S1].
            *   Backproject $p$ to 3D point $r_{un,y}$ using $\hat{P}_i$ and rendered depth $\hat{z}(p; \Theta, \hat{P}_i)$ [S1].
            *   Project $r_{un,y}$ into the virtual view $P_{un}$ to get pixel $y$ and depth $z_y$ [S1].
            *   Compute rendered depth $\hat{z}(y; \Theta, P_{un})$ at $y$ in the virtual view [S1].
            *   Calculate visibility mask $\gamma_y = T_{un, z_y} = \exp \left( -\sum_{m'=1}^{m} \sigma_{m'} \delta_{m'} \right)$ evaluated at the virtual view ray, where $T$ is accumulated transmittance. $\gamma_y \approx 1$ if unoccluded, $\approx 0$ if occluded [S1].
            *   Compute $\mathcal{L}_{DCons} = \sum \gamma_y \rho(z_y - \hat{z}(y; \Theta, P_{un}))$ [S1].
            *   *Note:* Backpropagate through the pseudo-supervision $z_y$ and $\hat{z}$, but not through $\hat{P}_i$ [S1].
        4.  **Total Loss:** $\mathcal{L} = \mathcal{L}_{photo} + \lambda_c \mathcal{L}_{MVCorr} + \lambda_d \mathcal{L}_{DCons}$ [S1].
        5.  **Update:** Update parameters using Adam optimizer [S1].
4.  **Loss Weight Decay:** After Stage 1 (poses frozen) or from start if poses are fixed, halve $\lambda_c$ every 10,000 iterations to gradually increase photometric loss importance and correct geometry errors [S1].

**Formulas:**
*   **Volume Rendering:** $\hat{I}_{i,p} = \sum_{m=1}^{M} \alpha_m c_m$, where $\alpha_m = T_m (1 - \exp(-\sigma_m \delta_m))$ and $T_m = \exp \left( -\sum_{m'=1}^{m} \sigma_{m'} \delta_{m'} \right)$. $\delta_m = t_{m+1} - t_m$ is the distance between adjacent samples [S1].
*   **Visibility Mask:** $\gamma_y = T_{un, z_y} = \exp \left( -\sum_{m'=1}^{m} \sigma_{m'} \delta_{m'} \right)$ evaluated at the unseen view ray. $\gamma_y$ is close to 1 if no point with large density exists between camera center and 3D point, otherwise close to 0 [S1].
*   **Rotation Recovery:** $R = f(r) = [b_1, b_2, b_3]$, where $b_1, b_2, b_3$ are computed using a generalized cross product to ensure orthogonality [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim

def huber_loss(error, delta=1.0):
    abs_err = torch.abs(error)
    quadratic = torch.min(abs_err, torch.full_like(abs_err, delta))
    return 0.5 * quadratic**2 + delta * (abs_err - quadratic)

def decay_lr(init_lr, final_lr, step, total_steps):
    return init_lr * (final_lr / init_lr) ** (step / total_steps)

def construct_rays(poses, u, v, intrinsics):
    B, H, W = u.shape
    ones = torch.ones_like(u)
    pix_coords = torch.stack([u, v, ones], dim=1).unsqueeze(1)
    pix_coords = pix_coords / intrinsics[:, :3, :3]
    pix_coords = pix_coords.view(B, 3, H, W)
    origins = poses[:, :3, 3].unsqueeze(-1).unsqueeze(-1).expand_as(pix_coords)
    directions = torch.bmm(pix_coords.view(B, 3, -1), poses[:, :3, :3].transpose(1, 2)).view(B, 3, H, W)
    directions = directions / torch.norm(directions, dim=1, keepdim=True)
    return origins, directions

def volume_render(nerf_model, origins, directions, near, far, num_samples):
    B, C, H, W = origins.shape
    t_vals = torch.linspace(0., 1., num_samples, device=origins.device)
    z_vals = near * (1. - t_vals) + far * t_vals
    pts = origins.unsqueeze(1) + directions.unsqueeze(1) * z_vals.view(1, -1, 1, 1, 1)
    pts_flat = pts.view(-1, num_samples, C)
    dirs_flat = directions.unsqueeze(1).expand(-1, num_samples, -1, -1).view(-1, num_samples, C)
    sigma_flat, rgb_flat = nerf_model(pts_flat, dirs_flat)
    sigma = torch.nn.functional.softplus(sigma_flat).view(B, num_samples, H, W)
    rgb = rgb_flat.view(B, num_samples, H, W, 3)
    deltas = z_vals[1:] - z_vals[:-1]
    delta_inf = torch.tensor(1e10, device=origins.device)
    deltas = torch.cat([deltas, delta_inf])
    alpha = 1.0 - torch.exp(-sigma * deltas)
    T = torch.cumprod(1.0 - alpha + 1e-10, dim=1)
    T = torch.cat([torch.ones_like(T[:, :1]), T[:, :-1]], dim=1)
    weights = alpha * T
    color = torch.sum(weights.unsqueeze(-1) * rgb, dim=1)
    depth = torch.sum(weights * z_vals, dim=1)
    return color, depth, weights, T

def project_3d_to_pixel(pts, pose, intrinsics):
    B, C, H, W = pts.shape
    cam_pts = torch.bmm(pose[:, :3, :3], pts.view(B, 3, -1)).view(B, 3, H, W) + pose[:, :3, 3].unsqueeze(-1).unsqueeze(-1)
    pix = torch.bmm(intrinsics, cam_pts.view(B, 3, -1)).view(B, 2, H, W)
    return pix[:, 0], pix[:, 1], cam_pts[:, 2]

def interpolate_pose(p1, p2, t=0.5):
    t_un = (1 - t) * p1[:, :3, 3] + t * p2[:, :3, 3]
    r1 = p1[:, :3, :3].view(-1, 9)
    r2 = p2[:, :3, :3].view(-1, 9)
    r_un = (1 - t) * r1 + t * r2
    R_un = r_un.view(-1, 3, 3)
    b1 = R_un[:, :, 0]
    b1 = b1 / torch.norm(b1, dim=1, keepdim=True)
    b2_raw = R_un[:, :, 1] - torch.sum(b1 * R_un[:, :, 1], dim=1, keepdim=True) * b1
    b2 = b2_raw / torch.norm(b2_raw, dim=1, keepdim=True)
    b3 = torch.cross(b1, b2, dim=1)
    R_un = torch.stack([b1, b2, b3], dim=2)
    P_un = torch.zeros_like(p1)
    P_un[:, :3, :3] = R_un
    P_un[:, :3, 3] = t_un
    return P_un

def sparf_step(nerf_model, poses, images, matches, config):
    B, C, H, W = images.shape
    optimizer_nerf, optimizer_poses = config['opt_nerf'], config['opt_poses']
    nerf_model.train()
    optimizer_nerf.zero_grad()
    optimizer_poses.zero_grad()

    # 1. Photometric Loss
    idx = torch.randint(0, H * W, (B, config['num_photo']))
    u = (idx % W).float()
    v = (idx // W).float()
    o, d = construct_rays(poses, u, v, config['K'])
    c_photo, _, _, _ = volume_render(nerf_model, o, d, config['near'], config['far'], config['num_samples'])
    gt = images[:, :, v.long(), u.long()]
    loss_photo = torch.mean((c_photo - gt) ** 2)

    # 2. Multi-View Correspondence Loss
    loss_mvc = 0.0
    pair_idx = torch.randint(0, B, (2,))
    i, j = pair_idx[0], pair_idx[1]
    m_idx = torch.randint(0, matches[i].shape[0], (config['num_mvc'],))
    p_u, p_v = matches[i][m_idx, :2].T
    q_u, q_v = matches[i][m_idx, 2:4].T
    w = matches[i][m_idx, 4]
    o_i, d_i = construct_rays(poses[i:i+1], p_u, p_v, config['K'])
    _, depth_i, _, _ = volume_render(nerf_model, o_i, d_i, config['near'], config['far'], config['num_samples'])
    pts_3d = o_i + d_i * depth_i.unsqueeze(1)
    q_pred_u, q_pred_v, _ = project_3d_to_pixel(pts_3d, poses[j:j+1], config['K'])
    error = torch.norm(torch.stack([q_pred_u - q_u, q_pred_v - q_v], dim=1), dim=1)
    loss_mvc = torch.mean(w * huber_loss(error))

    # 3. Depth Consistency Loss
    loss_dc = 0.0
    i_dc = torch.randint(0, B, (1,)).item()
    k_dc = (i_dc + 1) % B
    P_un = interpolate_pose(poses[i_dc:i_dc+1], poses[k_dc:k_dc+1])
    u_dc, v_dc = torch.rand(config['num_dc']), torch.rand(config['num_dc'])
    o_dc, d_dc = construct_rays(poses[i_dc:i_dc+1], u_dc, v_dc, config['K'])
    _, depth_dc, _, _ = volume_render(nerf_model, o_dc, d_dc, config['near'], config['far'], config['num_samples'])
    pts_un = o_dc + d_dc * depth_dc.unsqueeze(1)
    y_u, y_v, z_y = project_3d_to_pixel(pts_un, P_un, config['K'])
    o_un, d_un = construct_rays(P_un, y_u, y_v, config['K'])
    _, depth_un, _, T_un = volume_render(nerf_model, o_un, d_un, config['near'], config['far'], config['num_samples'])
    t_vals = torch.linspace(0., 1., config['num_samples'], device=o_un.device)
    z_vals_un = config['near'] * (1. - t_vals) + config['far'] * t_vals
    idx_z = torch.argmin(torch.abs(z_vals_un.unsqueeze(1) - z_y.unsqueeze(0)), dim=0)
    gamma_y = T_un[0, idx_z]
    loss_dc = torch.mean(gamma_y * huber_loss(z_y - depth_un))

    # Total Loss
    lambda_c = config['lambda_c']
    lambda_d = config['lambda_d']
    total_loss = loss_photo + lambda_c * loss_mvc + lambda_d * loss_dc
    total_loss.backward()
    optimizer_nerf.step()
    optimizer_poses.step()
    return total_loss.item(), loss_photo.item(), loss_mvc.item(), loss_dc.item()
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
