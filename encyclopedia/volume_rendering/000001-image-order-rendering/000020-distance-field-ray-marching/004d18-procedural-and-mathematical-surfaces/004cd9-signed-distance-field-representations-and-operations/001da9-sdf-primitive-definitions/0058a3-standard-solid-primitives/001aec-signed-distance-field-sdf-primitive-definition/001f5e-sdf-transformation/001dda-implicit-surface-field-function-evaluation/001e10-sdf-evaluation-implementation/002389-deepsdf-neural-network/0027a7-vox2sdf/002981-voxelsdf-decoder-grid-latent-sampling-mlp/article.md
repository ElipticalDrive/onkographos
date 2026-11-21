# VoxelSDF Decoder (Grid Latent Sampling + MLP)

Also called: Latent Grid SDF, Voxel-conditioned SDF MLP

## Overview
The VoxelSDF Decoder predicts Signed Distance Function (SDF) values at arbitrary 3D query points by interpolating a gridded latent code and passing the result through a Multi-Layer Perceptron (MLP) [S1, unofficial]. It solves the problem of evaluating implicit surfaces for shapes represented by voxel grids, enabling differentiable rendering and optimization in neural implicit reconstruction pipelines [S1, unofficial]. Unlike standard DeepSDF which uses a single global latent vector, this method conditions the SDF evaluation on local spatial context via a latent grid, allowing for shape-specific details to be encoded per-voxel [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A tensor of 3D query points $P$ with shape $(N, 3)$, where $N$ is the number of points. A latent embedding grid $E$ with shape $(L, G+1, G+1, G+1)$, where $L$ is the latent dimension and $G$ is the voxel grid resolution.
*   **Output:** A tensor of SDF values $S$ with shape $(N, 1)$.

**Data Structures:**
*   **Latent Grid:** A 5D tensor `embeddings` of shape `(batch_size, latent_dim, voxel_grid_size+1, voxel_grid_size+1, voxel_grid_size+1)`. The extra dimension ($+1$) accounts for boundary conditions or specific indexing schemes in the interpolation library [S1, unofficial].
*   **Points:** A tensor `points` of shape `(batch_size, num_points, 3)` containing normalized coordinates (typically in $[-1, 1]$ or $[0, 1]$ depending on the grid definition) [S1, unofficial].

**Parameters:**
*   `latent_dim`: Dimension of the latent vectors. Default: 32 [S1, unofficial].
*   `voxel_grid_size`: Spatial resolution of the latent grid. Default: 8 [S1, unofficial].
*   `num_layers`: Number of residual blocks in the MLP. Default: 4 [S1, unofficial].
*   `hidden_dim`: MLP hidden size. Default: 128 [S1, unofficial].

**Algorithm:**
1.  **Grid Sampling:** Interpolate the latent code for each query point from the latent grid using trilinear interpolation. The coordinates must be normalized to the range $[-1, 1]$ for standard grid sampling implementations.
2.  **Reshape:** Flatten the batch and point dimensions to prepare for the MLP.
3.  **MLP Evaluation:** Pass the interpolated latent codes through a residual MLP to predict the SDF value.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class VoxelSDFDecoder(nn.Module):
    def __init__(self, latent_dim=32, voxel_grid_size=8, num_layers=4, hidden_dim=128):
        super().__init__()
        self.latent_dim = latent_dim
        self.voxel_grid_size = voxel_grid_size
        # MLP with residual connections
        layers = []
        for i in range(num_layers):
            in_ch = latent_dim if i == 0 else hidden_dim
            out_ch = hidden_dim
            layers.append(nn.Linear(in_ch, out_ch))
            layers.append(nn.ReLU())
            # Residual connection for all but the last layer
            if i < num_layers - 1:
                layers.append(nn.Linear(out_ch, out_ch)) # Skip connection projection
                layers.append(nn.ReLU())
        self.mlp = nn.Sequential(*layers)

    def forward(self, embeddings, points):
        """
        embeddings: (batch_size, latent_dim, grid_size+1, grid_size+1, grid_size+1)
        points: (batch_size, num_points, 3) in normalized coordinates [-1, 1]
        """
        batch_size = embeddings.shape[0]
        num_points = points.shape[1]
        
        # Normalize points to [-1, 1] if they are in [0, 1]
        # Assuming points are already in [-1, 1] for grid_sample
        # If points are in [0, 1], use: points = 2.0 * points - 1.0
        
        # Grid sampling: F.grid_sample expects input (N, C, H, W) or (N, C, D, H, W)
        # and grid (N, H_out, W_out, 2) or (N, D_out, H_out, W_out, 3)
        # We need to permute points to match the expected grid format
        # points shape: (B, N, 3) -> (B, N, 1, 1, 3) for 3D grid sample
        grid = points.unsqueeze(2).unsqueeze(3) 
        
        # Interpolate
        # mode='bilinear' is used in source, but for 3D it implies trilinear
        # padding_mode='zeros' handles out-of-bounds queries
        # align_corners=True ensures corners align with grid boundaries
        latent = F.grid_sample(
            embeddings, 
            grid, 
            mode='trilinear', 
            padding_mode='zeros', 
            align_corners=True
        )
        
        # latent shape: (batch_size, latent_dim, 1, 1, num_points)
        # Reshape to (batch_size * num_points, latent_dim)
        latent = latent.view(batch_size * num_points, self.latent_dim)
        
        # MLP Forward
        sdf = self.mlp(latent)
        
        # Reshape output to (batch_size, num_points, 1)
        sdf = sdf.view(batch_size, num_points, 1)
        
        return sdf
```

**Edge Cases and Pitfalls:**
*   **Boundary Conditions:** Queries outside the voxel grid bounds return zero due to `padding_mode='zeros'`. This may create artificial flat regions in the SDF field. [G] Ensure query points are clamped or handled explicitly if the domain extends beyond the grid.
*   **Coordinate Normalization:** `F.grid_sample` requires coordinates in $[-1, 1]$. Mismatched coordinate systems (e.g., $[0, 1]$) will result in incorrect sampling or out-of-bounds errors. [S1, unofficial]
*   **Memory:** The latent grid size grows cubically with resolution. A grid of size $8^3$ with latent dim 32 is manageable, but higher resolutions require significant VRAM.

## Subtopics

### [Voxel-Conditional Neural Closest Point Predictor](10954-voxel-conditional-neural-closest-point-predictor.md)
A neural network module that synthesizes surface details by mapping any 3D query point $x$ to its closest point on a target surface $S$, conditioned on a voxelized representation $V$ of the shape. It solves the problem of refining coarse voxel-based mesh reconstructions into high-fidelity surfaces by predicting the displacement vector to the implicit surface, effectively defining the surface as the zero-level set of the predicted vector field. Unlike methods requiring ground-truth point clouds or normals (e.g., CSPNet), it operates solely on the voxel input, making it suitable for generative tasks where ground-truth geometry is unavailable.

### Vox-Fusion Implicit Decoder
A neural network decoder that maps 16-D voxel feature embeddings to signed distance field (SDF) and RGB values using fully connected layers [S2]. It processes the 16-D embedding through 2-4 fully connected layers, each containing 256 hidden units [S2]. The SDF head outputs a scalar SDF value $s$ and a 128-D hidden vector [S2]. The color head consists of two FC layers with 256 hidden units each, followed by a sigmoid activation to clamp RGB values to $[0, 1]$ [S2]. It uses a voxel size of 0.2m for all scenes and a step size ratio for sampling voxel points of 0.05-0.1 [S2].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot D_{latent} \cdot L_{mlp})$, where $N$ is the number of query points, $D_{latent}$ is the latent dimension, and $L_{mlp}$ is the number of MLP layers. Grid sampling is $O(N)$ with constant overhead for interpolation.
*   **Memory Complexity:** $O(G^3 \cdot D_{latent})$ for the latent grid, where $G$ is the grid resolution.
*   **Performance:** Default parameters (latent_dim=32, grid_size=8) provide a balance between detail and memory usage [S1, unofficial].
*   **Strengths:** Differentiable, supports arbitrary query points, integrates well with gradient-based optimization.
*   **Limitations:** Resolution is limited by the grid size; fine details require high-resolution grids which are memory-intensive.

## Background
The VoxelSDF Decoder builds on the concept of DeepSDF, which uses a global latent vector to encode shape [S1, unofficial]. By replacing the global vector with a local latent grid, it allows for more detailed shape representations conditioned on spatial location [S1, unofficial]. It is used in neural implicit reconstruction pipelines for 3D shape generation and editing [S1, unofficial].

## Sources
[S1] TurtleJayWang/VC-SDFGen, https://github.com/TurtleJayWang/VC-SDFGen, fetched 2026-10-28 (unofficial)
[S2] Vox-Fusion: Dense Tracking and Mapping with Voxel-based Neural Implicit Representation, https://doi.org/10.1109/ismar55827.2022.00066, fetched 2026-10-26
