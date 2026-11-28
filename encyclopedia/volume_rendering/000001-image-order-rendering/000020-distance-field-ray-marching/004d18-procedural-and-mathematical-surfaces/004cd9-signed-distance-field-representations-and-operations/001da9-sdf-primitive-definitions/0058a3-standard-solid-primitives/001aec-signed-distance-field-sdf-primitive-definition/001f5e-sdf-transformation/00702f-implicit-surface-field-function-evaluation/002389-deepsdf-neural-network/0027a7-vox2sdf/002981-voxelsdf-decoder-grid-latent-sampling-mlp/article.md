# VoxelSDF Decoder (Grid Latent Sampling + MLP)

Also called: Latent Grid SDF, Voxel-conditioned SDF MLP

## Overview
The VoxelSDF Decoder predicts Signed Distance Function (SDF) values at arbitrary 3D query points by interpolating a gridded latent code and passing the result through a Multi-Layer Perceptron (MLP) [S1, unofficial]. It solves the problem of evaluating implicit surfaces for shapes represented by voxel grids, enabling differentiable rendering and optimization in neural implicit reconstruction pipelines [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A tensor of 3D query points $P$ with shape `(batch_size, num_points, 3)` [S1, unofficial]. A latent embedding grid $E$ with shape `(batch_size, latent_dim, voxel_grid_size+1, voxel_grid_size+1, voxel_grid_size+1)` [S1, unofficial].
*   **Output:** A tensor of SDF values $S$ with shape `(batch_size, num_points, 1)` [S1, unofficial].

**Data Structures:**
*   **Latent Grid:** A 5D tensor `embeddings` of shape `(batch_size, latent_dim, voxel_grid_size+1, voxel_grid_size+1, voxel_grid_size+1)` [S1, unofficial].
*   **Points:** A tensor `points` of shape `(batch_size, num_points, 3)` containing 3D coordinates [S1, unofficial].

**Parameters:**
*   `latent_dim`: Dimension of the latent vectors. Default: 32 [S1, unofficial].
*   `voxel_grid_size`: Spatial resolution of the latent grid. Default: 8 [S1, unofficial].
*   `num_layers`: Number of residual blocks in the MLP. Default: 4 [S1, unofficial].
*   `hidden_dim`: MLP hidden size. Default: 128 [S1, unofficial].

**Algorithm:**
1. **Grid Sampling:** Interpolate the latent code for each query point from the latent grid using trilinear interpolation [G]. The coordinates must be normalized to the range $[-1, 1]$ for standard grid sampling implementations [G].
2. **Reshape:** Flatten the batch and point dimensions to prepare for the MLP [G].
3. **MLP Evaluation:** Pass the interpolated latent codes through a residual MLP to predict the SDF value [S1, unofficial].

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
        layers = []
        for i in range(num_layers):
            in_ch = latent_dim if i == 0 else hidden_dim
            out_ch = hidden_dim
            layers.append(nn.Linear(in_ch, out_ch))
            layers.append(nn.ReLU())
            if i < num_layers - 1:
                layers.append(nn.Linear(out_ch, out_ch))
                layers.append(nn.ReLU())
        self.mlp = nn.Sequential(*layers)

    def forward(self, embeddings, points):
        n_models = embeddings.shape[0]
        n_points_per_model = points.shape[1]
        # F.grid_sample expects grid of shape (N, D_out, H_out, W_out, 3) for 3D
        # points shape is (B, N, 3) -> reshape to (B, 1, 1, N, 3)
        grid = points.view(n_models, 1, 1, n_points_per_model, 3)
        
        # Source uses mode='bilinear', but 'trilinear' is required for 5D tensors [G]
        latent = F.grid_sample(
            embeddings, 
            grid, 
            mode='trilinear', 
            padding_mode='zeros', 
            align_corners=True
        )
        # latent shape: (batch_size, latent_dim, 1, 1, num_points)
        latent = latent.view(n_models, n_points_per_model, self.latent_dim)
        
        sdf = self.mlp(latent)
        sdf = sdf.view(n_models, n_points_per_model, 1)
        return sdf
```

**Edge Cases and Pitfalls:**
*   **Boundary Conditions:** Queries outside the voxel grid bounds return zero due to `padding_mode='zeros'` [G].
*   **Coordinate Normalization:** `F.grid_sample` requires coordinates in $[-1, 1]$ when `align_corners=True` [G].

## Subtopics

### [Voxel-Conditional Neural Closest Point Predictor](10954-voxel-conditional-neural-closest-point-predictor.md)
A neural network module that synthesizes surface details by mapping any 3D query point $x$ to its closest point on a target surface $S$, conditioned on a voxelized representation $V$ of the shape. It solves the problem of refining coarse voxel-based mesh reconstructions into high-fidelity surfaces by predicting the displacement vector to the implicit surface, effectively defining the surface as the zero-level set of the predicted vector field. Unlike methods requiring ground-truth point clouds or normals (e.g., CSPNet), it operates solely on the voxel input, making it suitable for generative tasks where ground-truth geometry is unavailable.

### Vox-Fusion Implicit Decoder
A neural network decoder that maps 16-D voxel feature embeddings to signed distance field (SDF) and RGB values using fully connected layers [S2]. It processes the 16-D embedding through 2-4 fully connected layers, each containing 256 hidden units [S2]. The SDF head outputs a scalar SDF value $s$ and a 128-D hidden vector [S2]. The color head consists of two FC layers with 256 hidden units each, followed by a sigmoid activation to clamp RGB values to $[0, 1]$ [S2]. It uses a voxel size of 0.2m for all scenes and a step size ratio for sampling voxel points of 0.05-0.1 [S2].

### SparseNeuS Surface Extraction
Given an arbitrary 3D location $q$, positional encoding $PE$ is applied to its 3D coordinates [S3]. The geometry encoding volume $M$ is interpolated at location $q$ to obtain $M(q)$ [S3]. The surface extraction operation is expressed as $s(q) = f_\theta(PE(q), M(q))$ [S3].

## Operating characteristics
Not given in the sources.

## Background
Not given in the sources.

## Sources
[S1] TurtleJayWang/VC-SDFGen, https://github.com/TurtleJayWang/VC-SDFGen, fetched 2026-10-28 (unofficial)
[S2] Vox-Fusion: Dense Tracking and Mapping with Voxel-based Neural Implicit Representation, https://doi.org/10.1109/ismar55827.2022.00066, fetched 2026-10-26
[S3] SparseNeuS: Fast Generalizable Neural Surface Reconstruction from Sparse Views, https://doi.org/10.1007/978-3-031-19824-3_13, fetched 2026-10-26
