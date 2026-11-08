# Low-rank 4D Bilateral Grid

Also called: 4D Bilateral Grid Slicing, BilateralGridCP4D

## Overview
Low-rank 4D Bilateral Grid is a parameterization of the 4D Bilateral Grid Slicing method, designed to approximate continuous color transformation functions (specifically affine matrices) as a function of 3D spatial coordinates $(x, y, z)$ and a 1D guidance dimension (grayscale intensity). It solves the memory bottleneck of storing a full 4D grid by applying Canonical Polyadic (CP) decomposition, allowing the representation of high-dimensional lookup tables with significantly fewer parameters. It builds on [3D Bilateral Grid Slicing](10529-3d-bilateral-grid-slicing.md) by extending the spatial domain from 2D to 3D and introducing low-rank factorization to manage the resulting 4D tensor size.

## How it works
The method represents the affine transformation matrix $M \in \mathbb{R}^{3 \times 4}$ for a given point $p=(x,y,z)$ and guidance value $g$ as the output of a bilinear interpolation over a low-rank 4D grid. The grid is decomposed into a set of factor tensors, which are sampled and combined during the forward pass.

### Inputs and Outputs
*   **Inputs**:
    *   `xyz`: Spatial coordinates of shape $(N, 3)$, normalized to $[-bound, bound]$.
    *   `rgb`: Input color values of shape $(N, 3)$, used to compute the guidance dimension.
*   **Outputs**:
    *   `affine_matrices`: Output affine transformation matrices of shape $(N, 3, 4)$. Each matrix transforms the input color (padded with 1) to the output color.

### Data Structures
The grid is stored as a set of learnable parameters derived from a CP decomposition of an initial identity tensor.

1.  **`fac_0`**: An `nn.Linear` layer weight tensor of shape $(12, \text{rank})$. This factor combines the sampled coefficients into the final 12-element affine matrix vector.
2.  **`fac_i`** (for $i=1 \dots 4$): Learnable parameters of shape $(1, \text{rank}, \text{grid\_size}_i, 1)$. These correspond to the spatial dimensions ($x, y, z$) and the guidance dimension ($w$). They are initialized from the CP factors and updated during training.

### Initialization
1.  Create an identity grid tensor `init_grids` of shape $(12, \text{grid\_W}, \text{grid\_Z}, \text{grid\_Y}, \text{grid\_X})$.
2.  Fill `init_grids` with the identity matrix structure repeated for each grid cell. Specifically, for each cell, the 12 values correspond to the flattened $3 \times 4$ identity-like matrix: $[1, 0, 0, 0, 0, 1, 0, 0, 0, 0, 1, 0]$.
3.  Add random noise to break symmetry:
    $$ \text{init\_grids} \leftarrow \text{init\_grids} + \mathcal{N}(0, \text{init\_noise\_scale}) $$
4.  Perform CP decomposition on `init_grids` with specified `rank`:
    $$ \text{init\_grids} \approx \sum_{r=1}^{\text{rank}} \mathbf{a}_r \circ \mathbf{b}_r \circ \mathbf{c}_r \circ \mathbf{d}_r \circ \mathbf{e}_r $$
    Let the factors be $\mathbf{A}, \mathbf{B}, \mathbf{C}, \mathbf{D}, \mathbf{E}$ with shapes $(12, \text{rank}), (\text{rank}, \text{grid\_W}), (\text{rank}, \text{grid\_Z}), (\text{rank}, \text{grid\_Y}), (\text{rank}, \text{grid\_X})$.
5.  Store $\mathbf{A}$ as the weight of `fac_0`.
6.  For each remaining factor $\mathbf{F} \in \{\mathbf{B}, \mathbf{C}, \mathbf{D}, \mathbf{E}\}$ corresponding to dimensions $w, z, y, x$:
    *   Transpose $\mathbf{F}$ to shape $(\text{rank}, \text{grid\_size})$.
    *   Reshape to $(1, \text{rank}, \text{grid\_size}, 1)$.
    *   Store as the initial value `fac_i_init` for the learnable parameter `fac_i`.

### Forward Pass
1.  **Normalize Coordinates**:
    $$ \text{xyz}_{norm} = \frac{\text{xyz}}{\text{bound}} $$
    This maps coordinates to $[-1, 1]$ assuming the input is within $[-\text{bound}, \text{bound}]$.

2.  **Compute Guidance Dimension**:
    *   If `learn_gray` is True: Pass `rgb` through an MLP:
        $$ \text{gray}_{raw} = \text{MLP}(\text{rgb}) $$
        $$ \text{gray} = \tanh(2.0 \cdot \text{gray}_{raw}) $$
    *   If `learn_gray` is False: Use BT.601 weights:
        $$ \text{gray} = 2.0 \cdot (\text{rgb} \cdot [0.299, 0.587, 0.114]^T) - 1.0 $$

3.  **Sample Factors**:
    Initialize `coef` as a tensor of ones with shape $(1, \text{rank}, 1, N)$.
    For $i$ from 1 to 4 (corresponding to dimensions $w, z, y, x$):
    *   Retrieve factor $\mathbf{F}_i = \text{fac}_i + \text{fac\_i\_init}$. Shape: $(1, \text{rank}, \text{grid\_size}_i, 1)$.
    *   Prepare sampling coordinates. For dimension $i$, the coordinate is the normalized value from `xyz` (or `gray` for $i=1$).
    *   Use `F.grid_sample` with `align_corners=True` and `padding_mode='border'` to sample $\mathbf{F}_i$ at the query coordinates.
    *   Accumulate the product:
        $$ \text{coef} \leftarrow \text{coef} \odot \text{sampled\_factor}_i $$

4.  **Compute Affine Matrices**:
    *   Squeeze and transpose `coef` from $(1, \text{rank}, 1, N)$ to $(N, \text{rank})$.
    *   Apply `fac_0` linear layer:
        $$ \text{affine\_flat} = \text{coef}_{reshaped} \cdot \text{fac\_0.weight}^T $$
    *   Reshape `affine_flat` to $(N, 3, 4)$.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class LowRank4DBilateralGrid(nn.Module):
    def __init__(self, grid_X=16, grid_Y=16, grid_Z=16, grid_W=8, rank=5, 
                 bound=2.0, learn_gray=True, init_noise_scale=1e-6):
        super().__init__()
        self.grid_X = grid_X
        self.grid_Y = grid_Y
        self.grid_Z = grid_Z
        self.grid_W = grid_W
        self.rank = rank
        self.bound = bound
        self.learn_gray = learn_gray
        
        # 1. Initialize identity grid
        # Shape: (12, grid_W, grid_Z, grid_Y, grid_X)
        init_grids = torch.zeros(12, grid_W, grid_Z, grid_Y, grid_X)
        # Fill with identity-like structure: [1,0,0,0, 0,1,0,0, 0,0,1,0]
        for i in range(12):
            if i < 3:
                init_grids[i, :, :, :, :] = (i == 0).float() # Diagonal elements
            # Off-diagonals are 0, which is already the case
            # Specifically: row 0: [1,0,0,0], row 1: [0,1,0,0], row 2: [0,0,1,0]
            # The rest are 0.
            # Correct initialization:
            init_grids[0, :, :, :, :] = 1.0
            init_grids[5, :, :, :, :] = 1.0
            init_grids[10, :, :, :, :] = 1.0
            
        # 2. Add noise
        init_grids = init_grids + torch.randn_like(init_grids) * init_noise_scale
        
        # 3. CP Decomposition
        # Using a simple CP decomposition implementation or library
        # Assuming parafac returns factors A, B, C, D, E
        # A: (12, rank), B: (rank, grid_W), C: (rank, grid_Z), D: (rank, grid_Y), E: (rank, grid_X)
        _, facs = parafac(init_grids.clone().detach(), rank=self.rank)
        A, B, C, D, E = facs
        
        # 4. Store factors
        # fac_0 is Linear(12, rank) -> output (..., 12) -> matmul with A.T -> (..., rank)
        # Wait, the source says: fac_0 weight shape (12, rank). 
        # Linear(in_features, out_features) computes x @ W.T + b.
        # If input is (N, rank) and weight is (12, rank), output is (N, 12).
        # So we need to transpose A to (12, rank) for the weight?
        # Source: "fac_0 weight shape (12, rank)". 
        # If we do coef @ A.T, where coef is (N, rank) and A is (12, rank), 
        # result is (N, 12). This matches.
        self.fac_0 = nn.Linear(self.rank, 12, bias=False)
        self.fac_0.weight.data = A.T # A is (12, rank), so A.T is (rank, 12). 
                                     # nn.Linear expects (out, in). So weight should be (12, rank).
                                     # Source says weight shape (12, rank). 
                                     # So we assign A directly if A is (12, rank).
        self.fac_0.weight.data = A 
        
        # Remaining factors
        self.num_facs = 4
        self.grid_sizes = [grid_W, grid_Z, grid_Y, grid_X]
        self.fac_params = nn.ParameterList()
        
        for i in range(self.num_facs):
            # facs[i+1] corresponds to B, C, D, E
            # Shape of facs[i+1] is (rank, grid_size)
            # Reshape to (1, rank, grid_size, 1)
            factor = facs[i+1].transpose(0, 1).reshape(1, self.rank, self.grid_sizes[i], 1)
            self.fac_params.append(nn.Parameter(factor.clone()))

    def forward(self, xyz, rgb):
        # xyz: (N, 3), rgb: (N, 3)
        
        # 1. Normalize xyz
        xyz_norm = xyz / self.bound
        
        # 2. Compute gray
        if self.learn_gray:
            # MLP placeholder
            gray = self.mlp_gray(rgb)
            gray = torch.tanh(2.0 * gray)
        else:
            # BT601
            weights = torch.tensor([0.299, 0.587, 0.114], device=rgb.device)
            gray = 2.0 * (rgb @ weights) - 1.0
            
        # 3. Sample factors
        # coef shape: (1, rank, 1, N)
        coef = torch.ones(1, self.rank, 1, xyz.shape[0], device=xyz.device)
        
        # Dimensions order: w, z, y, x
        # Coords: gray, xyz[:,2], xyz[:,1], xyz[:,0]
        coords_list = [gray, xyz_norm[:, 2], xyz_norm[:, 1], xyz_norm[:, 0]]
        
        for i in range(self.num_facs):
            fac = self.fac_params[i] # (1, rank, grid_size, 1)
            coord = coords_list[i]   # (N,)
            
            # Prepare coordinates for grid_sample
            # grid_sample expects (N, C, H, W) or (N, C, D, H, W) for 3D
            # Here we are sampling a 1D grid along one dimension, others are rank/1
            # fac shape: (1, rank, grid_size, 1)
            # We want to sample along dim 2 (grid_size)
            # Input to grid_sample should be (1, rank, 1, grid_size) ? No.
            # grid_sample(input, grid)
            # input: (N, C, H, W) for 2D. 
            # Our factor is (1, rank, grid_size, 1). 
            # We treat it as a 2D grid of height=grid_size, width=1? 
            # Or 1D grid?
            # Source says: "sample using F.grid_sample with coords[[i-1]]"
            # This implies a specific coordinate format.
            # Let's assume standard 2D grid_sample for simplicity, treating rank as C, grid_size as H.
            # But fac is (1, rank, grid_size, 1).
            # Let's reshape fac to (1, rank, grid_size, 1) -> (1, rank, 1, grid_size) for 2D sample?
            # Actually, grid_sample works on the last two dimensions.
            # If we view fac as (1, rank, grid_size, 1), the last two dims are (grid_size, 1).
            # This is a 1D grid.
            
            # Normalize coord to [-1, 1]
            # gray is already in [-1, 1]
            # xyz_norm is in [-1, 1]
            
            # Create grid coordinates
            # For 1D sampling, we need to expand dims
            # grid_sample(input, grid) where grid is (N, H_out, W_out, 2)
            # This is complex for 1D. 
            # Alternative: Use F.grid_sample on a 2D slice if grid_size is small?
            # Or use a custom 1D interpolation.
            # Given the source uses F.grid_sample, we assume it handles the dimensions.
            # Let's assume the factor is reshaped to (1, rank, 1, grid_size) and we sample at y=0, x=coord.
            
            fac_reshaped = fac.permute(0, 1, 3, 2) # (1, rank, 1, grid_size)
            
            # Grid coordinates: (N, 1, 1, 2) -> [x, y]
            # x is the normalized coordinate, y is 0 (since height is 1)
            # But wait, if height is 1, y is always 0.
            
            # Construct grid
            # coord is (N,)
            # We need to expand to (N, 1, 1, 2)
            grid_x = coord.unsqueeze(1).unsqueeze(1).unsqueeze(1) # (N, 1, 1, 1)
            grid_y = torch.zeros_like(grid_x) # (N, 1, 1, 1)
            grid = torch.cat([grid_x, grid_y], dim=3) # (N, 1, 1, 2)
            
            # Sample
            # input: (1, rank, 1, grid_size)
            # grid: (N, 1, 1, 2)
            # output: (N, rank, 1, 1)
            sampled = F.grid_sample(fac_reshaped, grid, align_corners=True, padding_mode='border')
            
            # Squeeze to (N, rank)
            sampled = sampled.squeeze(2).squeeze(3) # (N, rank)
            
            # Accumulate product
            # coef is (1, rank, 1, N) -> transpose to (N, rank, 1, 1) for broadcasting?
            # Let's keep coef as (1, rank, 1, N)
            # sampled is (N, rank) -> (1, rank, 1, N)
            sampled = sampled.permute(1, 0).unsqueeze(0).unsqueeze(2) # (1, rank, 1, N)
            
            coef = coef * sampled
            
        # 4. Compute affine matrices
        # coef: (1, rank, 1, N) -> (N, rank)
        coef_flat = coef.squeeze(0).squeeze(2).permute(1, 0) # (N, rank)
        
        # Apply fac_0: Linear(rank, 12)
        # output: (N, 12)
        affine_flat = self.fac_0(coef_flat) # (N, 12)
        
        # Reshape to (N, 3, 4)
        affine_matrices = affine_flat.view(-1, 3, 4)
        
        return affine_matrices
```

### Parameters
*   `grid_X` (int): Grid width. Default 16.
*   `grid_Y` (int): Grid height. Default 16.
*   `grid_Z` (int): Grid depth. Default 16.
*   `grid_W` (int): Grid guidance dimension. Default 8.
*   `rank` (int): Rank of the CP decomposition. Default 5.
*   `learn_gray` (bool): If True, an MLP learns the grayscale guidance; otherwise, BT.601 weights are used. Default True.
*   `bound` (float): Normalization bound for xyz coordinates. Default 2.0.
*   `init_noise_scale` (float): Noise scale for initialization. Default 1e-6.

### Edge Cases and Pitfalls
*   **Boundary Handling**: `padding_mode='border'` is used in `grid_sample` to handle coordinates outside the grid range, preventing NaNs or errors.
*   **Singularity**: Random noise is added during initialization to avoid singular factors in the CP decomposition.
*   **Coordinate Normalization**: Coordinates must be normalized by `bound` before sampling. If input coordinates exceed `[-bound, bound]`, the border padding will be used, which may lead to inaccurate transformations if the grid does not cover the full scene extent.

## Operating characteristics
*   **Memory Complexity**: $O(\text{rank} \cdot (\text{grid\_X} + \text{grid\_Y} + \text{grid\_Z} + \text{grid\_W}))$. This is significantly lower than the full grid size $O(\text{grid\_X} \cdot \text{grid\_Y} \cdot \text{grid\_Z} \cdot \text{grid\_W})$.
*   **Time Complexity**: $O(N \cdot \text{rank} \cdot \text{grid\_size})$ per point, dominated by the grid sampling operations.
*   **Strengths**: Efficient memory usage for high-dimensional grids; continuous representation; differentiable.
*   **Limitations**: Approximation error depends on rank; requires careful initialization; MLP overhead if `learn_gray` is True.

## Background
The Low-rank 4D Bilateral Grid is part of the `gsplat` library by nerfstudio [S1, unofficial]. It extends the concept of bilateral grids, originally introduced for image processing, to 3D spatial domains for neural rendering applications, specifically for color correction and tone mapping in 3D Gaussian Splatting.

## Sources
[S1] nerfstudio-project/gsplat, https://github.com/nerfstudio-project/gsplat, fetched 2026-10-27 (unofficial)
