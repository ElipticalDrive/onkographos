# 3D Bilateral Grid Slicing

Also called: BilateralGrid, bilateral guided radiance field processing

## Overview
3D Bilateral Grid Slicing is a method for approximating continuous, view-dependent color transformations (such as Image Signal Processing or ISP) by storing affine transformation matrices in a 3D grid indexed by 2D spatial coordinates $(x, y)$ and a 1D color guidance dimension (grayscale intensity) [S1, unofficial]. It solves the problem of learning a continuous function that maps pixel coordinates and colors to affine matrices, enabling efficient, differentiable color correction within radiance field representations [S1, unofficial]. It differs from sibling methods like Masked Wavelet Representation by focusing on explicit color-space transformations via grid sampling rather than compact spectral decomposition of the radiance field itself [S1, S7].

## How it works
**Inputs:**
- Input RGB image tensor of shape $(N, m, h, w, 3)$, where $N$ is the number of views, $m$ is the batch size per view, and $h, w$ are image height and width.
- Grid dimensions: width $W$, height $H$, and guidance dimension size $L$.

**Outputs:**
- Transformed RGB tensor of shape $(N, m, h, w, 3)$.

**Data Structures:**
The bilateral grid is stored as a learnable parameter tensor `grids` with shape $(N, 12, L, H, W)$ [S1, unofficial].
- $N$: Number of views (or grids).
- $12$: Number of coefficients per affine matrix (representing a $3 \times 4$ matrix).
- $L$: Size of the guidance dimension (grayscale intensity).
- $H$: Grid height.
- $W$: Grid width.

**Parameters:**
- `grid_X` (int): Grid width $W$. Default: 16 [S1, unofficial].
- `grid_Y` (int): Grid height $H$. Default: 16 [S1, unofficial].
- `grid_W` (int): Grid guidance dimension $L$. Default: 8 [S1, unofficial].

**Steps:**

1. **Initialize the Grid:**
   Initialize the `grids` tensor with identity affine matrices. Create a tensor of 12 floats representing the identity $3 \times 4$ affine matrix (diagonal elements 1.0 for the $3 \times 3$ rotation/scale/shear part, and 0.0 for the translation part). Repeat this vector $L \times H \times W$ times to fill the spatial and guidance dimensions. Reshape to $(1, 12, L, H, W)$ and then expand to $(N, 12, L, H, W)$ for $N$ views [S1, unofficial].

2. **Compute Grayscale Guidance:**
   Convert input RGB values to grayscale guidance $z$ using BT.601 weights. The formula is:
   $$ z = 2.0 \times (0.299 R + 0.587 G + 0.114 B) - 1.0 $$
   This maps the RGB color to the range $[-1, 1]$ for the guidance dimension [S1, unofficial].

3. **Construct Slicing Coordinates:**
   Rescale spatial coordinates $(x, y)$ from the image domain $[0, 1]$ to $[-1, 1]$ using:
   $$ x_{grid} = (x - 0.5) \times 2 $$
   $$ y_{grid} = (y - 0.5) \times 2 $$
   Concatenate the rescaled $x_{grid}$, $y_{grid}$, and the grayscale guidance $z$ along the last dimension to form `grid_xyz` of shape $(N, m, h, w, 3)$ [S1, unofficial].

4. **Sample Affine Matrices:**
   Sample the affine matrices from the `grids` tensor using `grid_xyz`. Use bilinear interpolation (`mode='bilinear'`) with `align_corners=True` and `padding_mode='border'`. This retrieves the 12 coefficients for each pixel [S1, unofficial].

5. **Reshape Coefficients:**
   Permute the sampled output from $(N, 12, m, h, w)$ to $(N, m, h, w, 12)$. Reshape the last dimension into $(3, 4)$ to form the affine transformation matrices $M$ for each pixel [S1, unofficial].

6. **Apply Affine Transformation:**
   Apply the affine transformation to the input RGB color. For each pixel, compute:
   $$ \mathbf{c}_{out} = M_{3 \times 3} \mathbf{c}_{in} + \mathbf{t}_{3 \times 1} $$
   where $\mathbf{c}_{in}$ is the input RGB vector, $M_{3 \times 3}$ is the first 3 columns of the affine matrix, and $\mathbf{t}_{3 \times 1}$ is the 4th column (translation) [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def initialize_bilateral_grid(N, L, H, W):
    """
    Initialize the bilateral grid with identity affine matrices.
    """
    # Identity 3x4 affine matrix: [1, 0, 0, 0, 0, 1, 0, 0, 0, 0, 1, 0]
    identity_coeffs = torch.tensor([1.0, 0.0, 0.0, 0.0,
                                    0.0, 1.0, 0.0, 0.0,
                                    0.0, 0.0, 1.0, 0.0])
    # Repeat for all grid cells
    grid_init = identity_coeffs.repeat(L * H * W)
    # Reshape to (1, 12, L, H, W)
    grid_init = grid_init.view(1, 12, L, H, W)
    # Expand to N views
    grids = grid_init.expand(N, -1, -1, -1, -1)
    return grids

def compute_grayscale_guidance(rgb):
    """
    Compute grayscale guidance z from RGB.
    rgb shape: (N, m, h, w, 3)
    """
    # BT.601 weights
    weights = torch.tensor([0.299, 0.587, 0.114], device=rgb.device)
    # Dot product along the last dimension (color channels)
    gray = torch.sum(rgb * weights, dim=-1, keepdim=True)
    # Scale to [-1, 1]
    z = 2.0 * gray - 1.0
    return z

def sample_bilateral_grid(grids, xy, z):
    """
    Sample affine matrices from the bilateral grid.
    grids shape: (N, 12, L, H, W)
    xy shape: (N, m, h, w, 2) in [-1, 1]
    z shape: (N, m, h, w, 1) in [-1, 1]
    """
    # Concatenate xy and z to form grid_xyz
    grid_xyz = torch.cat([xy, z], dim=-1) # Shape: (N, m, h, w, 3)
    
    # Reshape grids for grid_sample
    # grid_sample expects input of shape (N, C, H, W) or (N, C, D, H, W) for 3D
    # Here, we treat L, H, W as the spatial dimensions of the grid
    # grids shape: (N, 12, L, H, W)
    
    # grid_sample with align_corners=True and padding_mode='border'
    # Note: F.grid_sample expects coordinates in [-1, 1]
    sampled = F.grid_sample(grids, grid_xyz, mode='bilinear', align_corners=True, padding_mode='border')
    
    # sampled shape: (N, 12, m, h, w)
    return sampled

def apply_affine_transformation(sampled, rgb):
    """
    Apply the sampled affine transformation to RGB.
    sampled shape: (N, 12, m, h, w)
    rgb shape: (N, m, h, w, 3)
    """
    # Permute to (N, m, h, w, 12)
    sampled = sampled.permute(0, 2, 3, 4, 1)
    
    # Reshape to (N, m, h, w, 3, 4)
    sampled = sampled.view(sampled.shape[0], sampled.shape[1], sampled.shape[2], sampled.shape[3], 3, 4)
    
    # Extract 3x3 matrix and 3x1 translation
    # M_3x3: (N, m, h, w, 3, 3)
    # t_3x1: (N, m, h, w, 3, 1)
    M_3x3 = sampled[..., :3]
    t_3x1 = sampled[..., 3:]
    
    # Expand rgb to (N, m, h, w, 3, 1) for matrix multiplication
    rgb_expanded = rgb.unsqueeze(-1)
    
    # Apply transformation: c_out = M @ c_in + t
    rgb_out = torch.matmul(M_3x3, rgb_expanded) + t_3x1
    
    # Squeeze last dimension
    rgb_out = rgb_out.squeeze(-1)
    
    return rgb_out

def bilateral_grid_slicing(rgb, grids, image_h, image_w, grid_W, grid_H, grid_L):
    """
    Main function for 3D Bilateral Grid Slicing.
    """
    N, m, h, w, _ = rgb.shape
    
    # 1. Compute grayscale guidance
    z = compute_grayscale_guidance(rgb)
    
    # 2. Generate spatial coordinates
    # Create meshgrid for x and y in [0, 1]
    y_coords = torch.linspace(0, 1, image_h, device=rgb.device)
    x_coords = torch.linspace(0, 1, image_w, device=rgb.device)
    y_grid, x_grid = torch.meshgrid(y_coords, x_coords, indexing='ij')
    
    # Reshape to (1, 1, h, w, 1) and expand to (N, m, h, w, 1)
    x_grid = x_grid.unsqueeze(0).unsqueeze(0).expand(N, m, -1, -1).unsqueeze(-1)
    y_grid = y_grid.unsqueeze(0).unsqueeze(0).expand(N, m, -1, -1).unsqueeze(-1)
    
    # Rescale to [-1, 1]
    x_grid = (x_grid - 0.5) * 2
    y_grid = (y_grid - 0.5) * 2
    
    # Concatenate to form xy
    xy = torch.cat([x_grid, y_grid], dim=-1) # Shape: (N, m, h, w, 2)
    
    # 3. Sample from grid
    sampled = sample_bilateral_grid(grids, xy, z)
    
    # 4. Apply transformation
    rgb_out = apply_affine_transformation(sampled, rgb)
    
    return rgb_out
```

## Subtopics

### [Low-rank 4D Bilateral Grid](10587-low-rank-4d-bilateral-grid.md)
A parameterization of the 4D Bilateral Grid Slicing method, designed to approximate continuous color transformation functions (specifically affine matrices) as a function of 3D spatial coordinates $(x, y, z)$ and a 1D guidance dimension (grayscale intensity). It solves the memory bottleneck of storing a full 4D grid by applying Canonical Polyadic (CP) decomposition, allowing the representation of high-dimensional lookup tables with significantly fewer parameters. It builds on [3D Bilateral Grid Slicing](10529-3d-bilateral-grid-slicing.md) by extending the spatial domain from 2D to 3D and introducing low-rank factorization to manage the resulting 4D tensor size.

## Operating characteristics
- **Memory Complexity:** $O(N \cdot 12 \cdot L \cdot H \cdot W)$ for storing the grid parameters [S1, unofficial].
- **Time Complexity:** Dominated by `F.grid_sample`, which is $O(N \cdot m \cdot h \cdot w)$ for sampling [S1, unofficial].
- **Strengths:** Efficiently approximates continuous color transformations; differentiable; suitable for view-dependent color correction in radiance fields [S1, unofficial].
- **Limitations:** Memory usage scales linearly with grid dimensions and number of views; may not capture complex non-affine color transformations without higher-rank approximations [S1, unofficial].
- **When to use:** When approximating ISP or bilateral guided training in radiance field representations is required [S1, unofficial].
- **When not to use:** When memory is extremely constrained and low-rank approximations are insufficient, or when non-color transformations are needed [S1, unofficial].

## Background
The method is used in the research project 'Bilateral Guided Radiance Field Processing' for approximating ISP (Image Signal Processing) and bilateral guided training [S1, unofficial]. It builds on the concept of bilateral grids for color correction, extending it to 3D spatial coordinates for radiance field applications [S1, unofficial].

## Sources
[S1] nerfstudio-project/gsplat, https://github.com/nerfstudio-project/gsplat, fetched 2026-10-27 (unofficial)
