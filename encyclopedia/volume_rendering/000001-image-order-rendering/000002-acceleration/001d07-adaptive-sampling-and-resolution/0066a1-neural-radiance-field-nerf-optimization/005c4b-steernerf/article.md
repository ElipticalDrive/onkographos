# SteerNeRF

Also called: SmooTh viEwpoint trajEctoRy

## Overview
SteerNeRF accelerates Neural Radiance Field (NeRF) rendering for interactive applications by exploiting the smoothness of continuous viewpoint trajectories to reduce the computational cost of volume rendering [S1]. Unlike methods that rely on high-precision motion vectors from game engines or static geometry, SteerNeRF uses a low-resolution volume-rendered feature map combined with a lightweight 2D neural renderer to reconstruct high-resolution images, achieving near real-time frame rates with significantly lower memory usage than cache-based acceleration methods [S1].

## How it works
SteerNeRF operates in a pipeline consisting of a rendering buffer, neural feature fields, and a 2D neural renderer [S1]. The method reduces the number of sampled points along rays by leveraging depth information from preceding frames and recovers high-resolution detail using a convolutional neural network [S1].

### Inputs and Outputs
*   **Inputs:**
    *   Current camera pose $\xi_t$ and intrinsics $K_h$ (high resolution) or $K_l$ (low resolution) [S1].
    *   Rendering buffer containing low-resolution feature maps $\{F_{t-L}, \dots, F_{t-1}\}$ and depth maps $\{D_{t-L}, \dots, D_{t-1}\}$ from the previous $L$ frames [S1].
    *   Target image resolution $H \times W$ [S1].
*   **Outputs:**
    *   High-resolution color image $I_t \in \mathbb{R}^{H \times W \times 3}$ [S1].

### Data Structures
1.  **Neural Feature Fields ($f_\theta$):**
    *   **Input:** 3D point $x \in \mathbb{R}^3$, viewing direction $d \in S^2$ [S1].
    *   **Output:** Density $\sigma \in \mathbb{R}^+$, feature vector $f \in \mathbb{R}^K$ [S1].
    *   **Implementation:** Based on Instant-NGP [S1]. Uses multi-resolution hash tables with fixed length $2^{19}$ per resolution [S1].
2.  **Rendering Buffer:**
    *   Stores a queue of $L$ preceding frames [S1].
    *   Each frame contains:
        *   Low-resolution feature map $F \in \mathbb{R}^{\frac{H}{4} \times \frac{W}{4} \times K}$ [S1].
        *   Low-resolution depth map $D \in \mathbb{R}^{\frac{H}{4} \times \frac{W}{4}}$ [S1].
3.  **2D Neural Renderer ($g_\theta$):**
    *   **Input:** Concatenation of upsampled current feature map $\hat{F}_t$ and reprojected high-resolution feature maps $\{\hat{F}_{t_0 \to t}\}$ for $t_0 \in \{t-L, \dots, t-1\}$ [S1].
    *   **Output:** High-resolution image $I_t \in \mathbb{R}^{H \times W \times 3}$ [S1].
    *   **Architecture:** Lightweight 2D convolutional neural network. Compared to a traditional U-Net, it reduces convolution layers for high-resolution features and increases depth for low-resolution features to minimize inference time [S1]. Optimized via NVIDIA TensorRT [S1].

### Parameters
*   $K$: Number of feature channels. Typical value: 3 (RGB) or higher [S1].
*   $L$: Number of preceding frames in the buffer [S1].
*   $\epsilon$: Depth interval margin for sampling range reduction [S1].
*   Resolution: Low-resolution rendering is performed at $\frac{H}{4} \times \frac{W}{4}$ [S1].

### Algorithm Steps

1.  **Pre-training Neural Feature Fields:**
    *   Render a high-resolution feature map $\tilde{F} \in \mathbb{R}^{H \times W \times K}$ at target resolution [S1].
    *   Apply L2 reconstruction loss on the first 3 channels against the ground truth image [S1].
    *   Leave other $K-3$ channels unsupervised [S1].

2.  **Low-Resolution Volume Rendering (Current Frame $t$):**
    *   For each pixel $(u, v)$ in the low-resolution grid ($\frac{H}{4} \times \frac{W}{4}$):
        *   Cast ray $r$ [S1].
        *   **Buffer-Guided Sampling Range Reduction:**
            *   Fetch depth map $D_{t_0}$ from the buffer (where $t_0$ is a preceding frame, typically $t-1$) [S1].
            *   Unproject $D_{t_0}$ to 3D points $p$:
                $$ p = \xi_{t_0}^{-1} K_l^{-1} d_{t_0} [u, v, 1]^T $$
                where $d_{t_0}$ is the depth value at $(u,v)$ in $D_{t_0}$ [S1].
            *   Project $p$ to the current high-resolution view to get reprojected depth coordinates:
                $$ d_{t_0 \to t} [\hat{u}_{t_0 \to t}, \hat{v}_{t_0 \to t}, 1]^T = K_h \xi_t p $$
            *   The sampling range for the current ray is limited to $[d_{t_0 \to t} - \epsilon, d_{t_0 \to t} + \epsilon]$ [S1].
        *   Sample points within the reduced range [S1].
        *   Query $f_\theta(x, d)$ to get $\sigma$ and $f$ [S1].
        *   Compute feature vector $f_r$ via volume rendering integral:
            $$ f_r = \sum_{i=1}^{N} T_i \alpha_i f_i $$
            where $\alpha_i = 1 - \exp(-\sigma_i \delta_i)$ and $T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)$ [S1].
        *   Store $f_r$ in $F_t$ and depth $d_r$ in $D_t$ [S1].
    *   Store $F_t$ and $D_t$ in the rendering buffer [S1].

3.  **Reprojection of Preceding Features:**
    *   For each preceding frame $t_0 \in \{t-L, \dots, t-1\}$:
        *   Unproject $D_{t_0}$ to 3D points $p$ as in Step 2 [S1].
        *   Project $p$ to high-resolution coordinates using $K_h$ and $\xi_t$ [S1].
        *   Map the low-resolution feature $F_{t_0}(u, v)$ to the high-resolution grid $\hat{F}_{t_0 \to t}$ using rounded coordinates:
            $$ \hat{F}_{t_0 \to t}(\lfloor \hat{u}_{t_0 \to t} \rfloor, \lfloor \hat{v}_{t_0 \to t} \rfloor) = F_{t_0}(u, v) $$
            Rounding has negligible impact on performance [S1].

4.  **High-Resolution Reconstruction:**
    *   Upsample the current low-resolution feature map $F_t$ to high resolution $\hat{F}_t$ (e.g., bilinear interpolation) [S1].
    *   Concatenate $\hat{F}_t$ and $\{\hat{F}_{t_0 \to t}\}$ along the channel dimension [S1].
    *   Pass the concatenated tensor through $g_\theta$ to generate $I_t$ [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def steernrf_render(current_pose, current_intr_h, current_intr_l,
                    prev_features, prev_depths, prev_poses,
                    feature_fields, renderer, buffer, H, W, K, L, epsilon):
    """
    current_pose: (4, 4) camera-to-world matrix for frame t
    current_intr_h: (3, 3) high-res intrinsic matrix K_h
    current_intr_l: (3, 3) low-res intrinsic matrix K_l
    prev_features: List of (H_l, W_l, K) tensors
    prev_depths: List of (H_l, W_l) tensors
    prev_poses: List of (4, 4) camera-to-world matrices for t-L to t-1
    feature_fields: Instant-NGP style MLP returning (sigma, f)
    renderer: 2D CNN returning (H, W, 3) image
    buffer: RenderingBuffer instance
    H, W: Target resolution
    K: Feature channels
    L: Buffer capacity
    epsilon: Depth margin
    """
    H_l, W_l = H // 4, W // 4
    
    # 1. Low-Res Volume Rendering
    # Standard ray generation from camera intrinsics and pose [G]
    origins, directions = generate_rays(current_pose, current_intr_l, H_l, W_l)
    
    # Buffer-guided sampling range reduction using most recent frame
    if len(prev_depths) > 0:
        D_prev = prev_depths[-1]
        prev_pose = prev_poses[-1]
        
        # Unproject depth to 3D points
        u_l, v_l = torch.meshgrid(torch.arange(W_l), torch.arange(H_l), indexing='ij')
        uv1 = torch.stack([u_l.float(), v_l.float(), torch.ones_like(u_l)], dim=-1) # (H_l, W_l, 3)
        uv1 = uv1.view(-1, 3)
        depth_vals = D_prev.view(-1, 1)
        uv1_scaled = uv1 * depth_vals
        
        # p = inv(K_l) @ uv1_scaled, then transform to world: p_world = inv(P_prev) @ [p, 1]
        p_cam = torch.inverse(current_intr_l) @ uv1_scaled.T # (3, N)
        p_world = torch.cat([p_cam, torch.ones(1, p_cam.shape[1])], dim=0)
        p_world = torch.inverse(prev_pose) @ p_world
        p_world = p_world[:3].T # (N, 3)
        
        # Project to current high-res view
        p_curr_cam = current_pose @ torch.cat([p_world, torch.ones(p_world.shape[0], 1)], dim=1).T
        proj = current_intr_h @ p_curr_cam[:3]
        uv_hat = proj[:2] / proj[2] # (2, N)
        
        # Compute sampling bounds
        depth_proj = proj[2]
        t_min = depth_proj - epsilon
        t_max = depth_proj + epsilon
        
        # Apply bounds to ray origins/directions for volume rendering
        origins = origins.view(-1, 3)
        directions = directions.view(-1, 3)
        # Adjust ray segments to [t_min, t_max] [G]
    else:
        t_min, t_max = 0.0, 100.0
        
    # Volume rendering integral
    # Standard alpha compositing loop over N samples [G]
    F_t, D_t = volume_render_integral(origins, directions, t_min, t_max, feature_fields, K)
    
    # 2. Reproject Preceding Features
    high_res_features = []
    
    # Upsample current feature map
    F_t_up = F.interpolate(F_t.permute(2, 0, 1).unsqueeze(0), size=(H, W), mode='bilinear', align_corners=False).squeeze(0).permute(1, 2, 0) # (H, W, K)
    
    # Reproject previous frames
    for i in range(len(prev_features)):
        F_prev = prev_features[i]
        D_prev = prev_depths[i]
        prev_pose_i = prev_poses[i]
        
        # Unproject D_prev to 3D points
        uv1 = torch.stack([u_l.float(), v_l.float(), torch.ones_like(u_l)], dim=-1).view(-1, 3)
        depth_vals = D_prev.view(-1, 1)
        uv1_scaled = uv1 * depth_vals
        p_cam = torch.inverse(current_intr_l) @ uv1_scaled.T
        p_world = torch.cat([p_cam, torch.ones(1, p_cam.shape[1])], dim=0)
        p_world = torch.inverse(prev_pose_i) @ p_world
        p_world = p_world[:3].T
        
        # Project to current high-res
        p_curr_cam = current_pose @ torch.cat([p_world, torch.ones(p_world.shape[0], 1)], dim=1).T
        proj = current_intr_h @ p_curr_cam[:3]
        uv_hat = proj[:2] / proj[2]
        
        # Round coordinates and gather features
        u_int = torch.clamp(uv_hat[0].long(), 0, W-1)
        v_int = torch.clamp(uv_hat[1].long(), 0, H-1)
        # Map features using rounded coordinates
        F_hat_prev = F_prev.view(-1, K)[v_int * W_l + u_int].view(H, W, K)
        high_res_features.append(F_hat_prev)
        
    # Concatenate features
    concat_input = torch.cat(high_res_features + [F_t_up], dim=-1)
    
    # 3. Neural Rendering
    concat_input_tensor = concat_input.permute(2, 0, 1).unsqueeze(0)
    I_t = renderer(concat_input_tensor)
    
    # Update Buffer
    buffer.push(F_t, D_t)
    
    return I_t

def volume_render_integral(origins, directions, t_min, t_max, feature_fields, K):
    # Standard volume rendering with N samples [G]
    N = 64
    t_vals = torch.linspace(t_min, t_max, N)
    deltas = t_vals[1:] - t_vals[:-1]
    deltas = torch.cat([deltas, torch.tensor([1e10], device=t_vals.device)])
    
    x = origins.unsqueeze(1) + directions.unsqueeze(1) * t_vals.unsqueeze(2)
    sigma, f = feature_fields(x.view(-1, 3), directions.unsqueeze(1).expand(-1, N, -1).view(-1, 3))
    sigma = sigma.view(-1, N)
    f = f.view(-1, N, K)
    
    alpha = 1.0 - torch.exp(-sigma * deltas)
    T = torch.cumprod(torch.cat([torch.ones_like(alpha[:, :1]), 1.0 - alpha[:, :-1]], dim=1), dim=1)
    
    weights = alpha * T
    F = torch.sum(weights.unsqueeze(2) * f, dim=1)
    D = torch.sum(weights * t_vals, dim=1)
    
    return F, D

def generate_rays(pose, intr, H, W):
    # Standard ray generation from camera pose and intrinsics [G]
    u, v = torch.meshgrid(torch.arange(W), torch.arange(H), indexing='ij')
    uv1 = torch.stack([u.float(), v.float(), torch.ones_like(u)], dim=-1)
    uv1 = uv1.view(-1, 3)
    cam_dirs = torch.inverse(intr) @ uv1.T
    cam_dirs = cam_dirs / torch.norm(cam_dirs, dim=0, keepdim=True)
    world_dirs = pose[:3, :3] @ cam_dirs
    origins = pose[:3, 3].unsqueeze(1).expand(-1, H*W)
    return origins.T, world_dirs.T
```

### Edge Cases and Pitfalls
*   **Overfitting:** If the number of training views is small, the 2D neural renderer may overfit [S1]. This is mitigated by distillation: using a pre-trained NeRF to synthesize pseudo ground truth viewpoints by randomly sampling within the available viewing zone [S1].
*   **Coordinate Rounding:** Simply rounding the reprojected coordinates $(\hat{u}, \hat{v})$ has a negligible impact on performance and is used for efficiency [S1].
*   **No Perfect Motion Vectors:** Unlike game engines, NeRF does not provide high-precision motion vectors [S1]. The method relies on noisy depth maps from volume rendering, which requires the joint training of feature fields and the renderer to hallucinate high-fidelity details [S1].

## Operating characteristics
*   **Speed:** Achieves nearly 100 FPS at $800 \times 800$ resolution and 30 FPS at $1920 \times 1080$ resolution [S1]. On the Tanks & Temples dataset, it achieves 75.19 FPS (FP16) [S1].
*   **Memory:** Memory usage is less than 20% of KiloNeRF and comparable to Instant-NGP (29.4 MB on Tanks & Temples vs 25.2 MB for Instant-NGP) [S1].
*   **Quality:** Competitive with other methods. On NeRF-Synthetic, it achieves 31.60 PSNR, 0.954 SSIM, and 0.058 LPIPS [S1].
*   **Scaling:** Performance scales with the efficiency of the 2D neural renderer optimization (e.g., TensorRT FP16/INT8) [S1].
*   **Strengths:** Low memory footprint, high frame rates, suitable for smooth viewpoint trajectories [S1].
*   **Limitations:** Relies on smooth camera motion; performance may degrade with abrupt viewpoint changes or sparse training views without distillation [S1].

## Background
SteerNeRF was introduced in 2023 by authors exploring the efficiency-memory trade-off in NeRF rendering [S1]. It is inspired by super-sampling methods for real-time rendering (e.g., DLSS) but adapts them to the novel view synthesis (NVS) task where perfect geometry and motion vectors are unavailable [S1]. It builds on Instant-NGP for efficient feature field representation [S1].

## Sources
[S1] SteerNeRF: Accelerating NeRF Rendering via Smooth Viewpoint Trajectory, https://doi.org/10.1109/cvpr52729.2023.01983, fetched 2026-10-26
[S2] Neural Radiance Field-based Visual Rendering: a Comprehensive Review, https://ieeexplore.ieee.org/abstract/document/11455380, fetched 2026-10-30
