# Voxel-based Gaussian Splatting
Also called: VoxGS

## Overview
VoxGS is a specialized volume representation for Computed Tomography (CT) that places isometric 3D Gaussians at fixed voxel grid centroids to enable precise volume extraction and differentiable X-ray rendering [S1, unofficial]. Unlike standard 3D Gaussian Splatting (3DGS) which optimizes primitive positions and uses Spherical Harmonics (SH) for view-dependent color, VoxGS fixes positions to allow direct opacity indexing for density fields and removes SH coefficients to model isotropic X-ray attenuation [S1, unofficial]. This design solves the instability and inaccurate volume extraction issues that arise when applying original 3DGS to CT reconstruction in feed-forward frameworks [S1, unofficial].

## How it works
**Inputs:**
*   A set of $K$ 2D X-ray projection images.
*   Camera projection matrices $\mathbf{P}_k$ for each view $k \in \{1, \dots, K\}$.
*   A regular 3D voxel grid with dimensions $M \times N \times L$.

**Outputs:**
*   A set of $N_p$ 3D Gaussians $\mathcal{G} = \{g_i\}$, where $N_p = M \times N \times L$.
*   A CT density volume $V \in \mathbb{R}^{M \times N \times L}$ extracted from the Gaussians.

**Data Structures:**
Each Gaussian $g_i$ is defined by the following attributes. Note that the position is fixed and not stored as an optimizable parameter in the same way as standard 3DGS.

*   **Position ($\mu_i$):** Fixed 3D vector. $\mu_i = (x, y, z)^T$, where $x \in [1, M], y \in [1, N], z \in [1, L]$. It corresponds to the centroid of the $i$-th voxel in the grid.
*   **Opacity ($\alpha_i$):** Scalar float in $[0, 1]$. Represents the density/attenuation value.
*   **Scale ($\mathbf{s}_i$):** 3D vector of floats. Defines the isotropic or anisotropic spread of the Gaussian kernel.
*   **Rotation ($\mathbf{r}_i$):** 4D vector of floats, representing a unit quaternion. Defines the orientation of the Gaussian.
*   **Color/SH:** None. Spherical Harmonics coefficients are removed because X-ray attenuation is isotropic and view-independent [S1, unofficial].

**Algorithm Steps:**

1.  **Grid Initialization:** Define a 3D voxel grid of size $M \times N \times L$. Assign a fixed Gaussian $g_i$ to each voxel center $(x, y, z)$. The position $\mu_i$ is constant during optimization.

2.  **Feature Querying:** For each Gaussian $g_i$ with fixed position $\mu_i$ and for each view $k=1 \dots K$:
    *   Project $\mu_i$ to the 2D image plane of view $k$ using the projection matrix $\mathbf{P}_k$:
        $$ \mathbf{p}_k = \mathbf{P}_k(\mu_i) $$
    *   Query the view-specific feature map $\mathbf{F}_k$ (output from a vision transformer encoder) at $\mathbf{p}_k$ using bilinear interpolation:
        $$ F_k = \text{Interp}(\mathbf{F}_k, \mathbf{p}_k) \in \mathbb{R}^C $$
        where $C$ is the feature dimension.

3.  **Feature Aggregation:** Aggregate the $K$ view-specific features for Gaussian $g_i$ using MaxPooling to obtain a global view-invariant feature vector:
    $$ \mathbf{F}_{max}^i = \text{MaxPool}(F_1, F_2, \dots, F_K) \in \mathbb{R}^C $$

4.  **Attribute Regression:** Pass $\mathbf{F}_{max}^i$ through a Multi-Layer Perceptron (MLP), denoted as $MLPS$, to predict the Gaussian attributes:
    $$ [\alpha_i, \mathbf{s}_i, \mathbf{r}_i] = MLPS(\mathbf{F}_{max}^i) \in \mathbb{R}^{1+3+4} $$
    *   $\alpha_i$: Opacity (1 float).
    *   $\mathbf{s}_i$: Scale (3 floats).
    *   $\mathbf{r}_i$: Rotation quaternion (4 floats).

5.  **Volume Extraction:** To extract the CT density volume $V$, directly index the opacity of the Gaussian corresponding to each voxel coordinate:
    $$ V(x, y, z) = \alpha_i \quad \text{where} \quad \mu_i = (x, y, z)^T $$
    This allows for $O(1)$ extraction without trilinear interpolation [S1, unofficial].

6.  **Differentiable X-ray Rendering:** Render X-ray projections by accumulating the Gaussian contributions along rays. The rendering uses a rasterizer (e.g., as implemented in [75]) that supports differentiable rendering of the Gaussians defined by $\alpha_i, \mathbf{s}_i, \mathbf{r}_i$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class VoxGS:
    def __init__(self, grid_dims: tuple, feature_dim: int, mlp_hidden_dims: list):
        """
        Initialize VoxGS representation.
        
        Args:
            grid_dims: Tuple (M, N, L) defining the voxel grid size.
            feature_dim: Dimension C of the input view features.
            mlp_hidden_dims: List of hidden layer sizes for the attribute prediction MLP.
        """
        self.M, self.N, self.L = grid_dims
        self.num_gaussians = M * N * L
        
        # Fixed positions for all Gaussians
        # Shape: (num_gaussians, 3)
        x = torch.arange(1, M + 1).float().view(-1, 1)
        y = torch.arange(1, N + 1).float().view(-1, 1)
        z = torch.arange(1, L + 1).float().view(-1, 1)
        
        # Create meshgrid and flatten
        xx, yy, zz = torch.meshgrid(x, y, z, indexing='ij')
        self.mu = torch.cat([xx.reshape(-1, 1), yy.reshape(-1, 1), zz.reshape(-1, 1)], dim=1)
        
        # MLP to predict attributes from aggregated features
        layers = []
        prev_dim = feature_dim
        for hidden_dim in mlp_hidden_dims:
            layers.append(nn.Linear(prev_dim, hidden_dim))
            layers.append(nn.ReLU())
            prev_dim = hidden_dim
        layers.append(nn.Linear(prev_dim, 1 + 3 + 4)) # alpha, scale, rotation
        self.mlp = nn.Sequential(*layers)
        
        # Initialize attributes to neutral values
        self.alpha = torch.ones(self.num_gaussians)
        self.scale = torch.ones(self.num_gaussians, 3)
        self.rotation = torch.zeros(self.num_gaussians, 4)
        self.rotation[:, 0] = 1.0 # Identity quaternion

    def query_and_aggregate(self, view_features: torch.Tensor, 
                            projection_matrices: torch.Tensor) -> torch.Tensor:
        """
        Query view features and aggregate them for all Gaussians.
        
        Args:
            view_features: Tensor of shape (K, H, W, C) containing features for K views.
            projection_matrices: Tensor of shape (K, 3, 4) or similar, defining P_k.
            
        Returns:
            Aggregated features for all Gaussians: (num_gaussians, C)
        """
        K = view_features.shape[0]
        aggregated_features = []
        
        for k in range(K):
            F_k = view_features[k] # (H, W, C)
            P_k = projection_matrices[k] # Projection matrix for view k
            
            # Project all mu to 2D for view k
            # mu is (N_p, 3). P_k is (3, 4) or similar.
            # Assuming homogeneous coordinates: p = P * [mu; 1]
            mu_homogeneous = torch.cat([self.mu, torch.ones(self.num_gaussians, 1)], dim=1) # (N_p, 4)
            p_2d_hom = torch.matmul(mu_homogeneous, P_k.t()) # (N_p, 3)
            p_2d = p_2d_hom[:, :2] / p_2d_hom[:, 2:3] # (N_p, 2) normalized coordinates
            
            # Bilinear interpolation
            # Normalize p_2d to [-1, 1] range if necessary, depending on Interp implementation
            # Assuming Interp expects coordinates in [0, W]x[0, H] or normalized [-1, 1]
            # Here we assume a standard grid_sample or similar that takes normalized coords
            # Adjust scaling based on specific Interp implementation details not fully specified in source
            # For this pseudocode, we assume p_2d is in the correct range for Interp
            
            F_k_sampled = F.grid_sample(
                F_k.permute(2, 0, 1).unsqueeze(0), # (1, C, H, W)
                p_2d.view(1, 1, -1, 2), # (1, 1, N_p, 2)
                align_corners=False
            ).squeeze() # (C, N_p)
            
            F_k_sampled = F_k_sampled.permute(1, 0) # (N_p, C)
            aggregated_features.append(F_k_sampled)
            
        # Stack features from all views: (K, N_p, C)
        all_features = torch.stack(aggregated_features, dim=0)
        
        # MaxPool across views: (N_p, C)
        max_features = torch.max(all_features, dim=0).values
        
        return max_features

    def predict_attributes(self, aggregated_features: torch.Tensor):
        """
        Predict Gaussian attributes from aggregated features.
        
        Args:
            aggregated_features: Tensor of shape (num_gaussians, C)
            
        Returns:
            alpha: (num_gaussians,)
            scale: (num_gaussians, 3)
            rotation: (num_gaussians, 4)
        """
        output = self.mlp(aggregated_features) # (num_gaussians, 8)
        
        alpha = output[:, 0]
        scale = output[:, 1:4]
        rotation = output[:, 4:8]
        
        # Normalize rotation to be a unit quaternion
        rotation = F.normalize(rotation, p=2, dim=1)
        
        return alpha, scale, rotation

    def extract_volume(self, alpha: torch.Tensor) -> torch.Tensor:
        """
        Extract CT volume by indexing opacities.
        
        Args:
            alpha: Tensor of shape (num_gaussians,)
            
        Returns:
            Volume: Tensor of shape (M, N, L)
        """
        volume = alpha.reshape(self.M, self.N, self.L)
        return volume
```

## Operating characteristics
*   **Time Complexity:** Volume extraction is $O(1)$ per voxel due to direct indexing, significantly faster than trilinear interpolation used in standard 3DGS or discrete voxel grids [S1, unofficial]. Rendering complexity depends on the number of Gaussians and the rasterizer implementation, but benefits from the fixed position structure [S1, unofficial].
*   **Memory Complexity:** Requires storing $N_p$ Gaussians. Since positions are fixed and implicit in the grid index, memory is saved compared to storing explicit position vectors if the grid is sparse, but for a dense grid, it stores $\alpha, \mathbf{s}, \mathbf{r}$ per voxel. No SH coefficients are stored, reducing memory per Gaussian compared to standard 3DGS [S1, unofficial].
*   **Strengths:** Enables precise and fast CT volume extraction; supports differentiable X-ray rendering for training constraints; stable optimization for feed-forward frameworks [S1, unofficial].
*   **Limitations:** Limited to isotropic density modeling (no SH); fixed positions may lack flexibility for complex geometries compared to free-form 3DGS [S1, unofficial].
*   **When to use:** Sparse-view X-ray to CT reconstruction where differentiable rendering constraints are needed and fast volume extraction is critical [S1, unofficial].
*   **When not to use:** General 3D scene reconstruction with view-dependent appearance (use standard 3DGS) or when high flexibility in Gaussian positions is required [S1, unofficial].

## Background
VoxGS was introduced in the X-GRM (Large Gaussian Reconstruction Model for Sparse-view X-rays to Computed Tomography) paper [S1, unofficial]. It was developed to address the limitations of applying original 3DGS to CT reconstruction, which suffered from inaccurate volume extraction and unstable optimization in feed-forward settings [S1, unofficial]. The method draws inspiration from the efficient rendering of 3DGS [28] and modifications for CT [75, 17], but specifically fixes Gaussian positions to voxel centroids and removes color SH to model isotropic X-ray attenuation [S1, unofficial].

## Sources
[S1] X-GRM: Large Gaussian Reconstruction Model for Sparse-view X-rays to Computed Tomography, https://doi.org/10.48550/arXiv.2505.15235, fetched 2026-10-27 (unofficial)
