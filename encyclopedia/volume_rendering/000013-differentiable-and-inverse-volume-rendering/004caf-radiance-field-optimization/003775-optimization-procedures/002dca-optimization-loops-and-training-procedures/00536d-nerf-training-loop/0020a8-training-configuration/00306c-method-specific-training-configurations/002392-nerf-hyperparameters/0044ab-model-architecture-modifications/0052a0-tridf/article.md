# TriDF
Also called: Triplane-Accelerated Density Fields

## Overview
TriDF is a hybrid explicit–implicit 3D representation designed for few-shot novel view synthesis (NVS), particularly in remote sensing scenes with sparse input views [S1]. It decouples color and volume density to reduce the computational burden on implicit radiance fields and accelerate reconstruction from as few as 3 input views [S1]. Unlike standard NeRF architectures that jointly model low-frequency geometry and high-frequency appearance, TriDF uses explicit triplanes for high-frequency color prediction and implicit density fields for volume density, combined with image-based rendering (IBR) and depth-guided optimization [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** $V$ input images (typically $V=3$) with known camera poses, and their associated depth maps or point clouds generated via Structure-from-Motion (SfM) and PatchMatch Stereo (PMS) [S1].
*   **Outputs:** Rendered RGB images $\hat{C}(r)$ for arbitrary novel view rays $r$ [S1].

**Data Structures:**
1.  **Triplane ($P$):** A set of three mutually orthogonal feature planes $P = \{P_{XY}, P_{YZ}, P_{ZX}\}$ [S1].
    *   Each plane $P_{ij}$ is a tensor of size $H \times W \times C$ [S1].
    *   Typical values: $H, W$ are fixed resolutions; $C$ is the channel dimension [S1].
    *   Initialized randomly and optimized directly during training [S1].
2.  **Density MLP:** A Multi-Layer Perceptron implementing the CUTLASS architecture [S1].
    *   Width: 512 [S1].
    *   Depth: 8 layers [S1].
3.  **Color MLP:** A compact MLP for color prediction [S1].
4.  **Base MLP ($\Phi_{base}$):** Combines triplane and density features [S1].
5.  **Reference Features:** Extracted from input images using a pretrained CNN encoder [S1].

**Numbered Steps:**

1.  **Initialization:** Estimate initial 3D point clouds from training views and poses using COLMAP (SfM and PMS) [S1]. These point clouds provide sparse depth supervision [S1].
2.  **Feature Extraction:** Extract pixel-aligned reference features $f_{ref}$ from input images using a pretrained CNN encoder [S1]. The reference feature maps maintain a dimension of 64 [S1].
3.  **Triplane Feature Query:** For a 3D spatial point $\mathbf{x}$:
    *   Project $\mathbf{x}$ orthogonally onto the three planes $P_{XY}, P_{YZ}, P_{ZX}$ [S1].
    *   Identify the four nearest grid points on each plane [S1].
    *   Apply bilinear interpolation to compute feature vectors $f_{XY}, f_{YZ}, f_{ZX}$ [S1].
    *   Concatenate these vectors to form the triplane feature vector:
        $$
        f_{tri} = \text{concat}(f_{XY}, f_{YZ}, f_{ZX})
        $$
        [S1]
4.  **Density Field Prediction:**
    *   Project reference features to obtain corresponding feature vectors for each sampling point [S1].
    *   Concatenate the projected feature vectors with positionally encoded coordinates $\gamma(\mathbf{x})$ (positional encoding frequency = 6) [S1].
    *   Process the concatenated vector through the Density MLP to predict volume density $\sigma$ and intermediate features $f_m$ [S1].
5.  **Color Prediction:**
    *   Combine triplane features $f_{tri}$ and density features $f_m$ using the Base MLP to generate intermediate features $f_{base}$:
        $$
        f_{base} = \Phi_{base}(f_{tri}, f_m)
        $$
        [S1]
    *   Encode the viewing direction $\mathbf{d} \in \mathbb{R}^3$ on the spherical harmonics basis $SH(\cdot)$ [S1].
    *   Process $f_{base}$ and $SH(\mathbf{d})$ through the Color MLP to predict the point's color $\mathbf{c} \in \mathbb{R}^3$ [S1]:
        $$
        \mathbf{c} = \Phi_{color}(f_{base}, SH(\mathbf{d}))
        $$
        [S1]
6.  **Volume Rendering:** Aggregate attributes along a ray $r$ using differentiable volume rendering [S1]:
    $$
    \hat{C}(r) = \sum_{i=1}^{N} T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i
    $$
    where $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$, $\mathbf{c}_i$ and $\sigma_i$ are color and volume density, $\delta_i$ is the distance between adjacent samples, and $N$ is the number of sampling points [S1].
7.  **Optimization:**
    *   Use AdamW optimizer with initial learning rate 0.0001 [S1].
    *   **Phase 1 (Iterations 1–10,000):** Optimize with depth guidance. Re-project key points and their depth from the generated point clouds to use as spatial anchors [S1].
    *   **Phase 2 (Iterations 10,001–30,000):** Optimize with edge-aware smoothness loss. Process 64 key points and randomly render a $16 \times 16$ patch with stride 4 [S1].
    *   Loss functions include photometric loss between predicted and ground truth colors, depth loss, and smoothness loss [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class TriDF:
    def __init__(self, H, W, C, num_samples=128, pos_enc_freq=6):
        """
        Initialize TriDF model.
        :param H: Height of triplane feature maps
        :param W: Width of triplane feature maps
        :param C: Channel dimension of triplane feature maps
        :param num_samples: Number of coarse samples per ray
        :param pos_enc_freq: Frequency for positional encoding
        """
        self.H = H
        self.W = W
        self.C = C
        self.num_samples = num_samples
        self.pos_enc_freq = pos_enc_freq
        
        # Triplane: 3 planes of size H x W x C
        # Initialized randomly [S1]
        self.triplane = nn.Parameter(torch.randn(3, H, W, C))
        
        # Density MLP: CUTLASS architecture, width 512, depth 8 [S1]
        self.density_mlp = self._build_cutlass_mlp(input_dim=6 + 64, hidden_dim=512, depth=8)
        
        # Base MLP: Combines triplane features and density features [S1]
        # Input: f_tri (3*C) + f_m (output of density_mlp)
        # Output: f_base
        self.base_mlp = self._build_compact_mlp(input_dim=3*C + 512, hidden_dim=256, output_dim=256)
        
        # Color MLP: Predicts color from f_base and SH encoded direction [S1]
        # Input: f_base (256) + SH(d) (depends on degree, typically 12-27 for low degree)
        # Output: color (3)
        self.color_mlp = self._build_compact_mlp(input_dim=256 + 12, hidden_dim=128, output_dim=3)
        
        # Pretrained CNN for reference features (frozen) [S1]
        self.encoder = self._load_pretrained_cnn()
        
    def _build_cutlass_mlp(self, input_dim, hidden_dim, depth):
        layers = []
        for i in range(depth):
            in_dim = input_dim if i == 0 else hidden_dim
            layers.append(nn.Linear(in_dim, hidden_dim))
            layers.append(nn.ReLU())
        return nn.Sequential(*layers)

    def _build_compact_mlp(self, input_dim, hidden_dim, output_dim):
        layers = []
        layers.append(nn.Linear(input_dim, hidden_dim))
        layers.append(nn.ReLU())
        layers.append(nn.Linear(hidden_dim, output_dim))
        return nn.Sequential(*layers)

    def _load_pretrained_cnn(self):
        # Placeholder for pretrained CNN encoder [S1]
        # Returns a feature extractor that outputs 64-channel features
        pass

    def positional_encoding(self, x, freq):
        """
        Standard positional encoding [S1, S6]
        """
        # x is (..., 3)
        scales = torch.tensor([2**i for i in range(freq)])
        x = x.unsqueeze(-1) * scales.unsqueeze(0)
        x = torch.cat([torch.sin(x), torch.cos(x)], dim=-1)
        return x.reshape(x.shape[0], -1)

    def spherical_harmonics(self, d, degree=3):
        """
        Encode viewing direction d on spherical harmonics basis [S1]
        Simplified implementation for degree 3 (12 coefficients)
        """
        # d is (..., 3), normalized
        x, y, z = d[..., 0], d[..., 1], d[..., 2]
        # SH coefficients for degree 0-3
        sh = torch.zeros((*d.shape[:-1], 12), device=d.device)
        sh[..., 0] = 0.282095
        sh[..., 1] = 0.488603 * y
        sh[..., 2] = 0.488603 * z
        sh[..., 3] = 0.488603 * x
        sh[..., 4] = 1.092548 * x * y
        sh[..., 5] = -1.092548 * y * z
        sh[..., 6] = 0.315392 * (3 * z**2 - 1)
        sh[..., 7] = -1.092548 * x * z
        sh[..., 8] = 0.546274 * (x**2 - y**2)
        sh[..., 9] = 1.092548 * x * y
        sh[..., 10] = -1.092548 * y * z
        sh[..., 11] = 0.546274 * (x**2 - y**2)
        # Note: This is a simplified SH basis. Real implementation uses proper SH functions.
        # The source [S1] just says "SH(d)".
        return sh

    def query_triplane(self, x):
        """
        Query triplane features for point x [S1]
        x: (N, 3)
        """
        # Project x onto XY, YZ, ZX planes
        # XY plane: use x, y coordinates
        # YZ plane: use y, z coordinates
        # ZX plane: use z, x coordinates
        
        # Normalize coordinates to [-1, 1] for grid sampling
        # Assuming scene is bounded, e.g., [-1, 1]^3
        x_norm = x * 0.5 + 0.5 # Map to [0, 1]
        
        xy_coords = x_norm[:, [0, 1]] # (N, 2)
        yz_coords = x_norm[:, [1, 2]] # (N, 2)
        zx_coords = x_norm[:, [2, 0]] # (N, 2)
        
        # Bilinear interpolation for each plane
        # Grid is (3, H, W, C)
        # Sample from each plane
        xy_feat = F.grid_sample(self.triplane[0].unsqueeze(0), xy_coords.unsqueeze(0).unsqueeze(0), mode='bilinear', padding_mode='zeros').squeeze()
        yz_feat = F.grid_sample(self.triplane[1].unsqueeze(0), yz_coords.unsqueeze(0).unsqueeze(0), mode='bilinear', padding_mode='zeros').squeeze()
        zx_feat = F.grid_sample(self.triplane[2].unsqueeze(0), zx_coords.unsqueeze(0).unsqueeze(0), mode='bilinear', padding_mode='zeros').squeeze()
        
        # Concatenate features
        f_tri = torch.cat([xy_feat, yz_feat, zx_feat], dim=-1) # (N, 3*C)
        return f_tri

    def forward(self, rays_o, rays_d, reference_images, reference_poses):
        """
        Forward pass for a batch of rays [S1]
        rays_o: (B, 3) ray origins
        rays_d: (B, 3) ray directions
        reference_images: list of input images
        reference_poses: list of camera poses
        """
        # 1. Sample points along rays
        t_vals = torch.linspace(0, 1, self.num_samples, device=rays_o.device)
        z_vals = 0.1 + 0.9 * t_vals # Near/far bounds
        pts = rays_o.unsqueeze(1) + rays_d.unsqueeze(1) * z_vals.unsqueeze(2) # (B, N, 3)
        
        # 2. Get Triplane Features [S1]
        f_tri = self.query_triplane(pts.reshape(-1, 3)) # (B*N, 3*C)
        f_tri = f_tri.reshape(B, self.num_samples, -1) # (B, N, 3*C)
        
        # 3. Get Reference Features [S1]
        # Project reference features to sampling points
        # This involves projecting 3D points to 2D image planes and sampling features
        # Simplified: assume we have pre-computed reference features for each point
        f_ref = self.get_reference_features(pts, reference_images, reference_poses) # (B, N, 64)
        
        # 4. Density Field Prediction [S1]
        pos_enc = self.positional_encoding(pts.reshape(-1, 3), self.pos_enc_freq) # (B*N, 6*3=18)
        f_ref_flat = f_ref.reshape(-1, 64)
        density_input = torch.cat([pos_enc, f_ref_flat], dim=-1) # (B*N, 18+64=82)
        
        density_out = self.density_mlp(density_input) # (B*N, 512)
        sigma = density_out[..., 0] # First channel is density? Or separate head?
        # Source says "predict volume density sigma and intermediate features fm"
        # Assuming sigma is one output and fm is the rest or a separate head
        # Let's assume sigma is a scalar per point, fm is the rest of the vector
        sigma = torch.sigmoid(sigma) # Ensure positive density
        f_m = density_out[..., 1:] # (B*N, 511)
        
        sigma = sigma.reshape(B, self.num_samples, 1)
        f_m = f_m.reshape(B, self.num_samples, -1)
        
        # 5. Color Prediction [S1]
        f_base = self.base_mlp(torch.cat([f_tri, f_m], dim=-1)) # (B, N, 256)
        sh_d = self.spherical_harmonics(rays_d, degree=3) # (B, 12)
        # Expand sh_d to match N samples
        sh_d_expanded = sh_d.unsqueeze(1).expand(-1, self.num_samples, -1) # (B, N, 12)
        
        color_input = torch.cat([f_base, sh_d_expanded], dim=-1) # (B, N, 268)
        color = self.color_mlp(color_input) # (B, N, 3)
        
        # 6. Volume Rendering [S1]
        # Compute deltas
        deltas = z_vals[:, 1:] - z_vals[:, :-1]
        deltas = torch.cat([deltas, torch.tensor([1e10], device=rays_o.device)], dim=-1)
        
        # Transmittance
        alpha = 1.0 - torch.exp(-sigma * deltas)
        T = torch.cumprod(torch.cat([torch.ones_like(alpha[..., :1]), 1.0 - alpha + 1e-10], dim=-1), dim=-1)[..., :-1]
        
        # Rendered color
        weights = alpha * T
        rendered_color = torch.sum(weights.unsqueeze(-1) * color, dim=1)
        
        return rendered_color

    def get_reference_features(self, pts, ref_imgs, ref_poses):
        # Placeholder for IBR feature projection [S1]
        # Projects 3D points to 2D reference images and samples features
        pass
```

**Edge Cases and Pitfalls:**
*   **Overfitting with Sparse Views:** Standard triplane representations rely on dense input views and suffer from redundancy or inaccurate geometry modeling with sparse inputs [S1]. NeRF tends to overfit input views when trained with fewer images, degrading synthesized images [S1].
*   **IBR Integration Pitfall:** Integrating IBR features directly into the triplane branch causes overfitting because frozen encoder features are constant, and triplane features maintain a one-to-one correspondence with reference image features via projection, preventing updates [S1].
*   **Geometry Extrapolation:** Unlike 3DGS methods that require monocular depth priors or complex regularization, TriDF uses sparse point clouds for depth supervision without additional priors [S1]. However, implicit representations can leverage scene continuity to infer reasonable structures in adjacent regions, avoiding boundary distortions caused by 3DGS's explicit point cloud representation which fails to extrapolate to empty regions [S1].

## Operating characteristics
*   **Time Complexity:** Training time is approximately 5 minutes for 10,000 iterations (TriDF-10k) and 14 minutes for full training (30,000 iterations) on a single consumer GPU (Nvidia GeForce RTX 4090) [S1]. This is 30× faster than NeRF-based methods [S1].
*   **Memory Complexity:** Not explicitly given, but the triplane representation is compact ($H \times W \times C \times 3$) compared to dense voxel grids [S1].
*   **Measured Performance:** On the LEVIRNVS dataset, TriDF achieves a 7.4% increase in PSNR and 3.4% increase in SSIM compared to advanced few-shot methods [S1].
*   **Scaling:** The method is designed for few-shot scenarios (as few as 3 input views) [S1]. It does not require extensive pretraining on large multi-view datasets [S1].
*   **Strengths:** Fast convergence due to direct optimization of explicit triplane features; reduced computational burden by decoupling color and density; effective mitigation of overfitting via depth-guided optimization [S1].
*   **Limitations:** Rendering speed lags behind 3DGS-based methods designed for real-time rendering [S1]. Performance drops considerably when input views are sparse if not for the IBR and depth-guided components [S1].
*   **When to Use:** Remote sensing novel view synthesis with sparse input views (e.g., 3 views) [S1].
*   **When Not to Use:** Dense-view scenarios where standard NeRF or 3DGS might be more appropriate; real-time rendering applications requiring higher frame rates than TriDF provides [S1].

## Background
*   **Origin:** TriDF was proposed in a paper titled "TriDF: Triplane-Accelerated Density Fields for Few-Shot Remote Sensing Novel View Synthesis" [S1].
*   **History and Development:** The method addresses the limitations of existing NeRF-based methods, which are computationally intensive and perform sub-optimally in remote sensing scenes with sparse viewpoints [S1]. It builds on the triplane representation [S1] and image-based rendering frameworks [S1].
*   **Usage:** Validated on the LEVIRNVS dataset for remote sensing novel view synthesis [S1].

## Sources
[S1] TriDF: Triplane-Accelerated Density Fields for Few-Shot Remote Sensing Novel View Synthesis, https://doi.org/10.1109/tgrs.2025.3643432, fetched 2026-10-30
