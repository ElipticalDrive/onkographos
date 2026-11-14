# Voxel-based Gaussian Splatting


## Overview
Voxel-based Gaussian Splatting (VoxGS) is a volume representation designed for efficient Computed Tomography (CT) extraction and differentiable X-ray rendering from sparse projections [S1, unofficial]. It solves the instability and inaccuracy of applying standard 3D Gaussian Splatting (3DGS) to CT data by fixing Gaussian positions to a regular voxel grid and removing view-dependent color coefficients [S1, unofficial]. Unlike standard 3DGS, which optimizes primitive positions and uses Spherical Harmonics for color, VoxGS regresses opacity, scale, and rotation from transformer tokens, enabling direct volume extraction via indexing and stable feed-forward optimization [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of $K$ X-ray projection images, represented as view feature maps $\{\mathbf{F}_k\}_{k=1}^K$ where $\mathbf{F}_k \in \mathbb{R}^{H \times W \times C}$, and camera projection matrices $\{\mathbf{P}_k\}_{k=1}^K$ [S1, unofficial].
*   **Output:** A set of $N_p$ 3D Gaussians $\mathcal{G} = \{g_i\}_{i=1}^{N_p}$ representing the CT volume, and the extracted CT density volume $V \in \mathbb{R}^{M \times N \times L}$ [S1, unofficial].

**Data Structures:**
Each Gaussian $g_i$ is defined by the following attributes, stored as floating-point values:
*   **Position ($\mu_i$):** Fixed 3D coordinates $(x, y, z)$ corresponding to the centroid of a voxel in the grid. $x \in [1, M], y \in [1, N], z \in [1, L]$. This is not a learnable parameter but a structural index [S1, unofficial].
*   **Opacity ($\alpha_i$):** A scalar in $\mathbb{R}$, representing the density/attenuation value [S1, unofficial].
*   **Scale ($\mathbf{s}_i$):** A 3D vector $(s_x, s_y, s_z) \in \mathbb{R}^3$ defining the anisotropic spread of the Gaussian [S1, unofficial].
*   **Rotation ($\mathbf{r}_i$):** A 4D vector representing a quaternion $(w, x, y, z) \in \mathbb{R}^4$ defining the orientation [S1, unofficial].
*   **Color:** No Spherical Harmonics (SH) coefficients are stored. The representation is isotropic and view-independent regarding color/appearance, as X-ray attenuation depends only on density [S1, unofficial].

**Numbered Steps:**

1.  **Grid Initialization:** Define a regular 3D voxel grid with dimensions $M \times N \times L$. Place $N_p = M \times N \times L$ isometric 3D Gaussians such that the position $\mu_i$ of each Gaussian $g_i$ is fixed at the centroid of the corresponding voxel [S1, unofficial].
    $$ \mu_i = (x, y, z), \quad x \in [1, M], y \in [1, N], z \in [1, L] \quad (5) $$

2.  **Feature Querying:** For each Gaussian $g_i$ and each view $k \in \{1, \dots, K\}$, project the fixed 3D position $\mu_i$ to 2D image coordinates using the view's projection matrix $\mathbf{P}_k$. Query the corresponding view-specific feature $F_k$ from the view feature map $\mathbf{F}_k$ using bilinear interpolation [S1, unofficial].
    $$ F_k = \text{Interp}(\mathbf{F}_k, \mathbf{P}_k(\boldsymbol{\mu}_i)) \in \mathbb{R}^C, \quad \text{for } k = 1, 2, \dots, K \quad (6) $$
    Here, $\mathbf{P}_k: \mathbb{R}^3 \to \mathbb{R}^2$ is the projection matrix for the $k$-th view, and $\text{Interp}(\cdot)$ denotes bilinear interpolation.

3.  **Feature Aggregation:** Aggregate the $K$ queried view-specific features for Gaussian $g_i$ using a MaxPooling operation to obtain a single aggregated feature vector $\mathbf{F}_{max}^i$ [S1, unofficial].
    $$ \mathbf{F}_{max}^i = \text{MaxPool}(F_i^1, F_i^2, \dots, F_i^K) \in \mathbb{R}^C $$

4.  **Attribute Regression:** Pass the aggregated feature $\mathbf{F}_{max}^i$ through a Multi-Layer Perceptron (MLP), denoted as $\text{MLPS}$, to predict the Gaussian's opacity, scale, and rotation parameters [S1, unofficial].
    $$ [\alpha_i, \mathbf{s}_i, \mathbf{r}_i] = \text{MLPS}(\mathbf{F}_{max}^i) \in \mathbb{R}^{1+3+4} \quad (7) $$
    The output consists of 1 value for opacity, 3 values for the scale vector, and 4 values for the rotation quaternion.

5.  **Volume Extraction:** To extract the discrete CT density volume $V \in \mathbb{R}^{M \times N \times L}$, directly index the opacity $\alpha_i$ of the Gaussian whose fixed position $\mu_i$ matches the voxel coordinates $(x, y, z)$ [S1, unofficial].
    $$ V(x, y, z) = \alpha_i, \quad \text{where } \boldsymbol{\mu}_i = (x, y, z)^T \quad (8) $$
    This allows for fast extraction without trilinear interpolation.

6.  **Differentiable X-ray Rendering:** Render X-ray projections by accumulating the Gaussian contributions along rays. The rendering uses a rasterizer (e.g., as implemented in [75]) to compute differentiable X-ray projections, allowing gradients to flow back to the MLP parameters during training [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class VoxelBasedGaussianSplatting:
    def __init__(self, grid_dims: tuple, num_views: int, feature_dim: int, mlp_hidden_dims: list):
        """
        Initialize VoxGS representation.
        
        Args:
            grid_dims: Tuple (M, N, L) defining the voxel grid size.
            num_views: Number of X-ray projection views K.
            feature_dim: Dimension C of the input view feature maps.
            mlp_hidden_dims: List of hidden layer sizes for the attribute prediction MLP.
        """
        self.M, self.N, self.L = grid_dims
        self.K = num_views
        self.C = feature_dim
        
        # Generate fixed positions for all voxels
        # Shape: (N_p, 3) where N_p = M * N * L
        x_coords = torch.arange(1, M + 1).float()
        y_coords = torch.arange(1, N + 1).float()
        z_coords = torch.arange(1, L + 1).float()
        
        # Create meshgrid and reshape
        xx, yy, zz = torch.meshgrid(x_coords, y_coords, z_coords, indexing='ij')
        positions = torch.stack([xx.flatten(), yy.flatten(), zz.flatten()], dim=-1)
        self.positions = positions # Fixed, not learnable
        
        self.N_p = positions.shape[0]
        
        # Define the MLP to predict [alpha, s, r]
        # Output dim: 1 (alpha) + 3 (scale) + 4 (rotation quaternion) = 8
        layers = []
        prev_dim = feature_dim
        for hidden_dim in mlp_hidden_dims:
            layers.append(nn.Linear(prev_dim, hidden_dim))
            layers.append(nn.ReLU())
            prev_dim = hidden_dim
        layers.append(nn.Linear(prev_dim, 8))
        self.mlp = nn.Sequential(*layers)
        
        # Initialize MLP weights [G]
        for m in self.mlp.modules():
            if isinstance(m, nn.Linear):
                nn.init.kaiming_normal_(m.weight, nonlinearity='relu')
                if m.bias is not None:
                    nn.init.zeros_(m.bias)

    def query_and_aggregate_features(self, view_features: torch.Tensor, projection_matrices: torch.Tensor):
        """
        Query view-specific features and aggregate them.
        
        Args:
            view_features: Tensor of shape (K, H, W, C) containing feature maps for each view.
            projection_matrices: Tensor of shape (K, 3, 4) or similar, defining P_k.
                                 Note: Actual projection logic depends on the specific camera model.
                                 This assumes P_k(mu) returns 2D coordinates.
        
        Returns:
            aggregated_features: Tensor of shape (N_p, C)
        """
        # Project all 3D positions to 2D for all views
        # This step requires the specific projection implementation from the source framework.
        # Assuming a helper function project_points exists that handles P_k and bilinear sampling.
        
        # Placeholder for the actual projection and interpolation logic described in [S1, unofficial]
        # F_k = Interp(F_k, P_k(mu_i))
        # We iterate over views to aggregate features for each Gaussian
        
        aggregated_features = []
        
        for k in range(self.K):
            # Get 2D projected coordinates for all Gaussians in view k
            # coords_2d shape: (N_p, 2)
            coords_2d = self._project_to_2d(self.positions, projection_matrices[k])
            
            # Bilinear interpolation from view feature map F_k
            # view_features[k] shape: (H, W, C) or (1, H, W, C)
            # F_k shape: (N_p, C)
            F_k = self._bilinear_interpolate(view_features[k], coords_2d)
            aggregated_features.append(F_k)
            
        # MaxPool across views: MaxPool(F_i^1, ..., F_i^K)
        # aggregated_features is a list of K tensors of shape (N_p, C)
        # Stack them to (K, N_p, C) and take max over dim 0
        stacked_features = torch.stack(aggregated_features, dim=0) # (K, N_p, C)
        max_pooled_features = torch.max(stacked_features, dim=0).values # (N_p, C)
        
        return max_pooled_features

    def _project_to_2d(self, positions_3d: torch.Tensor, projection_matrix: torch.Tensor) -> torch.Tensor:
        """
        Project 3D positions to 2D image coordinates using the projection matrix.
        This is a simplified placeholder. The actual implementation depends on the 
        camera model (e.g., pinhole) and the definition of P_k in [S1, unofficial].
        """
        # Homogeneous coordinates
        ones = torch.ones_like(positions_3d[:, :1])
        positions_homo = torch.cat([positions_3d, ones], dim=-1) # (N_p, 4)
        
        # Project: P * X_homo -> X_2d_homo
        projected_homo = torch.matmul(positions_homo, projection_matrix.T) # (N_p, 3)
        
        # Normalize to get 2D coordinates
        x = projected_homo[:, 0] / projected_homo[:, 2]
        y = projected_homo[:, 1] / projected_homo[:, 2]
        
        return torch.stack([x, y], dim=-1)

    def _bilinear_interpolate(self, feature_map: torch.Tensor, coords_2d: torch.Tensor) -> torch.Tensor:
        """
        Perform bilinear interpolation on the feature map at the given 2D coordinates.
        
        Args:
            feature_map: Tensor of shape (H, W, C) or (1, H, W, C).
            coords_2d: Tensor of shape (N_p, 2) with coordinates in [0, W-1] x [0, H-1] or normalized [-1, 1].
                       Note: PyTorch grid_sample expects normalized coordinates [-1, 1].
        """
        # Normalize coordinates to [-1, 1] for grid_sample
        # Assuming feature_map is (H, W, C)
        H, W, C = feature_map.shape
        if feature_map.dim() == 3:
            feature_map = feature_map.unsqueeze(0) # (1, H, W, C)
            
        # Normalize: x in [0, W-1] -> [-1, 1], y in [0, H-1] -> [-1, 1]
        # grid_sample expects (x, y) order, normalized
        x_norm = 2.0 * coords_2d[:, 0] / (W - 1) - 1.0
        y_norm = 2.0 * coords_2d[:, 1] / (H - 1) - 1.0
        
        grid = torch.stack([x_norm, y_norm], dim=-1).unsqueeze(0).unsqueeze(0) # (1, 1, N_p, 2)
        
        # grid_sample expects input (N, C, H, W)
        feature_map = feature_map.permute(0, 3, 1, 2) # (1, C, H, W)
        
        # Interpolate
        sampled_features = F.grid_sample(feature_map, grid, align_corners=True) # (1, C, 1, N_p)
        
        return sampled_features.squeeze(2).squeeze(0) # (N_p, C)

    def forward(self, view_features: torch.Tensor, projection_matrices: torch.Tensor):
        """
        Forward pass to predict Gaussian attributes.
        
        Args:
            view_features: Tensor of shape (K, H, W, C).
            projection_matrices: Tensor of shape (K, 3, 4) or similar.
            
        Returns:
            alphas: Tensor of shape (N_p,)
            scales: Tensor of shape (N_p, 3)
            rotations: Tensor of shape (N_p, 4)
        """
        # Step 2 & 3: Query and Aggregate
        aggregated_features = self.query_and_aggregate_features(view_features, projection_matrices)
        
        # Step 4: Regress Attributes
        # Output shape: (N_p, 8) -> [alpha, s_x, s_y, s_z, r_w, r_x, r_y, r_z]
        raw_output = self.mlp(aggregated_features)
        
        alpha = raw_output[:, 0]
        scale = raw_output[:, 1:4]
        rotation = raw_output[:, 4:8]
        
        return alpha, scale, rotation

    def extract_volume(self, alphas: torch.Tensor) -> torch.Tensor:
        """
        Extract the CT density volume V from the predicted opacities.
        
        Args:
            alphas: Tensor of shape (N_p,) containing opacity values.
            
        Returns:
            volume: Tensor of shape (M, N, L)
        """
        # Reshape alphas to match the grid dimensions
        volume = alphas.view(self.M, self.N, self.L)
        return volume
```

**Parameters:**
*   **Grid Dimensions ($M, N, L$):** Determine the resolution of the CT volume. Typical values depend on the target clinical or experimental CT scan resolution [S1, unofficial].
*   **Number of Views ($K$):** Number of X-ray projections used for reconstruction. The method is designed for sparse views [S1, unofficial].
*   **MLP Architecture:** The depth and width of the MLP ($\text{MLPS}$) are hyperparameters. The output dimension is fixed at 8 (1 for opacity, 3 for scale, 4 for rotation) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Position Fixing:** The positions $\mu_i$ are fixed. This prevents the model from learning arbitrary Gaussian positions, which is crucial for stable optimization in feed-forward frameworks but limits flexibility compared to standard 3DGS [S1, unofficial].
*   **No Color/SH:** The absence of Spherical Harmonics means the representation cannot model view-dependent appearance. This is appropriate for X-ray/CT where attenuation is isotropic, but makes it unsuitable for standard RGB novel view synthesis without modification [S1, unofficial].
*   **Isotropic Assumption:** While scale and rotation are learned, the initial placement is isometric (voxel-centered). The scale parameters allow for anisotropy, but the grid structure imposes a regularity constraint [S1, unofficial].

## Subtopics

### Inverse Multiquadric Splatting
Uses inverse multiquadric functions instead of Gaussians for smoother blending. Chosen when smoother transitions between primitives are needed, but at higher computational cost. [Inverse Multiquadric Splatting](9910-inverse-multiquadric-splatting.md)

### Opacity-Independent Gaussian Truncation
Truncates Gaussians based on a threshold independent of opacity. Chosen for consistent memory usage regardless of scene density. [Opacity-Independent Gaussian Truncation](10266-opacity-independent-gaussian-truncation.md)

### Directed Distance Field (DDF) for Gaussian Splatting
Uses DDFs to represent surfaces more accurately. Chosen for high-fidelity surface reconstruction rather than volume rendering. [Directed Distance Field (DDF) for Gaussian Splatting](12854-directed-distance-field-ddf-for-gaussian-splatting.md)

### Atomized Proliferation
Dynamically splits Gaussians based on error. Chosen for adaptive detail refinement in standard 3DGS. [Atomized Proliferation](12924-atomized-proliferation.md)

### Gaussian Voxel Kernel Functions (GVKF)
Combines voxel grids with Gaussian kernels. Chosen for hybrid representations balancing speed and accuracy. [Gaussian Voxel Kernel Functions (GVKF)](13979-gaussian-voxel-kernel-functions-gvkf.md)

### 3D Gaussian Particle Approximation of VDB Datasets
Approximates VDB datasets with Gaussians. Chosen for integrating Gaussian splatting with existing VDB-based workflows. [3D Gaussian Particle Approximation of VDB Datasets](14723-3d-gaussian-particle-approximation-of-vdb-datasets.md)

### Pruning and Reduction
Removes redundant Gaussians. Chosen for reducing memory footprint and rendering time in dense scenes. [Pruning and Reduction](14909-pruning-and-reduction.md)

## Operating characteristics
*   **Time Complexity:** The feature query and aggregation step is $O(N_p \cdot K)$, where $N_p$ is the number of voxels and $K$ is the number of views. The MLP inference is $O(N_p)$. Volume extraction is $O(1)$ per voxel due to direct indexing. This is significantly faster than iterative optimization-based CT reconstruction [S1, unofficial].
*   **Memory Complexity:** $O(N_p)$ to store the Gaussian attributes (opacity, scale, rotation). No SH coefficients are stored, reducing memory compared to standard 3DGS [S1, unofficial].
*   **Measured Performance:** X-GRM using VoxGS reconstructs CT volumes from sparse X-rays within one second, outperforming SOTA methods in both quality and speed [S1, unofficial].
*   **Strengths:** Fast, feed-forward inference; stable optimization for CT; direct volume extraction; differentiable rendering for training [S1, unofficial].
*   **Limitations:** Fixed grid resolution; no view-dependent color; limited to isotropic density representation (suitable for CT/X-ray, not general RGB scenes) [S1, unofficial].
*   **When to Use:** Sparse-view X-ray to CT reconstruction, differentiable X-ray rendering, scenarios requiring fast volume extraction [S1, unofficial].
*   **When Not to Use:** General novel view synthesis with view-dependent effects, scenes requiring adaptive resolution not aligned with a regular grid [S1, unofficial].

## Background
Voxel-based Gaussian Splatting (VoxGS) was introduced in the paper "X-GRM: Large Gaussian Reconstruction Model for Sparse-view X-rays to Computed Tomography" [S1, unofficial]. It addresses the limitations of applying standard 3DGS to CT data, where direct use leads to inaccurate volume extraction and unstable optimization [S1, unofficial]. By fixing positions to a voxel grid and removing color coefficients, VoxGS enables efficient and stable reconstruction of CT volumes from sparse X-ray projections [S1, unofficial].

## Sources
[S1] X-GRM: Large Gaussian Reconstruction Model for Sparse-view X-rays to Computed Tomography, https://doi.org/10.48550/arXiv.2505.15235, fetched 2026-10-27 (unofficial)
