# InFusionSurf


## Overview
InFusionSurf is a NeRF-style RGB-D 3D surface reconstruction framework that refines depth information to enhance reconstruction quality with minimal impact on optimization time, using per-frame intrinsic refinement and TSDF Fusion prior learning [S1]. It addresses camera motion-induced blurs in depth frames by correcting the camera intrinsic matrix per frame and accelerates training by pre-training the geometric prior using classical TSDF Fusion [S1]. It builds on [Neural RGB-D Surface Reconstruction](11352-neural-rgb-d-surface-reconstruction.md) by adding specific intrinsic refinement and a two-stage initialization process [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A sequence of RGB-D video frames (color images $C_{u,v}^i$ and depth maps), initial camera poses $[R|t]$, and initial camera intrinsics ($f_x, f_y, c_x, c_y$).
*   **Outputs:** A trained hybrid scene representation consisting of a dense feature grid $\mathcal{V}^{(feat)}$, MLPs for color ($MLP^C$) and signed distance ($MLP^D$), per-frame latent embeddings $\xi_i$, per-frame intrinsic refinement parameters ($s^i, \tau^i$), and an image-plane deformation field (from parent method). The final output is a renderable radiance field and a mesh extracted via Marching Cubes [S1].

**Data Structures:**
1.  **Dense Feature Grid $\mathcal{V}^{(feat)}$:**
    *   Layout: 3D array of dimensions $N_x \times N_y \times N_z$.
    *   Dimensions calculated as $N_x = \lceil L_x/g_s \rceil$, $N_y = \lceil L_y/g_s \rceil$, $N_z = \lceil L_z/g_s \rceil$, where $L_x, L_y, L_z$ are scene bounds and $g_s$ is cell size [S1].
    *   Values: Each cell stores a feature vector of length $F$.
    *   Initialization: Zeros [S1].
2.  **MLPs:**
    *   $MLP^C$: Color decoder. 2 hidden layers, 128 nodes each [S1].
    *   $MLP^D$: Signed distance decoder. 2 hidden layers, 128 nodes each [S1].
    *   Image-plane deformation MLP (from parent): 2 hidden layers, 64 nodes each [S1].
3.  **Per-Frame Parameters:**
    *   Scaling matrix $s^i \in \mathbb{R}^{3 \times 3}$ and translation vector $\tau^i \in \mathbb{R}^3$ for each frame $i$ [S1].
    *   Latent embedding $\xi_i$ for each frame $i$ [S1].

**Numbered Steps:**

1.  **Scene Bounds and Grid Setup:**
    *   Estimate scene bounds $L_x, L_y, L_z$ from depth frames and initial poses [S1].
    *   Set cell size $g_s$ (default 10cm) and feature length $F$ (default 12) [S1].
    *   Compute grid dimensions $N_x, N_y, N_z$ and initialize $\mathcal{V}^{(feat)}$ with zeros [S1].

2.  **Phase 1: TSDF Fusion Prior Learning:**
    *   Set $F=1$ for this phase [S1].
    *   Run TSDF Fusion on the input depth frames to generate a TSDF volume [S1].
    *   Randomly sample grid cells. For each cell, query its center point $p$.
    *   Query the TSDF Fusion volume for the signed distance value $D_{TSDF}(p)$ [S1].
    *   Compute feature vector $\phi_p$ via trilinear interpolation of the 8 nearest vertices in $\mathcal{V}^{(feat)}$ [S1].
    *   Compute predicted signed distance $\hat{D}_p = MLP^D(\phi_p)$ [S1].
    *   Minimize MSE between $\hat{D}_p$ and $D_{TSDF}(p)$ [S1].
    *   **Optimization Scope:** Only update $\mathcal{V}^{(feat)}$ and $MLP^D$ [S1].
    *   **Iterations:** 3,000 [S1].

3.  **Phase 2: Progressive Learning (Low-Frequency):**
    *   Set $F=12$ [S1].
    *   Randomly sample ray batches $r_b$.
    *   For each ray, sample points $S_c$ every 15.625mm along the ray [S1].
    *   Compute normalized image coordinates $\varrho_{u,v}$ using initial intrinsics [S1]:
        $$ \varrho_{u,v} = \begin{bmatrix} 1/f_x & 0 & -c_x/f_x \\ 0 & -1/f_y & c_y/f_y \\ 0 & 0 & -1 \end{bmatrix} \begin{bmatrix} v \\ u \\ 1 \end{bmatrix} $$
        *(Note: Source [S1] Eq. 6 uses $v$ for the first component and $u$ for the second, which is non-standard; standard is $u$ then $v$. However, the source explicitly writes $\begin{bmatrix} v \\ v \\ 1 \end{bmatrix}$ in the table but $\begin{bmatrix} v \\ u \\ 1 \end{bmatrix}$ in the text. We follow the text's variable assignment $u,v$ to pixel coordinates, but the matrix multiplication order in the source table is ambiguous. Standard pinhole model is $x = (u-c_x)/f_x$. The source matrix multiplies $\begin{bmatrix} v \\ u \\ 1 \end{bmatrix}$. This implies the source might have swapped $u,v$ in the vector or the matrix. We implement the source's explicit formula: $\varrho_{u,v} = K^{-1} [v, u, 1]^T$ if following the table strictly, but standard is $[u, v, 1]^T$. Given the text says "pixel (u,v)", we assume standard $u$ is x, $v$ is y. The source table has a typo $\begin{bmatrix} v \\ v \\ 1 \end{bmatrix}$. We use standard $\begin{bmatrix} u \\ v \\ 1 \end{bmatrix}$ with the inverse intrinsic matrix provided, or follow the source's specific matrix application. The source defines $\varrho_{u,v}$ using the matrix applied to $\begin{bmatrix} v \\ u \\ 1 \end{bmatrix}$. This is likely a typo in the source for $\begin{bmatrix} u \\ v \\ 1 \end{bmatrix}$. We will use the standard interpretation of the inverse intrinsic matrix applied to $[u, v, 1]^T$ as per Eq 6's structure, assuming the vector was meant to be $[u, v, 1]^T$.)*
        *Correction based on strict source adherence:* The source Eq 6 shows the matrix multiplied by $\begin{bmatrix} v \\ u \\ 1 \end{bmatrix}$. This is highly likely a typo for $\begin{bmatrix} u \\ v \\ 1 \end{bmatrix}$. However, to be safe, we note the source's specific layout. We will use the standard inverse intrinsic matrix $K^{-1}$ applied to $[u, v, 1]^T$.
    *   Apply per-frame intrinsic refinement (see below) to get refined $\hat{\varrho}_{u,v}^i$.
    *   Compute ray direction $\hat{d}_{u,v}^i = [R|t] [\hat{\varrho}_{u,v}^i; 1]$ [S1].
    *   Sample points $p$ along $\hat{d}_{u,v}^i$.
    *   Compute $\phi_p$ via trilinear interpolation from $\mathcal{V}^{(feat)}$ [S1].
    *   Compute $\hat{D}_p = MLP^D(\phi_p)$ [S1].
    *   Compute weight $\omega_p = \sigma(\hat{D}_p/tr) \cdot \sigma(-\hat{D}_p/tr)$ [S1].
    *   Compute color $C_{p,d}^i = MLP^C(\phi_p, \Lambda(\hat{d}_{u,v}^i), \xi_i)$ [S1].
    *   Compute rendered color $\hat{C}_r^i = \frac{\sum \omega_p C_{p,d}^i}{\sum \omega_p}$ [S1].
    *   Compute Loss $\mathcal{L}$ (see below) [S1].
    *   **Optimization Scope:** All parameters ($\mathcal{V}^{(feat)}, MLP^C, MLP^D, \xi_i, s^i, \tau^i$, deformation field) [S1].
    *   **Iterations:** 7,000 [S1].

4.  **Phase 3: Progressive Learning (High-Frequency):**
    *   Halve the grid cell size $g_s$ (e.g., from 10cm to 5cm) [S1].
    *   Re-initialize or resize $\mathcal{V}^{(feat)}$ to new dimensions $N_x, N_y, N_z$ [S1].
    *   Sample points $S_c$ as in Phase 2.
    *   Identify surfaces in $S_c$ and sample additional points $S_f$ around these surfaces [S1].
    *   Use both $S_c$ and $S_f$ for loss computation [S1].
    *   **Optimization Scope:** All parameters [S1].
    *   **Iterations:** 65,000 [S1].

**Per-Frame Intrinsic Refinement:**
*   For each frame $i$, optimize scaling $s^i$ and translation $\tau^i$ [S1].
*   $s^i = \text{diag}(s_x^i, s_y^i, 1)$, $\tau^i = [\tau_x^i, \tau_y^i, 0]^T$ [S1].
*   Initialize $s^i$ to identity (1,1,1) and $\tau^i$ to zero [S1].
*   Refined coordinate: $\hat{\varrho}_{u,v}^i = s^i \cdot (\varrho_{u,v} + \tau^i)$ [S1].

**Loss Function:**
$$ \mathcal{L} = \lambda_{fs} \mathcal{L}_{fs} + \lambda_{sdf} \mathcal{L}_{sdf} + \lambda_{rgb} \mathcal{L}_{rgb} + \lambda_{reg} \mathcal{L}_{reg} $$
*   $\mathcal{L}_{fs}$ (Points outside truncation $S_{fs}$): $\frac{1}{|r_b|} \sum_{r \in r_b} \frac{1}{|S_{fs}|} \sum_{p \in S_{fs}} (\hat{D}_p - tr)^2$ [S1].
*   $\mathcal{L}_{sdf}$ (Points inside truncation $S_{sdf}$): $\frac{1}{|r_b|} \sum_{r \in r_b} \frac{1}{|S_{sdf}|} \sum_{p \in S_{sdf}} (\hat{D}_p - D_r^i)^2$, where $D_r^i$ is the observed depth-derived signed distance [S1].
*   $\mathcal{L}_{rgb}$: $\frac{1}{|r_b|} \sum_{r \in r_b} (\hat{C}_r^i - C_{u,v}^i)^2$ [S1].
*   $\mathcal{L}_{reg}$: L2 regularization on $\xi_i$, $s^i, \tau^i$, and deformation parameters. Scaling parameters regularized around 1 [S1].

**Parameters:**
*   $tr = 5$cm [S1].
*   $\lambda_{fs} = 10, \lambda_{sdf} = 6 \times 10^3, \lambda_{rgb} = 0.5, \lambda_{reg} = 0.1$ [S1].
*   Optimizer: ADAM, lr=$5 \times 10^{-4}$, decays to 1/10 every 250K iterations, $\beta_1=0.9, \beta_2=0.999$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import numpy as np

class InFusionSurf:
    def __init__(self, scene_bounds, depth_frames, color_frames, initial_poses, initial_intrinsics):
        self.scene_bounds = scene_bounds # Lx, Ly, Lz
        self.depth_frames = depth_frames
        self.color_frames = color_frames
        self.poses = initial_poses # [R|t]
        self.intrinsics = initial_intrinsics # fx, fy, cx, cy
        
        # Hyperparameters
        self.gs = 0.1 # 10cm
        self.F = 12
        self.tr = 0.05 # 5cm
        
        # Grid dimensions
        Nx = int(np.ceil(scene_bounds[0] / self.gs))
        Ny = int(np.ceil(scene_bounds[1] / self.gs))
        Nz = int(np.ceil(scene_bounds[2] / self.gs))
        
        # Data structures
        self.V_feat = torch.zeros(Nx, Ny, Nz, 1) # Phase 1 uses F=1
        self.MLP_D = MLP(1, 128, 128, 1) # Input F, 2 hidden 128, Output 1 (SDF)
        self.MLP_C = MLP(12, 128, 128, 3) # Input F+enc+latent, 2 hidden 128, Output 3 (RGB)
        # Note: MLP_C input dimension is F + pos_enc_dim + latent_dim. 
        # Source says MLP_C takes phi_p, Lambda(d), xi_i. 
        # We assume standard positional encoding and latent vector concatenation.
        
        # Per-frame parameters
        self.num_frames = len(depth_frames)
        self.scales = torch.ones(self.num_frames, 3) # sx, sy, 1
        self.translations = torch.zeros(self.num_frames, 3) # tx, ty, 0
        self.latents = torch.randn(self.num_frames, 16) # Example latent dim
        
        # Optimizers
        self.opt_phase1 = torch.optim.Adam(list(self.V_feat.parameters()) + list(self.MLP_D.parameters()), lr=5e-4)
        self.opt_phase2 = torch.optim.Adam(
            list(self.V_feat.parameters()) + list(self.MLP_C.parameters()) + 
            list(self.MLP_D.parameters()) + [self.scales, self.translations, self.latents], 
            lr=5e-4
        )

    def trilinear_interp(self, V, points):
        # points: (N, 3) normalized to [0, 1] or grid coords
        # Returns (N, F)
        # Standard trilinear interpolation implementation
        pass

    def get_normalized_coords(self, u, v, fx, fy, cx, cy):
        # Eq 6
        # Standard: x = (u - cx) / fx, y = -(v - cy) / fy (assuming y-down or similar)
        # Source Eq 6 matrix:
        # [1/fx, 0, -cx/fx]
        # [0, -1/fy, cy/fy]
        # [0, 0, -1]
        # Multiplied by [v, u, 1]^T in source table, but likely [u, v, 1]^T.
        # We implement standard inverse intrinsic:
        x = (u - cx) / fx
        y = -(v - cy) / fy # Assuming y-axis direction matches source convention
        z = -1
        return torch.stack([x, y, z], dim=-1)

    def refine_intrinsics(self, rho, frame_idx):
        s = self.scales[frame_idx]
        t = self.translations[frame_idx]
        # rho is (N, 3)
        # s is (3,) or (3,3) diag
        s_mat = torch.diag(s)
        rho_refined = s_mat @ (rho + t.unsqueeze(0))
        return rho_refined

    def compute_sdf_loss(self, points, D_obs, is_outside):
        D_pred = self.MLP_D(self.trilinear_interp(self.V_feat, points))
        if is_outside:
            # L_fs
            target = self.tr
        else:
            # L_sdf
            target = D_obs
        return torch.mean((D_pred - target) ** 2)

    def render(self, ray_dirs, frame_idx):
        # Sample points along rays
        # Compute phi, D, C
        # Compute weights
        # Return weighted sum
        pass

    def train(self):
        # Phase 1
        self.V_feat.data = torch.zeros_like(self.V_feat) # Reset F=1
        # Run TSDF Fusion on depth_frames to get D_TSDF
        D_TSDF = run_tsdf_fusion(self.depth_frames)
        
        for _ in range(3000):
            # Sample grid cells
            # Query centers
            # Get D_TSDF
            # Forward MLP_D
            # Backward
            self.opt_phase1.step()
            
        # Phase 2
        self.V_feat.data = torch.zeros_like(self.V_feat) # Reset for F=12? Or keep? 
        # Source says "builds a dense grid... with F=1" for phase 1. 
        # Phase 2 says "All parameters... undergo optimized". 
        # It implies F changes to 12. We re-init or resize.
        self.V_feat = torch.zeros(Nx, Ny, Nz, 12)
        
        for _ in range(7000):
            # Sample rays and points Sc every 15.625mm
            # Compute loss
            # Backward
            self.opt_phase2.step()
            
        # Phase 3
        self.gs /= 2
        Nx, Ny, Nz = ... # Recalculate
        self.V_feat = torch.zeros(Nx, Ny, Nz, 12)
        
        for _ in range(65000):
            # Sample Sc and Sf
            # Compute loss
            # Backward
            self.opt_phase2.step()

class MLP(nn.Module):
    def __init__(self, in_dim, h1, h2, out_dim):
        super().__init__()
        self.net = nn.Sequential(
            nn.Linear(in_dim, h1),
            nn.ReLU(),
            nn.Linear(h1, h2),
            nn.ReLU(),
            nn.Linear(h2, out_dim)
        )
    def forward(self, x):
        return self.net(x)
```

## Operating characteristics
*   **Time Complexity:** Training is significantly faster than Neural RGB-D. 75K iterations take ~96 minutes on average, which is 7.3 to 8.7 times faster than Neural RGB-D [S1].
*   **Memory Complexity:** Uses a dense feature grid. Memory usage scales with $N_x N_y N_z F$. The grid size is determined by scene bounds and cell size $g_s$ [S1].
*   **Measured Performance:** On ScanNet V2 and synthetic scenes, InFusionSurf outperforms GO-Surf in C-ℓ1, IoU, and F-score with 20% less training time at 20K iterations [S1]. It is less effective on Normal Consistency (NC), indicating uneven surfaces [S1].
*   **Strengths:** Fast training, high-quality geometry reconstruction, handles motion blur in depth frames via intrinsic refinement [S1].
*   **Limitations:** Relies on simple transformation matrices for intrinsic refinement; does not handle local blurs (object motion, rolling shutter) effectively [S1]. Less effective on normal consistency [S1].
*   **When to use:** Quick reconstruction of complex geometries from RGB-D video with camera motion [S1].
*   **When not to use:** Scenes with significant local motion blur or rolling shutter effects where uniform intrinsic refinement is insufficient [S1].

## Background
InFusionSurf was introduced by Seunghwan Lee, Gwanmo Park, Hyewon Son, Jiwon Ryu, and Han Joo Chae from ROKIT Healthcare, Inc. [S1]. It was published in 2024 [S1]. It builds upon Neural RGB-D Surface Reconstruction [S1]. It is implemented in PyTorch and evaluated on the ScanNet V2 dataset and synthetic scenes [S1].

## Sources
[S1] InFusionSurf: Refining Neural RGB-D Surface Reconstruction Using Per-Frame Intrinsic Refinement and TSDF Fusion Prior Learning, https://doi.org/10.1109/icme57554.2024.10687901, fetched 2026-10-26
