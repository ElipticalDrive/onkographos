# GNeuVox

Also called: General Neural Voxels, Generalizable Neural Voxels

## Overview
GNeuVox is a fast-training framework for rendering free-view moving human bodies from monocular video by integrating neural fields with explicit neural voxel grids [S1, unofficial]. It solves the high computational cost of traditional NeRF-based human rendering by using two types of voxels: "general voxels" pretrained across multiple bodies to provide geometric priors, and "individual voxels" optimized per subject for unique details [S1, unofficial]. Unlike implicit MLP representations, this explicit structure significantly accelerates optimization, allowing fine-tuning for a new body in minutes rather than hours [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Monocular video frames (images), initial human pose estimates $P$ (joint positions $J$ and rotation angles $\Omega$), camera intrinsics/extrinsics, and ray origins/directions.
*   **Outputs:** Rendered RGB color $\mathbf{c}$ and volume density $\sigma$ for each ray sample, used to compute pixel color via volume rendering.

**Data Structures:**
1.  **General Voxels ($V_g$):** A 3D grid of shape $N_x \times N_y \times N_z$ with channel depth $C_v$. Initialized via pretraining on multiple subjects.
    *   Layout: `float32[Nx][Ny][Nz][Cv]`
    *   Typical Size: $160^3$ grid, $C_v = 6$.
2.  **Individual Voxels ($V_i$):** A 3D grid of the same shape as $V_g$, initialized to zero for each new subject.
    *   Layout: `float32[Nx][Ny][Nz][Cv]`
3.  **Deformation Embedding ($Z$):** A learnable vector/grid mapped to blend weights via 3D convolutions. Initialized to zero for new subjects.
4.  **Radiance Field ($\Phi_r$):** A small MLP taking concatenated voxel features, positional encoded coordinates, direction, and timestamp to predict $\sigma$ and $\mathbf{c}$.

**Numbered Steps:**

1.  **Pose Refinement:**
    Correct the initial pose $P = (J, \Omega)$ using a pose refinement network $\Phi_p$ to obtain $P_c = (J_c, \Omega_c)$.
    $$ P_c = (J, \Omega \otimes \Delta\Omega) $$
    where $\otimes$ denotes combining rotation matrices $R_i \cdot \Delta R_i$ and translations $T_i + \Delta T_i$ derived from $\Omega$ and $\Delta\Omega$ [S1, unofficial].

2.  **Coordinate Deformation:**
    Map observation space coordinates $\mathbf{x}_o = (x_o, y_o, z_o)$ to canonical T-pose coordinates $\mathbf{x}_c = (x_c, y_c, z_c)$ using the pose deformation network $\mathbf{F}_d$ and corrected pose $P_c$.
    $$ \mathbf{x}_c = \mathbf{F}_d(\mathbf{x}_o, P_c) $$
    The deformation is a weighted sum of rigid transformations:
    $$ T_{skel}(\mathbf{x}, P_c) = \sum_{i=1}^K w^i(\mathbf{x})(R_c^i \mathbf{x} + T_c^i) $$
    where $w^i(\mathbf{x})$ are blend weights obtained by mapping embedding $Z$ through 3D convolutions and trilinear interpolation [S1, unofficial].

3.  **Voxel Interpolation:**
    Perform trilinear interpolation on the canonical coordinates $\mathbf{x}_c$ to fetch features from both voxel grids.
    $$ \mathbf{v}_g = \text{TrilinearInterp}(\mathbf{x}_c, V_g) $$
    $$ \mathbf{v}_i = \text{TrilinearInterp}(\mathbf{x}_c, V_i) $$
    Where $\mathbf{v}_g \in \mathbb{R}^{C_v}$ and $\mathbf{v}_i \in \mathbb{R}^{C_v}$ [S1, unofficial].

4.  **Feature Concatenation and Encoding:**
    Concatenate features: $\mathbf{v}_{cat} = [\mathbf{v}_g; \mathbf{v}_i]$.
    Apply positional encoding $\gamma$ to $\mathbf{v}_{cat}$, coordinates $\mathbf{x}_o$, direction $\mathbf{d}$, and timestamp $t$.
    $$ \gamma(\mathbf{x}) = (\sin(2^0 \pi \mathbf{x}), \cos(2^0 \pi \mathbf{x}), \dots, \sin(2^{L-1} \pi \mathbf{x}), \cos(2^{L-1} \pi \mathbf{x})) $$
    *   Frequency for $\mathbf{v}$: 2.
    *   Frequency for $\mathbf{x}_o$: 10.
    *   Frequency for $\mathbf{d}$: 4 [S1, unofficial].

5.  **Radiance Field Prediction:**
    Feed encoded features into the radiance field $\Phi_r$ to predict density $\sigma$ and color $\mathbf{c}$.
    $$ \sigma, \mathbf{c} = \Phi_r(\gamma(\mathbf{v}_{cat}), \gamma(\mathbf{x}_o), \gamma(\mathbf{d}), \gamma(t)) $$

6.  **Volume Rendering:**
    Compute pixel color $\hat{C}(\mathbf{r})$ along ray $\mathbf{r}$ with $N$ samples.
    $$ \hat{C}(\mathbf{r}) = \sum_{i=1}^N T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i $$
    $$ T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right) $$
    where $\delta_i = t_{i+1} - t_i$ [S1, unofficial].

7.  **Loss Computation:**
    $$ \mathcal{L} = \lambda_M \|\hat{C}(\mathbf{r}) - C(\mathbf{r})\|_2^2 + \lambda_L \mathcal{L}_{LPIPS} $$
    $\mathcal{L}_{LPIPS}$ is computed on $G=6$ patches of size $32 \times 32$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class GNeuVox:
    def __init__(self, grid_res=160, channel_dim=6, num_bones=24):
        # Voxel grids: [Nx, Ny, Nz, Cv]
        self.V_g = nn.Parameter(torch.randn(grid_res, grid_res, grid_res, channel_dim))
        self.V_i = nn.Parameter(torch.zeros(grid_res, grid_res, grid_res, channel_dim))
        
        # Deformation network components (simplified structure)
        # In practice, Z is mapped to weights via 3D Conv layers
        self.Z = nn.Parameter(torch.zeros(1, 1, 1, 1)) # Embedding
        self.deform_conv = nn.Conv3d(1, num_bones, kernel_size=1) # Maps Z to blend weights
        self.radiance_field = nn.Sequential(
            nn.Linear(channel_dim * 2 + 30 + 10 + 12 + 10, 256), # 2*6(v) + 30(x) + 10(d) + 10(t) approx
            nn.ReLU(),
            nn.Linear(256, 4) # sigma, r, g, b
        )
        
        # Hyperparameters
        self.freq_v = 2
        self.freq_x = 10
        self.freq_d = 4
        
    def positional_encoding(self, x, freq_bands):
        """Standard positional encoding."""
        # x shape: [..., D]
        # Output: [..., D * 2 * freq_bands]
        x = x.unsqueeze(-1)
        scales = 2.0 ** torch.arange(freq_bands).to(x.device)
        pe = torch.cat([torch.sin(x * scales * torch.pi), torch.cos(x * scales * torch.pi)], dim=-1)
        return pe.view(*x.shape[:-1], -1)

    def deform_coordinates(self, x_o, P_c, Z):
        """
        Deform observation coordinates to canonical space.
        x_o: [B, N, 3]
        P_c: Pose parameters (J, Omega)
        Z: Embedding
        Returns: x_c [B, N, 3]
        """
        # 1. Get blend weights from Z
        # Z is typically a small tensor, expanded and convolved to match spatial resolution or bone count
        # Simplified: Assume Z generates weights w^i for each bone i
        # In GNeuVox, Z is mapped via 3D conv layers to volume, then interpolated.
        # For implementation, we assume a function get_blend_weights(Z, x_o) exists.
        # Here we simulate the weighted sum of rigid transforms.
        
        # Rigid transforms R_c^i, T_c^i are derived from P_c
        # w^i(x) is obtained from Z via 3D Conv + Trilinear Interp
        # This step is complex; we assume a helper `compute_deformation` handles the SMPL-like deformation
        x_c = compute_deformation(x_o, P_c, Z) 
        return x_c

    def forward(self, x_o, d, t, P_c, Z):
        """
        Forward pass for a batch of rays.
        x_o: [B, N, 3] Observation coordinates
        d: [B, N, 3] Ray directions
        t: [B, N, 1] Timestamps
        P_c: Corrected pose
        Z: Subject embedding
        """
        # 1. Deform to canonical space
        x_c = self.deform_coordinates(x_o, P_c, Z)
        
        # 2. Interpolate Voxel Features
        # Trilinear interpolation on V_g and V_i
        # Normalize x_c to [-1, 1] for grid sampling
        x_c_norm = (x_c / (160/2)) - 1 # Assuming grid size 160 and centered
        x_c_norm = x_c_norm.unsqueeze(1) # [B, 1, N, 3]
        
        v_g = F.grid_sample(self.V_g.permute(0, 3, 1, 2).unsqueeze(0), x_c_norm, 
                            mode='bilinear', padding_mode='zeros').squeeze(-1).squeeze(1) # [B, N, Cv]
        # Note: grid_sample expects [N, C, H, W] or [N, C, D, H, W]. 
        # Correct 3D grid sample:
        # V_g shape: [1, Cv, Nx, Ny, Nz]
        # x_c_norm shape: [B, N, 1, 1, 3] -> needs to be [B, N, 1, 1, 3] for grid_sample in PyTorch 3D
        # Simplified logic for clarity:
        v_g = trilinear_interpolate(x_c, self.V_g)
        v_i = trilinear_interpolate(x_c, self.V_i)
        
        # 3. Concatenate and Encode
        v_cat = torch.cat([v_g, v_i], dim=-1) # [B, N, 2*Cv]
        
        pe_v = self.positional_encoding(v_cat, self.freq_v)
        pe_x = self.positional_encoding(x_o, self.freq_x)
        pe_d = self.positional_encoding(d, self.freq_d)
        pe_t = self.positional_encoding(t, 1) # Simple encoding for time
        
        input_features = torch.cat([pe_v, pe_x, pe_d, pe_t], dim=-1)
        
        # 4. Radiance Field
        # Output: sigma, color
        out = self.radiance_field(input_features)
        sigma = torch.sigmoid(out[..., 0]) # Ensure positive density
        color = torch.sigmoid(out[..., 1:]) # RGB
        
        return sigma, color

    def render(self, rays, P_c, Z):
        """
        rays: [B, N, 6] (ox, oy, oz, dx, dy, dz, t)
        """
        x_o = rays[..., :3]
        d = rays[..., 3:6]
        t = rays[..., 6:7]
        
        sigma, color = self.forward(x_o, d, t, P_c, Z)
        
        # Volume Rendering
        # delta_i is distance between samples. Assume uniform or given.
        # Simplified: assume uniform delta
        delta = 0.01 # Placeholder for actual ray step size
        alpha = 1.0 - torch.exp(-sigma * delta)
        
        # Transmittance
        T = torch.cumprod(torch.cat([torch.ones_like(alpha[..., :1]), 1.0 - alpha[..., :-1]], dim=-1), dim=-1)
        
        # Accumulate color
        weights = T * alpha
        C_hat = torch.sum(weights.unsqueeze(-1) * color, dim=-2)
        
        return C_hat
```

**Parameters:**
*   **Voxel Resolution:** $160^3$ for both $V_g$ and $V_i$ [S1, unofficial].
*   **Channel Depth ($C_v$):** 6 [S1, unofficial].
*   **Positional Encoding Frequencies:** $L=2$ for voxel features, $L=10$ for coordinates, $L=4$ for direction [S1, unofficial].
*   **Ray Sampling:** 128 points per ray [S1, unofficial].
*   **Optimizer:** Adam ($\beta_1=0.9, \beta_2=0.99$) [S1, unofficial].
*   **Learning Rates:** $5 \times 10^{-5}$ for most params, $2 \times 10^{-2}$ for voxel grids, $5 \times 10^{-4}$ for radiance field [S1, unofficial].
*   **LR Decay:** Factor 0.1 every 500k iterations [S1, unofficial].
*   **Loss Weights:** $\lambda_M=0.2, \lambda_L=1$ (pretrain/scratch); $\lambda_M=10, \lambda_L=0$ (fine-tune first 300 iters); $\lambda_M=0.2, \lambda_L=1$ (fine-tune remaining) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Training Collapse:** Training from scratch can lead to high variance or collapse. Pretrained general voxels provide necessary priors to stabilize fitting [S1, unofficial].
*   **Face Distortion:** Due to limited fine-tuning iterations ($\le 3k$) and reliance on SMPL priors, complex non-rigid facial deformations may appear vague or distorted [S1, unofficial].

## Subtopics

### Pretraining Strategy
Train $V_g$, $\mathbf{F}_d$, and $\Phi_r$ on multiple subjects (e.g., 7 from ZJU-MoCap or Human3.6M). $V_i$ and $Z$ are zero-initialized or unused. This builds a "template" of human geometry [S1, unofficial].

### Fine-tuning Strategy
For a new subject, initialize $V_i$ and $Z$ to zero. Load pretrained $V_g$ and $\Phi_r$. Optimize $V_i$ (and optionally $\mathbf{F}_d$) for a short duration (1k-3k iterations). This captures differential textures and unique body shapes [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Significantly faster than implicit NeRFs. Pretraining takes 15-50 minutes. Fine-tuning takes 5-15 minutes (1k-3k iterations) [S1, unofficial].
*   **Memory Complexity:** Dominated by the voxel grids. $2 \times 160^3 \times 6 \times 4$ bytes $\approx 18.4$ MB for voxels, plus network weights. Much lower than dense MLP storage for high-res scenes.
*   **Performance:** On ZJU-MoCap, GNeuVox-ZJU achieves PSNR 29.93 in 5 mins, comparable to HumanNeRF (30.41 PSNR, 288 hours) [S1, unofficial]. On PeopleSnapshot, it achieves ~27.7-27.9 PSNR in 5 mins, vs Neural Body (24.47 PSNR, 14 hours) [S1, unofficial].
*   **Strengths:** Extremely fast training/inference adaptation; explicit representation allows for easier editing and generalization; stable convergence due to priors.
*   **Limitations:** Lower fidelity on complex non-rigid details (face) compared to longer training methods; requires initial pose estimation (SMPL); resolution limited by voxel grid size [S1, unofficial].

## Background
GNeuVox was introduced in the paper "Generalizable Neural Voxels for Fast Human Radiance Fields" (2023) [S1, unofficial]. It builds upon concepts from Anim-NeRF, Neural Body, and HumanNeRF, replacing implicit MLPs with explicit neural voxel grids to accelerate optimization. The method leverages the commonality of human skeletal structures across individuals to enable generalization [S1, unofficial].

## Sources
[S1] Generalizable Neural Voxels for Fast Human Radiance Fields, https://doi.org/10.48550/arxiv.2303.15387, fetched 2026-10-26 (unofficial)
