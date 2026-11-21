# Voxel-based Gaussian Splatting
Also called: VoxGS

## Overview
VoxGS is a volume representation for Computed Tomography (CT) that places isometric 3D Gaussians at the centers of a regular 3D voxel grid, enabling efficient CT volume extraction and differentiable X-ray rendering [S1, unofficial]. It sets itself apart from standard 3D Gaussian Splatting (3DGS) by fixing Gaussian positions to voxel centroids and removing view-dependent spherical harmonic (SH) color coefficients, which solves the inaccurate volume extraction and unstable optimization issues associated with applying original 3DGS to CT data [S1, unofficial]. Unlike discrete 3D voxels, VoxGS supports differentiable rendering, allowing gradients from X-ray projection losses to propagate back to the Gaussian attributes during training [S1, unofficial].

## How it works
**Inputs:**
- A set of $K$ X-ray projection images.
- Camera projection matrices $\mathbf{P}_k \in \mathbb{R}^{3 \times 4}$ for each view $k=1 \dots K$.
- A regular 3D voxel grid $V$ of dimensions $M \times N \times L$.
- View-specific feature maps $\mathbf{F}_k \in \mathbb{R}^{H \times W \times C}$ extracted from the X-ray projections via a Transformer encoder (e.g., ViT) [S1, unofficial].

**Outputs:**
- A set of $N_p$ 3D Gaussians $\mathcal{G} = \{g_i\}_{i=1}^{N_p}$, where $N_p = M \times N \times L$.
- A CT density volume $V \in \mathbb{R}^{M \times N \times L}$.

**Data Structures:**
Each Gaussian $g_i$ is defined by the following attributes:
- **Position $\boldsymbol{\mu}_i$**: Fixed at the centroid of the voxel grid. $\boldsymbol{\mu}_i = (x, y, z)^T$, where $x \in [1, M], y \in [1, N], z \in [1, L]$. This is not a learnable parameter but a structural index [S1, unofficial].
- **Opacity $\alpha_i$**: A scalar float representing the density/attenuation coefficient. Learnable.
- **Scale $\mathbf{s}_i$**: A 3D vector $(s_x, s_y, s_z)$ controlling the isotropic spread. Learnable.
- **Rotation $\mathbf{r}_i$**: A 4D quaternion $(w, x, y, z)$ controlling orientation. Learnable.
- **Color**: No SH coefficients or view-dependent color are stored. The representation assumes isotropic density for X-ray attenuation [S1, unofficial].

**Numbered Steps:**

1. **Position Initialization**: Assign each Gaussian $g_i$ a fixed position $\boldsymbol{\mu}_i$ corresponding to the integer coordinates of the voxel grid. There are $N_p = M \times N \times L$ Gaussians [S1, unofficial].

2. **Feature Querying**: For each Gaussian $g_i$ and each view $k=1 \dots K$:
   - Project the fixed 3D position $\boldsymbol{\mu}_i$ to 2D image coordinates using the view's projection matrix: $\mathbf{p}_{i,k} = \mathbf{P}_k \boldsymbol{\mu}_i$.
   - Query the view-specific feature map $\mathbf{F}_k$ at $\mathbf{p}_{i,k}$ using bilinear interpolation:
     $$
     \mathbf{F}_{i,k} = \text{Interp}(\mathbf{F}_k, \mathbf{p}_{i,k}) \in \mathbb{R}^C
     $$
     where $\text{Interp}(\cdot)$ denotes bilinear interpolation [S1, unofficial].

3. **Feature Aggregation**: Aggregate the $K$ queried features for Gaussian $g_i$ using MaxPooling to obtain a view-invariant feature vector:
   $$
   \mathbf{F}_{max}^i = \text{MaxPool}(\mathbf{F}_{i,1}, \mathbf{F}_{i,2}, \dots, \mathbf{F}_{i,K}) \in \mathbb{R}^C
   $$
   This step selects the most salient feature across all views [S1, unofficial].

4. **Attribute Regression**: Pass the aggregated feature $\mathbf{F}_{max}^i$ through a prediction Multi-Layer Perceptron (MLP), denoted as $MLPS$, to predict the Gaussian attributes:
   $$
   [\alpha_i, \mathbf{s}_i, \mathbf{r}_i] = MLPS(\mathbf{F}_{max}^i) \in \mathbb{R}^{1+3+4}
   $$
   The MLP outputs:
   - Opacity $\alpha_i$ (1 float).
   - Scale $\mathbf{s}_i$ (3 floats).
   - Rotation $\mathbf{r}_i$ (4 floats, quaternion) [S1, unofficial].

5. **Volume Extraction**: To extract the final CT density volume $V \in \mathbb{R}^{M \times N \times L}$, directly index the opacity of the Gaussian at the corresponding voxel position:
   $$
   V(x, y, z) = \alpha_i \quad \text{where} \quad \boldsymbol{\mu}_i = (x, y, z)^T
   $$
   This allows for $O(1)$ extraction without trilinear interpolation [S1, unofficial].

6. **Differentiable X-ray Rendering**: Render X-ray projections from the Gaussian set $\mathcal{G}$ using a differentiable rasterizer (e.g., based on [75] in the source) to compute the loss against input X-ray projections. The rendering process integrates the Gaussian densities along rays, leveraging the fixed positions for efficient computation [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class VoxGS:
    def __init__(self, M, N, L, C, K):
        """
        Initialize VoxGS representation.
        M, N, L: Voxel grid dimensions.
        C: Feature dimension from transformer.
        K: Number of views.
        """
        self.M, self.N, self.L = M, N, L
        self.K = K
        self.C = C
        self.num_gaussians = M * N * L
        
        # Fixed positions: shape (N_p, 3)
        # Coordinates are 1-based in source, converted to 0-based for indexing if needed,
        # but projection matrices handle the coordinate system.
        x = torch.arange(1, M + 1).float().view(-1, 1, 1)
        y = torch.arange(1, N + 1).float().view(1, -1, 1)
        z = torch.arange(1, L + 1).float().view(1, 1, -1)
        self.mu = torch.stack([x, y, z], dim=-1).reshape(-1, 3) # Shape (N_p, 3)
        
        # Learnable attributes initialized randomly or via prior
        self.alpha = nn.Parameter(torch.randn(self.num_gaussians, 1) * 0.1)
        self.scale = nn.Parameter(torch.randn(self.num_gaussians, 3))
        self.rotation = nn.Parameter(torch.randn(self.num_gaussians, 4))
        
        # Prediction MLP
        self.mlp = nn.Sequential(
            nn.Linear(C, 128),
            nn.ReLU(),
            nn.Linear(128, 128),
            nn.ReLU(),
            nn.Linear(128, 1 + 3 + 4) # alpha, s, r
        )

    def query_features(self, F_k_list, P_k_list):
        """
        F_k_list: List of K tensors, each (H, W, C)
        P_k_list: List of K tensors, each (3, 4) projection matrix
        Returns: F_max (N_p, C)
        """
        all_features = []
        for k in range(self.K):
            F_k = F_k_list[k] # (H, W, C)
            P_k = P_k_list[k] # (3, 4)
            
            # Project positions: mu is (N_p, 3). P_k is (3, 4).
            # Homogeneous coordinates: (N_p, 4)
            mu_h = torch.cat([self.mu, torch.ones(self.num_gaussians, 1, device=self.mu.device)], dim=1)
            p_2d_h = torch.matmul(mu_h, P_k.t()) # (N_p, 3)
            p_2d = p_2d_h[:, :2] / p_2d_h[:, 2:3] # (N_p, 2) normalized coordinates? 
            # Note: Source uses Interp(F_k, P_k(mu_i)). 
            # Assuming P_k outputs coordinates in the range of F_k's spatial dimensions.
            # Standard bilinear interpolation expects coordinates in [-1, 1] or [0, H/W].
            # We assume P_k maps to valid 2D coordinates for F_k.
            
            # Reshape for grid_sample: (N, C, H, W) -> (N, C, 1, 1) for point sampling
            # grid_sample expects grid of shape (N, H_out, W_out, 2)
            # Here we sample N_p points.
            grid = p_2d.view(1, 1, self.num_gaussians, 2)
            F_k_perm = F_k.permute(2, 0, 1).unsqueeze(0) # (1, C, H, W)
            
            # Sample features
            F_sampled = F.grid_sample(F_k_perm, grid, align_corners=False) # (1, C, 1, N_p)
            F_k_i = F_sampled.squeeze(0).squeeze(2).permute(1, 0) # (N_p, C)
            all_features.append(F_k_i)
        
        # MaxPool across views: shape (N_p, C)
        features_stack = torch.stack(all_features, dim=0) # (K, N_p, C)
        F_max = torch.max(features_stack, dim=0).values # (N_p, C)
        return F_max

    def predict_attributes(self, F_max):
        """
        F_max: (N_p, C)
        Returns: alpha, scale, rotation
        """
        out = self.mlp(F_max) # (N_p, 8)
        alpha = out[:, 0:1]
        scale = out[:, 1:4]
        rotation = out[:, 4:8]
        return alpha, scale, rotation

    def extract_volume(self):
        """
        Returns CT volume V of shape (M, N, L)
        """
        V = self.alpha.view(self.M, self.N, self.L)
        return V

    def render_xray(self, P_k, F_k):
        """
        Placeholder for differentiable X-ray rendering using the Gaussian set.
        Uses a rasterizer similar to [75] in source.
        """
        # Implementation depends on the specific differentiable rasterizer used.
        # The key is that gradients flow from the rendered projection to alpha, scale, rotation.
        pass
```

**Parameters:**
- $M, N, L$: Voxel grid dimensions. Typical values depend on the CT scan resolution (e.g., $256^3$, $512^3$).
- $K$: Number of X-ray views. Sparse-view scenarios (e.g., $K=12, 20$) are common in the source context [S1, unofficial].
- $C$: Feature dimension from the Transformer encoder.

**Edge Cases and Pitfalls:**
- **Direct 3DGS Application**: Using standard 3DGS with optimized positions and SH coefficients for CT leads to inaccurate volume extraction and unstable optimization in feed-forward frameworks [S1, unofficial]. VoxGS avoids this by fixing positions and removing SH.
- **Isotropic Assumption**: The removal of SH coefficients assumes isotropic density for X-ray attenuation. If view-dependent effects are significant, this simplification may limit accuracy [S1, unofficial].
- **Projection Coordinate Mapping**: The bilinear interpolation requires accurate mapping of 3D voxel centroids to 2D image coordinates via $\mathbf{P}_k$. Incorrect calibration will lead to misaligned feature queries [S1, unofficial].

## Operating characteristics
- **Time Complexity**: Volume extraction is $O(1)$ per voxel due to direct indexing [S1, unofficial]. Feature querying is $O(N_p \cdot K)$ for bilinear interpolation [S1, unofficial]. Attribute regression is $O(N_p)$ via MLP [S1, unofficial].
- **Memory Complexity**: Stores $N_p$ Gaussians with 8 floats each (1 opacity, 3 scale, 4 rotation) plus fixed positions. Total memory is proportional to $N_p$. This is comparable to storing a dense volume but with additional overhead for scale/rotation [S1, unofficial].
- **Performance**: X-GRM using VoxGS reconstructs CT volumes within one second for sparse X-ray inputs, significantly outperforming SOTA methods in inference speed and reconstruction quality [S1, unofficial].
- **Strengths**: Enables differentiable X-ray rendering, allowing gradient-based optimization from X-ray projections. Fast volume extraction. Stable optimization for feed-forward models.
- **Limitations**: Assumes isotropic density (no SH). Fixed positions limit flexibility in representing complex geometries compared to optimized 3DGS.

## Background
VoxGS was introduced in the paper "X-GRM: Large Gaussian Reconstruction Model for Sparse-view X-rays to Computed Tomography" [S1, unofficial]. It addresses the limitations of applying standard 3D Gaussian Splatting to CT reconstruction, specifically the instability and inaccuracy in volume extraction. By fixing Gaussian positions to voxel centroids and removing view-dependent color, it creates a representation tailored for X-ray attenuation modeling while retaining the differentiable rendering benefits of 3DGS.

## Sources
[S1] X-GRM: Large Gaussian Reconstruction Model for Sparse-view X-rays to Computed Tomography, https://doi.org/10.48550/arXiv.2505.15235, fetched 2026-10-27 (unofficial)
