# GNeuVox

Also called: General Neural Voxels, Generalizable Neural Voxels

## Overview
GNeuVox is a fast-training framework for rendering free-view moving human bodies from monocular video by integrating neural fields with explicit neural voxel grids [S1, unofficial]. It solves the high computational cost of traditional NeRF-based human rendering by using two types of voxels: general voxels ($V_g$) pretrained across multiple bodies to provide geometric priors, and individual voxels ($V_i$) optimized per subject for unique details [S1, unofficial]. Unlike implicit MLP representations, GNeuVox uses explicit voxel grids, enabling training in minutes rather than hours while maintaining comparable rendering quality [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Monocular video frames (images), initial pose estimates $P$ (joint positions $J$ and rotation angles $\Omega$), camera intrinsics/extrinsics, and ray origins/directions.
*   **Outputs:** Rendered pixel colors $\hat{C}(\mathbf{r})$ and densities $\sigma$ for novel views.

**Data Structures:**
1.  **General Voxels ($V_g$):** A 3D grid of shape $C_v \times N_x \times N_y \times N_z$.
    *   $N_x = N_y = N_z = 160$.
    *   $C_v = 6$ (feature channels).
    *   Initialized via pretraining on multiple subjects.
2.  **Individual Voxels ($V_i$):** A 3D grid of shape $C_v \times N_x \times N_y \times N_z$.
    *   Same dimensions as $V_g$.
    *   Initialized to zero for each new subject.
3.  **Pose Deformation Network ($F_d$):** Contains 3D convolutional layers and a learnable embedding vector $Z$.
    *   $Z$ is zero-initialized for new subjects.
4.  **Radiance Field ($\Phi_r$):** A neural network mapping voxel features and coordinates to color and density.

**Numbered Steps:**

1.  **Pose Refinement:**
    Correct the initial pose $P = (J, \Omega)$ to obtain $P_c = (J_c, \Omega_c)$ using a pose refinement network $\Phi_p$.
    $$P_c = (J, \Omega \otimes \Delta\Omega)$$
    where $\otimes$ denotes $(R_i \cdot \Delta R_i, T_i + \Delta T_i)$, with $R_i, T_i$ derived from $\Omega$ and $\Delta R_i, \Delta T_i$ derived from $\Delta\Omega = \Phi_p(\Omega)$ [S1, unofficial].

2.  **Coordinate Deformation:**
    Map observation coordinates $\mathbf{x}_o = (x_o, y_o, z_o)$ to canonical T-pose coordinates $\mathbf{x}_c = (x_c, y_c, z_c)$ using the pose deformation network $F_d$ and corrected pose $P_c$:
    $$\mathbf{x}_c = F_d(\mathbf{x}_o, P_c)$$
    The deformation is defined as:
    $$T_{skel}(\mathbf{x}, P_c) = \sum_{i=1}^K w^i(\mathbf{x})(R_c^i \mathbf{x} + T_c^i)$$
    where $w^i(\mathbf{x})$ are blend weights obtained by mapping the embedding $Z$ through 3D convolutional layers and trilinear interpolation [S1, unofficial].

3.  **Voxel Interpolation:**
    Perform trilinear interpolation on the canonical coordinates $\mathbf{x}_c$ to extract features from both voxel grids:
    $$\mathbf{v}_g = \text{interp}(\mathbf{x}_c, V_g)$$
    $$\mathbf{v}_i = \text{interp}(\mathbf{x}_c, V_i)$$
    where $\text{interp}$ denotes multi-distance interpolation (MDI) [S1, unofficial].

4.  **Feature Concatenation and Encoding:**
    Concatenate $\mathbf{v}_g$ and $\mathbf{v}_i$. Apply positional encoding $\gamma$ to the concatenated features, coordinates $\mathbf{x}_o$, direction $\mathbf{d}$, and timestamp $t$:
    $$\gamma(\mathbf{x}) = (\sin(2^0 \mathbf{x}), \cos(2^0 \mathbf{x}), \dots, \sin(2^{L-1} \mathbf{x}), \cos(2^{L-1} \mathbf{x}))$$
    Parameters:
    *   $L=2$ for voxel features $\mathbf{v}$.
    *   $L=10$ for coordinates $\mathbf{x}_o$.
    *   $L=4$ for direction $\mathbf{d}$ [S1, unofficial].

5.  **Radiance Field Prediction:**
    Feed encoded features into the radiance field $\Phi_r$ to predict density $\sigma$ and color $\mathbf{c}$:
    $$\sigma, \mathbf{c} = \Phi_r(\gamma(\mathbf{v}_g), \gamma(\mathbf{v}_i), \gamma(t), \gamma(\mathbf{x}_o), \gamma(\mathbf{d}))$$

6.  **Volume Rendering:**
    Sample $N=128$ points along each ray. Compute pixel color $\hat{C}(\mathbf{r})$:
    $$\hat{C}(\mathbf{r}) = \sum_{i=1}^N T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i$$
    $$T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$$
    where $\delta_i = t_{i+1} - t_i$ [S1, unofficial].

7.  **Loss Computation:**
    $$\mathcal{L} = \lambda_M \|\hat{C}(\mathbf{r}) - C(\mathbf{r})\|_2^2 + \lambda_L \mathcal{L}_{LPIPS}$$
    $\mathcal{L}_{LPIPS}$ is computed on $G=6$ patches of size $32 \times 32$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

def positional_encoding(x, num_freq_bands):
    """
    Computes positional encoding for input x.
    x: tensor of shape (..., D)
    Returns: tensor of shape (..., D * 2 * num_freq_bands)
    """
    freq_bands = 2.0 ** torch.arange(num_freq_bands, device=x.device).float()
    scaled_x = x.unsqueeze(-1) * freq_bands
    pe = torch.cat([torch.sin(scaled_x), torch.cos(scaled_x)], dim=-1)
    return pe.view(*x.shape[:-1], -1)

class GNeuVox:
    def __init__(self, voxel_res=160, voxel_channels=6):
        # Voxel grids
        self.V_g = nn.Parameter(torch.randn(voxel_channels, voxel_res, voxel_res, voxel_res))
        self.V_i = nn.Parameter(torch.zeros(voxel_channels, voxel_res, voxel_res, voxel_res))
        
        # Embedding for deformation weights
        self.Z = nn.Parameter(torch.zeros(1, 1, 1, 1)) # Zero-initialized
        
        # Pose refinement network (simplified placeholder)
        self.pose_refine = nn.Sequential(
            nn.Linear(72, 72) # Assuming 24 joints * 3 params
        )
        
        # Deformation network (simplified placeholder)
        self.deform_net = nn.Sequential(
            nn.Conv3d(1, 64, 3, padding=1), # Input: Z (broadcasted)
            nn.ReLU(),
            nn.Conv3d(64, 64, 3, padding=1),
            nn.ReLU(),
            nn.Conv3d(64, 1, 1) # Output: blend weights
        )
        
        # Radiance field (simplified placeholder)
        self.radiance_field = nn.Sequential(
            nn.Linear(6 * 2 * 2 + 10 * 2 * 3 + 4 * 2 * 3 + 10 * 2, 256), # Concatenated encodings
            nn.ReLU(),
            nn.Linear(256, 4) # sigma, r, g, b
        )
        
        # Hyperparameters
        self.voxel_pe_freq = 2
        self.coord_pe_freq = 10
        self.dir_pe_freq = 4
        self.num_samples = 128

    def refine_pose(self, P):
        """
        P: tensor of shape (..., 72) representing pose (J, Omega)
        Returns: P_c, tensor of shape (..., 72)
        """
        # Simplified: just pass through a linear layer for demonstration
        # Real implementation involves rotation matrix composition
        return self.pose_refine(P)

    def deform_coordinates(self, x_o, P_c, Z):
        """
        x_o: tensor of shape (..., 3) observation coordinates
        P_c: tensor of pose parameters
        Z: tensor of shape (1, 1, 1, 1) embedding
        Returns: x_c, tensor of shape (..., 3) canonical coordinates
        """
        # Compute blend weights w^i from Z
        # In practice, Z is mapped to a volume via conv layers
        # Here we assume a simplified weight generation
        # Note: Real implementation uses K bones and trilinear interpolation of weights
        # Placeholder: identity deformation for simplicity if Z is zero
        # Real logic:
        # w = interpolate(conv_layers(Z), x_o)
        # x_c = sum(w_i * (R_i * x_o + T_i))
        
        # Simplified placeholder: return x_o shifted by a small offset based on P_c
        # This is a placeholder for the complex skeletal deformation logic
        offset = torch.zeros_like(x_o) # Placeholder for actual deformation
        x_c = x_o + offset
        return x_c

    def interpolate_voxels(self, x_c, V):
        """
        Trilinear interpolation of voxel grid V at coordinates x_c.
        x_c: tensor of shape (..., 3), normalized to [-1, 1] or [0, 1] depending on grid definition
        V: tensor of shape (C, N, N, N)
        Returns: features of shape (..., C)
        """
        # Normalize coordinates to [-1, 1] for grid_sample if necessary
        # Assuming x_c is in [-1, 1]
        x_c_norm = x_c # Placeholder for normalization logic
        # grid_sample expects (N, C, H, W, D) and grid of (N, ..., 3)
        # Reshape for grid_sample
        V_expanded = V.unsqueeze(0) # (1, C, N, N, N)
        x_c_grid = x_c_norm.unsqueeze(0).unsqueeze(0).unsqueeze(0) # (1, 1, 1, ..., 3)
        
        # Use F.grid_sample for trilinear interpolation
        # Note: F.grid_sample works on 2D/3D grids. For 3D, use grid_sample with align_corners=False
        # This is a simplified call; actual implementation requires careful handling of batch dims
        features = F.grid_sample(V_expanded, x_c_grid, mode='bilinear', align_corners=False) # Placeholder for 3D interp
        return features.squeeze()

    def forward(self, x_o, d, t, P):
        """
        Forward pass for rendering.
        x_o: (N, 3) observation coordinates
        d: (N, 3) ray directions
        t: (N,) timestamps
        P: (B, 72) poses for batch B
        Returns: sigma (N,), color (N, 3)
        """
        # 1. Pose Refinement
        P_c = self.refine_pose(P)
        
        # 2. Deformation
        x_c = self.deform_coordinates(x_o, P_c, self.Z)
        
        # 3. Voxel Interpolation
        v_g = self.interpolate_voxels(x_c, self.V_g)
        v_i = self.interpolate_voxels(x_c, self.V_i)
        
        # 4. Concatenation and Encoding
        v_concat = torch.cat([v_g, v_i], dim=-1) # (N, 2*C_v)
        
        pe_v = positional_encoding(v_concat, self.voxel_pe_freq)
        pe_x = positional_encoding(x_o, self.coord_pe_freq)
        pe_d = positional_encoding(d, self.dir_pe_freq)
        pe_t = positional_encoding(t.unsqueeze(-1), self.coord_pe_freq) # Treat t as scalar coord
        
        input_features = torch.cat([pe_v, pe_x, pe_d, pe_t], dim=-1)
        
        # 5. Radiance Field
        output = self.radiance_field(input_features)
        sigma = output[..., 0]
        color = torch.sigmoid(output[..., 1:])
        
        return sigma, color

    def render_ray(self, ray_o, ray_d, t_near, t_far):
        """
        Renders a single ray.
        """
        # Sample points along ray
        t_vals = torch.linspace(t_near, t_far, self.num_samples)
        x_samples = ray_o.unsqueeze(0) + ray_d.unsqueeze(0) * t_vals.unsqueeze(1)
        
        # Get sigma and color
        sigma, color = self.forward(x_samples, ray_d, t_vals, self.P_current)
        
        # Volume Rendering
        deltas = t_vals[1:] - t_vals[:-1]
        deltas = torch.cat([deltas, torch.tensor([1e10], device=deltas.device)])
        
        alpha = 1.0 - torch.exp(-sigma * deltas)
        T = torch.cumprod(torch.cat([torch.ones_like(alpha[:, :1]), 1.0 - alpha[:, :-1]], dim=-1), dim=-1)
        
        weights = alpha * T
        color_rendered = torch.sum(weights.unsqueeze(-1) * color, dim=-2)
        
        return color_rendered
```

**Edge Cases and Pitfalls:**
*   **Training Collapse:** Training from scratch can lead to collapse or high variance in results. Pretrained general voxels provide prior knowledge to stabilize fitting [S1, unofficial].
*   **Face Rendering:** Face rendering may show vagueness and distortion due to limited fine-tuning iterations ($\le 3k$) and reliance on SMPL priors, which struggle with complex non-rigid facial deformations [S1, unofficial].
*   **Boundary Conditions:** Trilinear interpolation requires coordinates to be within the voxel grid bounds. Out-of-bounds coordinates should be handled (e.g., clamping or zero-padding) [G].

## Operating characteristics
*   **Time Complexity:** Training time is significantly reduced compared to implicit NeRFs. Pretraining takes 15-50 minutes on multiple subjects. Fine-tuning for a new subject takes 5-15 minutes (1k-3k iterations) [S1, unofficial].
*   **Memory Complexity:** Uses explicit voxel grids of size $160^3$ with 6 channels. Two grids ($V_g, V_i$) are stored. This is more memory-intensive than implicit MLPs but allows faster optimization [S1, unofficial].
*   **Performance:** On ZJU-MoCap, GNeuVox-ZJU achieves PSNR 29.93 in 5 minutes, comparable to HumanNeRF (30.41 PSNR, 288 hours) [S1, unofficial]. On PeopleSnapshot, GNeuVox-ZJU achieves PSNR 27.73 in 5 minutes vs Neural Body (24.47 PSNR, 14 hours) [S1, unofficial].
*   **Strengths:** Fast training, generalization across subjects, explicit representation for easier optimization.
*   **Limitations:** Face rendering quality, limited to human bodies (though extensible), requires initial pose estimates [S1, unofficial].

## Background
GNeuVox was introduced in the paper "Generalizable Neural Voxels for Fast Human Radiance Fields" [S1, unofficial]. It builds upon concepts from Anim-NeRF, Neural Body, and HumanNeRF. The method addresses the high computational cost of NeRF-based human rendering by using explicit neural voxel grids and a two-stage training process (pretraining and fine-tuning) to leverage generalizable geometric priors [S1, unofficial].

## Sources
[S1] Generalizable Neural Voxels for Fast Human Radiance Fields, https://doi.org/10.48550/arxiv.2303.15387, fetched 2026-10-26 (unofficial)
