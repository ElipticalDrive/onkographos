# Neural RGB-D Surface Reconstruction

Also called: Hybrid Scene Representation Network, Neural RGB-D

## Overview
Neural RGB-D Surface Reconstruction is a method for high-quality, metric 3D reconstruction of room-scale scenes from an RGB-D input sequence [S1]. It solves the problem of generating clean, watertight meshes from noisy depth data by replacing the standard density-based volumetric representation of NeRF with a hybrid representation: a truncated signed distance field (TSDF) for geometry and a volumetric radiance field for appearance [S1]. Unlike standard NeRF which produces semi-transparent artifacts and lacks metric geometry, this method enforces hard boundaries between occupied and free space, enabling direct mesh extraction via Marching Cubes [S1]. It builds on [Specific Model Architectures](8585-specific-model-architectures.md) by integrating geometric priors directly into the rendering integral rather than as auxiliary losses alone [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** $N$ color frames $I_i$, corresponding aligned depth frames $D_i$, and initial camera poses $T_i$ [S1].
*   **Outputs:** Optimized MLP parameters, camera poses, and a triangle mesh extracted from the final TSDF [S1].

**Data Structures:**
1.  **Shape MLP:** A multi-layer perceptron [S1].
    *   Input: Sinusoidal positional encoding $\gamma(\mathbf{p})$ of a 3D query point $\mathbf{p}$ [S1].
    *   Output: Truncated signed distance value $D_i$ [S1].
2.  **Radiance MLP:** A multi-layer perceptron [S1].
    *   Inputs: Sinusoidal positional encoding $\gamma(\mathbf{d})$ of the viewing direction $\mathbf{d}$, an intermediate feature vector from the Shape MLP, and a per-frame latent corrective code vector $\mathbf{z}_i$ [S1].
    *   Output: View-dependent surface radiance $\mathbf{c}_i$ [S1].
3.  **Deformation Field:** A 6-layer ReLU MLP acting as a residual correction for pixel locations [S1].
4.  **Camera Poses:** Represented as Euler angles and a translation vector for each frame [S1].
5.  **Scene Scaling:** The scene is scaled such that the truncation region maps to $[-1, 1]$, where positive values are in front of the surface and negative values are behind [S1].

**Parameters:**
*   $tr$: Truncation distance. Typical value: $5$ cm [S1].
*   $\lambda_1$: RGB loss weight. Value: $0.1$ [S1].
*   $\lambda_2$: Free-space loss weight. Value: $10$ [S1].
*   $\lambda_3$: Truncation loss weight. Value: $6 \times 10^3$ [S1].
*   $S'_f$: Number of fine samples. Value: $16$ [S1].
*   $S'_c$: Coarse sampling rate. One sample for every $1.5$ cm of ray length on average [S1].
*   Ray length: Must exceed the largest distance in the scene. Typical range: $4$ to $8$ meters [S1].
*   Optimizer: ADAM with learning rate $5 \times 10^{-4}$ [S1].
*   Training iterations: $2 \times 10^5$ [S1].
*   Batch size: $|P_b| = 1024$ randomly chosen rays per iteration [S1].

**Numbered Steps:**
1.  **Initialization:** Initialize camera poses $T_i$ using BundleFusion [S1]. Initialize MLP weights [S1].
2.  **Ray Generation & Refinement:** For a pixel $p$ in batch $b$:
    a. Retrieve the 2D pixel coordinate [S1].
    b. Apply the deformation field MLP to get a 2D shift vector [S1].
    c. Add the shift to the pixel coordinate [S1].
    d. Unproject the corrected 2D coordinate into a 3D ray using the camera pose $T_i$ and intrinsics [S1].
3.  **Coarse Sampling:** Generate $S'_c$ sample points along the ray using stratified sampling [S1].
4.  **Coarse Evaluation:** Evaluate the Shape MLP at these $S'_c$ points to get signed distance values [S1].
5.  **Zero-Crossing Search:** Search for the zero-crossing (where $D_i$ changes sign) in the coarse samples to estimate the surface depth [S1].
    *   *Pitfall:* If the sampling rate $S'_c$ is too low, samples may miss the truncation region, causing the zero-crossing to be missed [S1].
6.  **Fine Sampling:** Generate $S'_f$ sample points clustered around the identified zero-crossing [S1].
7.  **Fine Evaluation:** Evaluate the Shape MLP and Radiance MLP at all $S'_c + S'_f$ samples (concatenating outputs from coarse and fine passes) [S1].
8.  **Weight Calculation:** For each sample $i$ with signed distance $D_i$, compute the weight $w_i$:
    $$ w_i = \sigma \left( \frac{D_i}{tr} \right) \cdot \sigma \left( -\frac{D_i}{tr} \right) $$
    where $\sigma$ is the sigmoid function [S1]. Weights fall to zero as distance from the surface increases [S1]. To account for multiple intersections, weights of samples beyond the first truncation region are set to zero [S1].
9.  **Color Integration:** Compute the rendered color $\mathbf{C}$ as a weighted sum:
    $$ \mathbf{C} = \frac{1}{\sum_{i=0}^{K-1} w_i} \sum_{i=0}^{K-1} w_i \cdot \mathbf{c}_i $$
    where $K = S'_c + S'_f$ is the total number of samples, and $\mathbf{c}_i$ is the radiance output from the Radiance MLP [S1].
10. **Loss Computation:** Compute the total loss $\mathcal{L}(\mathcal{P})$ for the batch:
    $$ \mathcal{L}(\mathcal{P}) = \sum_{b=0}^{B-1} \lambda_1 \mathcal{L}_{rgb}^b(\mathcal{P}) + \lambda_2 \mathcal{L}_{fs}^b(\mathcal{P}) + \lambda_3 \mathcal{L}_{tr}^b(\mathcal{P}) $$
    *   **RGB Loss:** $\mathcal{L}_{rgb}^b(\mathcal{P}) = \frac{1}{|P_b|} \sum_{p \in P_b} (\mathbf{C}_p - \hat{\mathbf{C}}_p)^2$, where $\hat{\mathbf{C}}_p$ is the observed color [S1].
    *   **Free-Space Loss:** For samples $s$ between the camera origin and the truncation region ($\mathcal{S}_p^{fs}$):
        $$ \mathcal{L}_{fs}^b(\mathcal{P}) = \frac{1}{|P_b|} \sum_{p \in P_b} \frac{1}{|\mathcal{S}_p^{fs}|} \sum_{s \in \mathcal{S}_p^{fs}} (D_s - tr)^2 $$
        This forces the MLP to predict $tr$ for free space [S1].
    *   **Truncation Loss:** For samples $s$ within the truncation region ($\mathcal{S}_p^{tr}$):
        $$ \mathcal{L}_{tr}^b(\mathcal{P}) = \frac{1}{|P_b|} \sum_{p \in P_b} \frac{1}{|\mathcal{S}_p^{tr}|} \sum_{s \in \mathcal{S}_p^{tr}} (D_s - \hat{D}_s)^2 $$
        where $\hat{D}_s$ is the signed distance observed by the depth sensor along the optical axis [S1].
11. **Optimization:** Update network parameters $\Theta$, camera poses $T_i$, latent codes, and deformation field using ADAM [S1].
12. **Mesh Extraction:** After training, extract a triangle mesh from the optimized TSDF using Marching Cubes with a spatial resolution of $1$ cm [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class ShapeMLP(nn.Module):
    def __init__(self, input_dim=3, hidden_dim=256, output_dim=1):
        super().__init__()
        self.layers = nn.ModuleList([nn.Linear(input_dim, hidden_dim)] + 
                                    [nn.Linear(hidden_dim, hidden_dim) for _ in range(4)])
        self.out = nn.Linear(hidden_dim, output_dim)
        
    def forward(self, x):
        h = x
        for layer in self.layers:
            h = F.relu(layer(h))
        return self.out(h), h  # Return D and intermediate feature

class RadianceMLP(nn.Module):
    def __init__(self, feature_dim=256, view_dim=3, latent_dim=16, output_dim=3):
        super().__init__()
        self.layers = nn.ModuleList([nn.Linear(feature_dim + view_dim + latent_dim, 256)] + 
                                    [nn.Linear(256, 256) for _ in range(4)])
        self.out = nn.Linear(256, output_dim)
        
    def forward(self, feature, view_dir, latent_code):
        x = torch.cat([feature, view_dir, latent_code], dim=-1)
        h = x
        for layer in self.layers:
            h = F.relu(layer(h))
        return self.out(h)

class DeformationField(nn.Module):
    def __init__(self):
        super().__init__()
        self.layers = nn.ModuleList([nn.Linear(2, 64)] + 
                                    [nn.Linear(64, 64) for _ in range(5)])
        self.out = nn.Linear(64, 2)
        
    def forward(self, x):
        h = x
        for layer in self.layers:
            h = F.relu(layer(h))
        return self.out(h)

def positional_encoding(x, num_freq_bands=10):
    freq_bands = 2.0 ** torch.linspace(0, num_freq_bands - 1, num_freq_bands).to(x.device)
    out = [x]
    for freq in freq_bands:
        out.append(torch.sin(x * freq))
        out.append(torch.cos(x * freq))
    return torch.cat(out, dim=-1)

def unproject(coord_2d, pose, intrinsics):
    # Standard pinhole unprojection: 2D coord -> 3D ray origin & direction
    # coord_2d: [2], pose: [4x4], intrinsics: [3x3]
    fx, fy = intrinsics[0, 0], intrinsics[1, 1]
    cx, cy = intrinsics[0, 2], intrinsics[1, 2]
    x, y = coord_2d[0], coord_2d[1]
    # Direction in camera space
    dir_cam = torch.tensor([(x - cx) / fx, (y - cy) / fy, 1.0], dtype=torch.float32)
    dir_cam = dir_cam / torch.norm(dir_cam)
    # Transform to world space
    R = pose[:3, :3]
    t = pose[:3, 3]
    dir_world = R @ dir_cam
    origin_world = -R.T @ t
    return origin_world, dir_world

def get_observed_signed_distance(point_3d, pose, intrinsics, depth_map):
    # Project 3D point to 2D, sample depth, compute signed distance
    R = pose[:3, :3]
    t = pose[:3, 3]
    point_cam = R.T @ (point_3d - t)
    x = point_cam[0] / point_cam[2] * intrinsics[0, 0] + intrinsics[0, 2]
    y = point_cam[1] / point_cam[2] * intrinsics[1, 1] + intrinsics[1, 2]
    # Sample depth (nearest neighbor for simplicity)
    u, v = int(x), int(y)
    depth_val = depth_map[v, u]
    # Signed distance: positive in front, negative behind
    return point_cam[2] - depth_val

def render_ray(shape_mlp, radiance_mlp, deformation_mlp, pose, intrinsics, 
               pixel_coord, latent_code, tr, S_c, S_f, ray_length):
    # 1. Refine Ray
    shift = deformation_mlp(pixel_coord)
    refined_coord = pixel_coord + shift
    ray_origin, ray_direction = unproject(refined_coord, pose, intrinsics)
    
    # 2. Coarse Sampling (Stratified)
    t_coarse = (torch.arange(S_c, dtype=torch.float32) + torch.rand(S_c)) / S_c * ray_length
    points_coarse = ray_origin.unsqueeze(0) + t_coarse.unsqueeze(1) * ray_direction.unsqueeze(0)
    enc_points_coarse = positional_encoding(points_coarse)
    
    # 3. Coarse Evaluation
    D_coarse, feat_coarse = shape_mlp(enc_points_coarse)
    
    # 4. Zero-Crossing Search
    signs = torch.sign(D_coarse)
    zero_cross_idx = torch.where(signs[:-1] != signs[1:])[0]
    if len(zero_cross_idx) == 0:
        zero_cross_idx = torch.tensor([S_c // 2])
        
    # 5. Fine Sampling
    idx = zero_cross_idx[0]
    t_cross = (t_coarse[idx] + t_coarse[idx+1]) / 2.0
    # [G] Window size not specified in sources; use small fixed range
    t_fine = torch.linspace(t_cross - 0.1, t_cross + 0.1, S_f)
    points_fine = ray_origin.unsqueeze(0) + t_fine.unsqueeze(1) * ray_direction.unsqueeze(0)
    enc_points_fine = positional_encoding(points_fine)
    
    # 6. Fine Evaluation
    D_fine, feat_fine = shape_mlp(enc_points_fine)
    
    # Concatenate
    D_all = torch.cat([D_coarse, D_fine])
    feat_all = torch.cat([feat_coarse, feat_fine])
    enc_points_all = torch.cat([enc_points_coarse, enc_points_fine])
    
    # 7. Weight Calculation
    weights = torch.sigmoid(D_all / tr) * torch.sigmoid(-D_all / tr)
    # Zero out weights beyond first truncation region
    # [G] Simple heuristic: zero out after first zero-crossing index
    if len(zero_cross_idx) > 0:
        cross_idx = zero_cross_idx[0]
        weights[cross_idx + 1:] = 0.0
        
    # 8. Radiance Evaluation
    enc_view = positional_encoding(ray_direction.unsqueeze(0).expand(len(D_all), -1))
    latent_expanded = latent_code.unsqueeze(0).expand(len(D_all), -1)
    radiance = radiance_mlp(feat_all, enc_view, latent_expanded)
    
    # 9. Color Integration
    color = torch.sum(weights.unsqueeze(1) * radiance, dim=0) / (torch.sum(weights) + 1e-8)
    
    return color, D_all, weights

def train_step(shape_mlp, radiance_mlp, deformation_mlp, poses, intrinsics, 
               images, depths, latent_codes, tr, S_c, S_f, ray_length, 
               lambda_rgb, lambda_fs, lambda_tr, optimizer):
    optimizer.zero_grad()
    total_loss = 0
    batch_size = 1024
    H, W = images[0].shape[1], images[0].shape[2]
    pixel_indices = torch.randint(0, H*W, (batch_size,))
    frame_indices = torch.randint(0, len(images), (batch_size,))
    
    for i in range(batch_size):
        f_idx = frame_indices[i]
        p_idx = pixel_indices[i]
        u, v = divmod(p_idx, W)
        pixel_coord = torch.tensor([u, v], dtype=torch.float32)
        img = images[f_idx][:, v, u]
        pose = poses[f_idx]
        latent = latent_codes[f_idx]
        
        color_pred, D_all, weights = render_ray(
            shape_mlp, radiance_mlp, deformation_mlp, pose, intrinsics,
            pixel_coord, latent, tr, S_c, S_f, ray_length
        )
        
        loss_rgb = (color_pred - img).pow(2).mean()
        
        mask_fs = D_all > tr
        mask_tr = (D_all >= -tr) & (D_all <= tr)
        
        loss_fs = torch.tensor(0.0)
        if mask_fs.any():
            target_fs = torch.full_like(D_all[mask_fs], tr)
            loss_fs = (D_all[mask_fs] - target_fs).pow(2).mean()
            
        loss_tr = torch.tensor(0.0)
        if mask_tr.any():
            observed_D = get_observed_signed_distance(D_all[mask_tr], pose, intrinsics, depths[f_idx])
            loss_tr = (D_all[mask_tr] - observed_D).pow(2).mean()
            
        loss = lambda_rgb * loss_rgb + lambda_fs * loss_fs + lambda_tr * loss_tr
        total_loss += loss
        
    total_loss.backward()
    optimizer.step()
    return total_loss.item()
```

## Subtopics

### [InFusionSurf](12834-infusionsurf.md)
InFusionSurf is a NeRF-style RGB-D 3D surface reconstruction framework that refines depth information to enhance reconstruction quality with minimal impact on optimization time, using per-frame intrinsic refinement and TSDF Fusion prior learning. It addresses camera motion-induced blurs in depth frames by correcting the camera intrinsic matrix per frame and accelerates training by pre-training the geometric prior using classical TSDF Fusion. It builds on [Neural RGB-D Surface Reconstruction](11352-neural-rgb-d-surface-reconstruction.md) by adding specific intrinsic refinement and a two-stage initialization process.

## Operating characteristics
*   **Time Complexity:** Training requires $2 \times 10^5$ iterations [S1]. Each iteration processes 1024 rays [S1]. The two-step sampling (coarse + fine) doubles the forward pass cost compared to single-pass methods [S1].
*   **Memory Complexity:** Depends on the MLP size and the number of samples per ray [S1]. The deformation field and latent codes add minor overhead [S1].
*   **Measured Performance:** On 10 synthetic scenes, the method achieved a Chamfer $\ell_1$ distance of 0.044, IoU of 0.747, Normal Consistency of 0.918, and F-score of 0.924 [S1].
*   **Strengths:** Produces high-quality, metric 3D reconstructions with hard boundaries, avoiding the semi-transparent artifacts common in density-based NeRFs [S1]. Robust to noisy depth data by combining color and depth losses [S1].
*   **Limitations:** Requires RGB-D input [S1]. Performance depends on the quality of initial camera poses (initialized via BundleFusion) [S1]. If coarse sampling is too sparse, the zero-crossing may be missed, leading to reconstruction failure [S1].
*   **When to use:** When metric 3D mesh reconstruction is required from RGB-D sequences, especially in room-scale scenes [S1].
*   **When not to use:** When only novel view synthesis is needed without metric geometry, or when only RGB input is available (use standard NeRF or RGB-D methods with monocular depth estimation) [S1].

## Background
Neural RGB-D Surface Reconstruction was proposed by Dejan Azinović, Ricardo Martin-Brualla, Dan B. Goldman, Matthias Nießner, and Justus Thies in 2022 [S1]. It was presented at CVPR 2022 [S1]. The method addresses the limitations of neural radiance fields (NeRF) which, while excellent for novel view synthesis, do not reconstruct metric and clean meshes [S1]. It builds on the idea of implicit surface representations (SDFs) and integrates them with the differentiable rendering framework of NeRF [S1]. It is related to concurrent work such as NeuS [S1]. The method is particularly useful for AR/VR applications, teleconferencing, and robotic applications requiring high-fidelity 3D models [S1].

## Sources
[S1] Neural RGB-D Surface Reconstruction, https://doi.org/10.1109/cvpr52688.2022.00619, fetched 2026-10-26
[S2] Geometric implicit neural representations for signed distance functions, https://doi.org/10.1016/j.cag.2024.104085, fetched 2026-10-26
