# BilateralGrid Class

Also called: Bilateral Grid Slicing

## Overview
The `BilateralGrid` class is a PyTorch module that implements the grid-based implementation of the cross-bilateral filter for efficient, differentiable color correction [S1, S2, unofficial]. It solves the problem of applying spatially and color-adaptive affine transformations to input images by storing transformation parameters in a sparse 5D tensor and retrieving them via trilinear interpolation [S2, unofficial]. It differs from standard bilateral filtering by operating on a learned or pre-computed grid of affine matrices rather than computing weights on the fly, enabling high-performance inference in differentiable rendering pipelines [S1, S2, unofficial].

## How it works
The method operates by initializing a 5D grid of affine transformation parameters, normalizing input coordinates into the grid's coordinate space, sampling the grid using bilinear interpolation, and applying the resulting affine transforms to the input colors [S2, unofficial].

### Inputs and Outputs
- **Inputs**:
  - `grids`: A 5-D tensor of shape `(N, 12, L, H, W)` containing the bilateral grid parameters [S1, S2, unofficial].
  - `rgb`: Input color tensor of shape `(..., 3)` representing RGB values [S2, unofficial].
  - `grid_idx` (optional): Indices to select specific grids from `N` [S2, unofficial].
- **Outputs**:
  - `rgb_out`: Transformed color tensor of shape `(..., 3)` [S2, unofficial].

### Data Structures
The bilateral grid is stored as a 5-D tensor with the following layout [S1, S2, unofficial]:
- **Shape**: `(N, 12, L, H, W)`
- **N**: Number of bilateral grids (e.g., number of views) [S1, S2, unofficial].
- **12**: Number of affine coefficients per cell, representing a flattened $3 \times 4$ affine transformation matrix [S1, S2, unofficial].
- **L**: Guidance dimension size (grayscale/luma) [S1, S2, unofficial].
- **H**: Grid height (spatial Y) [S1, S2, unofficial].
- **W**: Grid width (spatial X) [S1, S2, unofficial].

The module also maintains a buffer `rgb2gray_weight` defined as the tensor `[[0.299, 0.587, 0.114]]` for BT.601 RGB-to-gray conversion [S1, unofficial].

### Parameters
- `num` (int): The number of bilateral grids/views [S1, unofficial].
- `grid_X` (int): Grid width $W$. Default: 16 [S2, unofficial].
- `grid_Y` (int): Grid height $H$. Default: 16 [S2, unofficial].
- `grid_W` (int): Grid guidance dimension $L$. Default: 8 [S1, S2, unofficial].

### Initialization
The grid is initialized as an identity affine transformation repeated across all spatial and guidance dimensions [S2, unofficial]. The identity matrix parameters are flattened into a 12-element vector: `[1.0, 0, 0, 0, 0, 1.0, 0, 0, 0, 0, 1.0, 0]` [S2, unofficial].

### Steps
1. **Coordinate Rescaling**: Rescale spatial coordinates $xy$ from the range $[0, 1]$ to $[-1, 1]$ using the formula [S2, unofficial]:
   $$
   xy_{rescaled} = (xy - 0.5) \times 2
   $$
2. **Grayscale Conversion**: Convert input RGB values to grayscale guidance using BT.601 weights [S2, unofficial]:
   $$
   gray = (R \times 0.299 + G \times 0.587 + B \times 0.114) \times 2.0 - 1.0
   $$
   This maps the guidance value to the range $[-1, 1]$ [S2, unofficial].
3. **Grid Coordinate Formation**: Concatenate the rescaled spatial coordinates and the grayscale guidance to form the 3D slicing coordinates [S2, unofficial]:
   $$
   grid_{xyz} = [grid_{xy}, grid_z]
   $$
   where $grid_{xy}$ are the rescaled spatial coordinates and $grid_z$ is the grayscale guidance [S2, unofficial].
4. **Grid Sampling**: Sample the grid values at the slicing coordinates using bilinear interpolation with aligned corners and border padding [S2, unofficial]:
   $$
   affine_{mats} = \text{grid\_sample}(grids, grid_{xyz}, \text{mode}="bilinear", \text{align\_corners}=\text{True}, \text{padding\_mode}="border")
   $$
   The output shape is `(..., 12)` [S2, unofficial].
5. **Reshaping**: Reshape the sampled affine matrices from shape `(..., 12)` to `(..., 3, 4)` to represent $3 \times 4$ affine transformation matrices [S2, unofficial].
6. **Affine Transformation**: Apply the retrieved affine transformation to the input RGB color [S2, unofficial]:
   $$
   rgb_{out} = M_{0:3} \times rgb + M_{3}
   $$
   where $M$ is the $3 \times 4$ affine matrix, $M_{0:3}$ are the first three columns (linear transform), and $M_{3}$ is the fourth column (translation) [S2, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

class BilateralGrid(torch.nn.Module):
    def __init__(self, num=1, grid_X=16, grid_Y=16, grid_W=8):
        super().__init__()
        self.num = num
        self.grid_X = grid_X
        self.grid_Y = grid_Y
        self.grid_W = grid_W
        
        # Initialize grid with identity affine transformations
        # Identity 3x4 matrix flattened: [1,0,0, 0,1,0, 0,0,1, 0,0,0]
        identity_params = torch.tensor([1.0, 0, 0, 0, 0, 1.0, 0, 0, 0, 0, 1.0, 0])
        
        # Shape: (N, 12, L, H, W)
        self.grids = torch.nn.Parameter(
            identity_params.view(1, 12, 1, 1, 1).repeat(num, 1, grid_W, grid_Y, grid_X)
        )
        
        # BT.601 weights for RGB to Gray
        self.rgb2gray_weight = torch.tensor([[0.299, 0.587, 0.114]])

    def forward(self, rgb, grid_idx=None):
        """
        Args:
            rgb: Input color tensor of shape (..., 3) with values in [0, 1]
            grid_idx: Optional index to select specific grid from N. 
                      If None, assumes single grid or broadcasts.
        Returns:
            rgb_out: Transformed color tensor of shape (..., 3)
        """
        # 1. Grayscale Conversion
        # gray = (R * 0.299 + G * 0.587 + B * 0.114) * 2.0 - 1.0
        gray = torch.sum(rgb * self.rgb2gray_weight, dim=-1, keepdim=True) * 2.0 - 1.0
        
        # 2. Spatial Coordinate Rescaling
        # Assuming rgb has spatial dimensions H, W at the end.
        # We need to generate meshgrid for xy coordinates in [0, 1] range first.
        # Let's assume rgb shape is (B, H, W, 3) or (H, W, 3).
        # For generality, we extract spatial dims.
        # Note: The source implies xy comes from the spatial layout of the input image.
        
        # Get spatial dimensions
        # If rgb is (..., H, W, 3), we need to handle arbitrary batch dims.
        # Simplified for standard (H, W, 3) or (B, H, W, 3)
        if rgb.dim() == 3:
            H, W, _ = rgb.shape
            rgb_reshaped = rgb.view(-1, 3)
            batch_size = 1
        elif rgb.dim() == 4:
            B, H, W, _ = rgb.shape
            rgb_reshaped = rgb.view(-1, 3)
            batch_size = B
        else:
            raise ValueError("Unsupported input dimension")

        # Create spatial coordinates [0, 1]
        y_coords = torch.linspace(0, 1, H, device=rgb.device)
        x_coords = torch.linspace(0, 1, W, device=rgb.device)
        xx, yy = torch.meshgrid(x_coords, y_coords, indexing='xy')
        
        # Stack to form (H, W, 2)
        spatial_coords = torch.stack([xx, yy], dim=-1) # Shape: (H, W, 2)
        
        # Reshape spatial_coords to match rgb_reshaped batch
        # rgb_reshaped is (B*H*W, 3)
        # We need spatial_coords to be (B*H*W, 2)
        spatial_coords_flat = spatial_coords.view(-1, 2)
        
        # 3. Rescale spatial coordinates to [-1, 1]
        # xy_rescaled = (xy - 0.5) * 2
        xy_rescaled = (spatial_coords_flat - 0.5) * 2
        
        # 4. Form grid_xyz coordinates
        # grid_xyz = [grid_xy, grid_z] where grid_z is gray
        # gray is (B*H*W, 1)
        grid_xyz = torch.cat([xy_rescaled, gray], dim=-1) # Shape: (B*H*W, 3)
        
        # 5. Handle grid_idx if provided
        # If grid_idx contains a unique index, avoid tensor indexing to prevent copy
        # For simplicity, we assume grid_idx selects one grid or we process all.
        # The source mentions avoiding tensor indexing if unique.
        # Here we assume we sample from the first grid or broadcast if N=1.
        # If N > 1 and grid_idx is provided, we would index self.grids[grid_idx].
        # For this pseudocode, we assume N=1 or grid_idx is handled by caller.
        # Let's assume we use the first grid for demonstration if N=1.
        current_grids = self.grids[0] if self.num == 1 else self.grids[grid_idx]
        
        # Reshape current_grids for grid_sample
        # grid_sample expects (N, C, H, W) or (N, C, D, H, W) for 3D
        # Here we have (12, L, H, W) for a single grid instance.
        # We need to expand batch dim for grid_sample if necessary.
        # grid_sample works on (N, C, H, W) -> (N, C, H, W)
        # We have (B*H*W, 3) coordinates. We need to sample from (1, 12, L, H, W).
        # grid_sample expects coordinates in (-1, 1).
        
        # Reshape grids to (1, 12, L, H, W)
        grids_4d = current_grids.unsqueeze(0) # (1, 12, L, H, W)
        
        # grid_sample for 3D (D, H, W)
        # Input coordinates should be (N, H_out, W_out, D_out) or similar?
        # PyTorch grid_sample for 3D: input (N, C, D, H, W), grid (N, H_out, W_out, D_out, 3)
        # Our grid is (1, 12, L, H, W). L is D, H is H, W is W.
        # We want to sample at specific (z, y, x) points.
        # The grid_xyz is (B*H*W, 3) -> (x, y, z) or (grid_x, grid_y, grid_z)?
        # Source says: grid_xyz = [grid_xy, grid_z]. grid_xy is 2D.
        # So grid_xyz is (..., 3) where last dim is [x, y, z].
        # PyTorch grid_sample expects grid coordinates in the order [x, y, z] for 3D?
        # Actually, PyTorch 3D grid_sample expects grid of shape (N, H_out, W_out, D_out, 3)
        # with values in [-1, 1] corresponding to [x, y, z] normalized to input dims.
        
        # We need to reshape grid_xyz to match the output spatial dimensions.
        # The output spatial dimensions are H, W. The guidance dimension L is sampled implicitly?
        # No, we are sampling at specific (x, y, z) points.
        # The "slice" is defined by the (x, y, z) coordinates.
        # We need to construct a grid tensor for grid_sample.
        
        # Let's reshape grid_xyz to (B, H, W, 3)
        grid_xyz_3d = grid_xyz.view(batch_size, H, W, 3)
        
        # Sample
        # affine_mats shape: (B, H, W, 12)
        affine_mats = F.grid_sample(
            grids_4d, 
            grid_xyz_3d, 
            mode='bilinear', 
            align_corners=True, 
            padding_mode='border'
        )
        
        # Reshape affine_mats to (B*H*W, 12)
        affine_mats_flat = affine_mats.view(batch_size * H * W, 12)
        
        # Reshape to (B*H*W, 3, 4)
        affine_mats_3x4 = affine_mats_flat.view(-1, 3, 4)
        
        # Apply affine transform: rgb_out = M[0:3] * rgb + M[3]
        # rgb_reshaped is (B*H*W, 3)
        # affine_mats_3x4 is (B*H*W, 3, 4)
        # M[0:3] is (3, 3), M[3] is (3,)
        
        linear_part = torch.bmm(affine_mats_3x4[:, :3, :], rgb_reshaped.unsqueeze(-1)).squeeze(-1)
        translation_part = affine_mats_3x4[:, 3, :]
        
        rgb_out = linear_part + translation_part
        
        # Reshape back to original spatial dimensions
        if rgb.dim() == 3:
            return rgb_out.view(H, W, 3)
        else:
            return rgb_out.view(batch_size, H, W, 3)
```

### Edge Cases and Pitfalls
- **Precision**: The grid sampling uses bilinear interpolation, which may introduce smoothing artifacts if the grid resolution is too low [S2, unofficial].
- **Boundaries**: Border padding is used during grid sampling to handle coordinates outside the $[-1, 1]$ range [S2, unofficial].
- **Memory Overhead**: If `grid_idx` contains a unique index for all pixels from a single view, the implementation avoids tensor indexing to prevent data copy and extra memory overhead [S2, unofficial].

## Operating characteristics
- **Time Complexity**: The dominant operation is the grid sampling, which is $O(N \cdot H \cdot W)$ where $N$ is the batch size, and $H, W$ are spatial dimensions. The affine transformation is $O(N \cdot H \cdot W)$.
- **Memory Complexity**: The grid storage is $O(N \cdot 12 \cdot L \cdot H \cdot W)$. For default parameters ($N=1, L=8, H=16, W=16$), this is very small.
- **Performance**: Efficient for real-time color correction in differentiable rendering due to the sparse grid structure and GPU-accelerated interpolation [S1, S2, unofficial].
- **Strengths**: Differentiable, fast inference, compact representation [S1, S2, unofficial].
- **Limitations**: Limited by grid resolution; higher resolution increases memory usage [S2, unofficial].

## Background
The Bilateral Grid class is part of the grid-based implementation of the cross-bilateral filter, which is used in differentiable volume rendering and color correction [S1, S2, unofficial]. It originates from the work on efficient bilateral filtering using splatting and slicing [S1, S2, unofficial]. It is used in projects like Difix3D and AnySplat for feed-forward 3D Gaussian Splatting and color correction [S1, S2, unofficial].

## Sources
[S1] nv-tlabs/Difix3D, https://github.com/nv-tlabs/Difix3D, fetched 2026-10-26 (unofficial)
[S2] InternRobotics/AnySplat, https://github.com/InternRobotics/AnySplat, fetched 2026-10-27 (unofficial)
