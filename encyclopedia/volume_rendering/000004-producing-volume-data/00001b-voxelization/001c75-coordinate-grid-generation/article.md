# Coordinate Grid Generation

Also called: Meshgrid Setup, Pixel coordinate grid, Pixel Coordinate Generation, NDC Pixel Coordinate Generation, [-1, 1] coordinate grid

## Overview
Coordinate Grid Generation produces a structured set of spatial coordinates (typically in Normalized Device Coordinates, NDC, range $[-1, 1]$) that align with the discrete sampling grid of a volume or image [S2, S5, unofficial]. It solves the problem of mapping integer pixel or voxel indices to continuous physical or normalized space for neural field training, ray casting, and SDF evaluation [G]. It differs from its sibling [Rasterization-based Voxelization](908-rasterization-based-voxelization.md) by generating the *sample points* themselves rather than determining *which* voxels a primitive occupies, and from [3D Coordinate Grid Generation](2333-3d-coordinate-grid-generation.md) by primarily targeting 2D image-plane coordinates for ray origin generation, though it extends to 3D [G].

## How it works
**Inputs:**
- Image dimensions: Width $W$, Height $H$ (for 2D) or dimensions $D_x, D_y, D_z$ (for 3D) [S1, unofficial][S2, unofficial][S4, unofficial][S5, unofficial].
- Optional physical scale: `scale` (default 0.5) or `points_distance` (default 0.5) defining the physical extent or spacing [S3][S5, unofficial].
- Optional info channels: `info_size` (default 4 for RGB+Alpha) for cell data [S5, unofficial].

**Outputs:**
- 2D Case: A tensor of shape $[W \times H, 2]$ containing $(x, y)$ coordinates in NDC space $[-1, 1]$ [S2, unofficial][S4, unofficial].
- 3D Case: A tensor of shape $[S_x \times S_y \times S_z, 3]$ containing $(x, y, z)$ coordinates, optionally paired with a cell tensor of shape $[S_x, S_y, S_z, \text{info\_size}]$ [S5, unofficial].

**Data Structures:**
- **Grid Tensor:** A flattened 1D array of coordinates [S2, S5, unofficial]. For 2D, shape is $[N, 2]$ where $N=W \times H$ [S2, unofficial]. For 3D, shape is $[N, 3]$ where $N=S_x S_y S_z$ [S5, unofficial].
- **Cell Tensor (3D only):** A tensor storing attributes per voxel, initialized with `requires_grad=True` for differentiable rendering [S5, unofficial].

**Steps:**

1. **Generate Integer Indices:**
   Create 1D arrays of integer indices for each dimension [S4, unofficial][S5, unofficial].
   - For 2D: $x_{idx} \in [0, W-1]$, $y_{idx} \in [0, H-1]$ [S4, unofficial].
   - For 3D: $i \in [0, S_x-1]$, $j \in [0, S_y-1]$, $k \in [0, S_z-1]$ [S5, unofficial].

2. **Normalize to NDC $[-1, 1]$:**
   Map indices to the range $[-1, 1]$. The standard formula centers the grid on the origin [S4, unofficial]:
   $$
   x_{ndc} = \frac{x_{idx} - \frac{W-1}{2}}{\frac{W-1}{2}}
   $$
   $$
   y_{ndc} = \frac{y_{idx} - \frac{H-1}{2}}{\frac{H-1}{2}}
   $$
   Alternatively, using `torch.linspace` from $-1$ to $1$ with $W$ and $H$ steps achieves the same result [S2, unofficial][S3].

3. **Create Meshgrid:**
   Use a meshgrid function to combine the 1D coordinate arrays into a grid [S2, unofficial][S5, unofficial].
   - **Indexing:** Use `"ij"` indexing (row-major) which is standard in PyTorch/TensorFlow [S2, unofficial][S5, unofficial].
   - **Order:** For 2D, stack $y$ and $x$ (or $x$ and $y$ depending on convention) and reverse if necessary to match image layout $(x, y)$ [S3][S4, unofficial].

4. **Flatten:**
   Reshape the grid tensor from $[H, W, 2]$ (or $[S_z, S_y, S_x, 3]$) to $[H \times W, 2]$ (or $[S_z S_y S_x, 3]$) in row-major order [S2, unofficial][S4, unofficial].

5. **Physical Scaling (Optional):**
   If physical coordinates are needed instead of NDC, scale the coordinates by the `points_distance` or `scale` parameter [S3][S5, unofficial].
   $$
   x_{phys} = x_{ndc} \times \text{scale}
   $$

6. **Centering (Optional):**
   For 3D grids, explicitly center the grid by subtracting $\lceil D/2 \rceil - 1$ from each coordinate index before scaling [S5, unofficial].

7. **Negation (Optional):**
   Some implementations negate the final grid coordinates to align with specific camera coordinate systems (e.g., right-handed vs left-handed) [S1, unofficial][S4, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np

def generate_2d_coordinate_grid(width: int, height: int, scale: float = 1.0, negate: bool = True):
    """
    Generates a 2D coordinate grid in NDC space [-1, 1].
    
    Args:
        width: Image width W.
        height: Image height H.
        scale: Physical scale factor (default 1.0 for pure NDC).
        negate: Whether to negate the final coordinates [S1, unofficial][S4, unofficial].
        
    Returns:
        Tensor of shape [W*H, 2] with coordinates [x, y].
    """
    # Step 1: Generate integer indices
    x_idx = torch.arange(width, dtype=torch.float32)
    y_idx = torch.arange(height, dtype=torch.float32)
    
    # Step 2: Normalize to [-1, 1]
    # Formula: (idx - center) / center
    x_ndc = (x_idx - (width - 1) / 2) / ((width - 1) / 2)
    y_ndc = (y_idx - (height - 1) / 2) / ((height - 1) / 2)
    
    # Step 3: Create Meshgrid with 'ij' indexing
    # torch.meshgrid returns (X_grid, Y_grid) where X_grid varies along columns, Y_grid along rows
    x_grid, y_grid = torch.meshgrid(x_ndc, y_ndc, indexing='ij')
    
    # Step 4: Stack and Flatten
    # Stack along last dimension to get [H, W, 2]
    grid = torch.stack([x_grid, y_grid], dim=-1)
    
    # Flatten to [W*H, 2]
    grid_flat = grid.view(-1, 2)
    
    # Step 5: Physical Scaling (if scale != 1.0)
    if scale != 1.0:
        grid_flat = grid_flat * scale
        
    # Step 6: Negation (if required by convention)
    if negate:
        grid_flat = -grid_flat
        
    return grid_flat

def generate_3d_coordinate_grid(sx: int, sy: int, sz: int, points_distance: float = 0.5, info_size: int = 4):
    """
    Generates a 3D coordinate grid for SDF evaluation or voxel attributes.
    
    Args:
        sx, sy, sz: Grid dimensions.
        points_distance: Physical spacing between points.
        info_size: Number of channels per cell (e.g., RGB + Alpha).
        
    Returns:
        grid_coords: Tensor of shape [sx*sy*sz, 3].
        grid_cells: Tensor of shape [sx, sy, sz, info_size].
    """
    # Step 1: Create index arrays
    x_idx = torch.arange(sx, dtype=torch.float32)
    y_idx = torch.arange(sy, dtype=torch.float32)
    z_idx = torch.arange(sz, dtype=torch.float32)
    
    # Step 2: Generate meshgrid of indices with 'ij' indexing
    x_grid, y_grid, z_grid = torch.meshgrid(x_idx, y_idx, z_idx, indexing='ij')
    
    # Step 3: Stack meshgrid components
    grid_indices = torch.stack([x_grid, y_grid, z_grid], dim=-1) # Shape [sx, sy, sz, 3]
    
    # Step 4: Center the grid
    # Subtract ceil(dim/2) - 1 to center around 0
    center_x = np.ceil(sx / 2) - 1
    center_y = np.ceil(sy / 2) - 1
    center_z = np.ceil(sz / 2) - 1
    
    grid_centered = grid_indices - torch.tensor([center_x, center_y, center_z], dtype=torch.float32)
    
    # Step 5: Scale coordinates by points_distance
    grid_scaled = grid_centered * points_distance
    
    # Step 6: Reshape to flat list of coordinates
    grid_coords = grid_scaled.view(-1, 3)
    
    # Step 7: Initialize grid_cells tensor
    # Shape [sx, sy, sz, info_size], requires gradients for training
    grid_cells = torch.zeros(sx, sy, sz, info_size, requires_grad=True)
    
    return grid_coords, grid_cells
```

**Edge Cases and Pitfalls:**
- **Even vs Odd Dimensions:** The normalization formula $\frac{x - (W-1)/2}{(W-1)/2}$ ensures the center pixel (at index $(W-1)/2$) maps to 0 [S4, unofficial].
- **Indexing Convention:** `torch.meshgrid` defaults to `"ij"` [S6, unofficial].
- **Precision:** Using `float32` is standard, but for very large grids, `float64` may be needed to prevent precision loss in coordinate values [G].

## Subtopics

### [3D Coordinate Grid Generation](2333-3d-coordinate-grid-generation.md)
Generates a 3D grid of sample points within a voxel space for Signed Distance Function (SDF) evaluation. It produces a set of 3D points forming a regular grid within a unit cube, typically excluding boundary regions to ensure samples lie strictly inside the volume. It differs from its parent [Coordinate Grid Generation](1273-coordinate-grid-generation.md) by explicitly mapping integer grid indices to continuous 3D coordinates in world space (often $[-1, 1]$) using voxel size and origin parameters, rather than generating normalized 2D image coordinates [G].

### Meshgrid XY
Mimics `np.meshgrid(..., indexing="xy")` in PyTorch, as `torch.meshgrid` only allows `"ij"` indexing [S6, unofficial]. It is chosen when compatibility with NumPy's coordinate layout is required [S6, unofficial]. The implementation calls `torch.meshgrid(tensor1, tensor2)` and returns `ii.transpose(-1, -2), jj.transpose(-1, -2)` [S6, unofficial].

## Operating characteristics
- **Time Complexity:** $O(W \times H)$ for 2D and $O(S_x \times S_y \times S_z)$ for 3D, as every grid point must be computed [S2, unofficial][S5, unofficial].
- **Memory Complexity:** $O(W \times H \times D)$ where $D$ is the dimensionality (2 or 3) [S2, unofficial]. For 3D with cell attributes, $O(S_x S_y S_z \times \text{info\_size})$ [S5, unofficial].
- **Performance:** Highly parallelizable on GPU using tensor operations. Generation is typically instantaneous for resolutions up to $1024^2$ or $256^3$ on modern GPUs [G].
- **Strengths:** Provides exact, differentiable coordinate mappings essential for neural rendering and SDF optimization [S2, unofficial][S5, unofficial].
- **Limitations:** Memory usage grows cubically for 3D grids, limiting resolution for dense volumes [G].

## Background
Coordinate grid generation is a fundamental preprocessing step in neural radiance fields (NeRF) and implicit surface representations [S1, unofficial][S2, unofficial]. It originated with the need to sample input coordinates for multi-layer perceptrons (MLPs) that map spatial positions to color and density [S1, unofficial][S2, unofficial]. The $[-1, 1]$ NDC convention is widely adopted in computer graphics for camera space normalization [S4, unofficial].

## Sources
[S1] omkarchittar/Neural_Surfaces, https://github.com/omkarchittar/Neural_Surfaces, fetched 2026-10-26 (unofficial)
[S2] Yonghao-Lee/differentiable-rendering, https://github.com/Yonghao-Lee/differentiable-rendering, fetched 2026-10-26 (unofficial)
[S3] learning3d/assignment3, https://github.com/learning3d/assignment3, fetched 2026-10-26
[S4] samirangode/Volume-Rendering-and-Neural-Radiance-Fields, https://github.com/samirangode/Volume-Rendering-and-Neural-Radiance-Fields, fetched 2026-10-26 (unofficial)
[S5] DanJbk/Plenoxels, https://github.com/DanJbk/Plenoxels, fetched 2026-10-27 (unofficial)
[S6] rigved-sanku/NeRF-Implementation-for-Efficient-3D-Scene-Reconstruction, https://github.com/rigved-sanku/NeRF-Implementation-for-Efficient-3D-Scene-Reconstruction, fetched 2026-10-27 (unofficial)
