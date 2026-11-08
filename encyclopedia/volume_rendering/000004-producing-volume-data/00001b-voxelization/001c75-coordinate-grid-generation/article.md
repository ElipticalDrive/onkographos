# Coordinate Grid Generation

Also called: Meshgrid Setup, Pixel coordinate grid, Pixel Coordinate Generation, NDC Pixel Coordinate Generation, [-1, 1] coordinate grid

## Overview
Coordinate Grid Generation produces a regular lattice of sample points within a specified spatial domain, primarily to provide input coordinates for Signed Distance Function (SDF) evaluation or density field sampling in neural rendering pipelines [S1, S2, unofficial]. It differs from its sibling [Rasterization-based Voxelization](908-rasterization-based-voxelization.md), which maps geometry to occupancy via projection, by explicitly generating continuous coordinate tensors (often in Normalized Device Coordinates) rather than discrete integer indices [S2, unofficial]. It serves as the foundational sampling mechanism for methods like [Coarse-to-Fine SDF Sampling for Mesh Creation](3495-coarse-to-fine-sdf-sampling-for-mesh-creation.md) and [Grid-based Density Sampling with Blue Noise](2608-grid-based-density-sampling-with-blue-noise.md) [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Image dimensions ($W, H$) or grid resolution ($s_x, s_y, s_z$); spatial bounds (typically $[-1, 1]$ or physical spacing); device (CPU/GPU).
*   **Outputs:** A tensor of coordinates. For 2D, shape $[W \times H, 2]$ or $[H, W, 2]$; for 3D, shape $[s_x \times s_y \times s_z, 3]$ or $[s_x, s_y, s_z, 3]$ [S2, S5, unofficial].

**Data Structures:**
*   **Coordinate Tensor:** A dense tensor where each row (or element) represents a point in space. In 2D, it matches the flattened layout of an image (row-major) [S2, unofficial].
*   **Grid Cells (Optional):** In some implementations (e.g., Plenoxels), a separate learnable tensor `grid_cells` of shape $[s_x, s_y, s_z, \text{info\_size}]$ is initialized alongside coordinates, where `info_size` includes channels like RGB and alpha [S5, unofficial].

**Steps:**
1.  **Generate 1D Axes:** Create 1D tensors for each dimension.
    *   For image-based 2D grids, generate pixel indices $x \in [0, W-1]$ and $y \in [0, H-1]$ [S4, unofficial].
    *   For general 3D grids, generate indices using `arange` or `linspace` over the desired resolution $s$ [S5, unofficial].
2.  **Normalize to NDC (for 2D/Image):** Convert pixel indices to Normalized Device Coordinates (NDC) in the range $[-1, 1]$.
    *   Formula:
        $$ x_{ndc} = \frac{x_{idx} - \frac{W-1}{2}}{\frac{W-1}{2}} $$
        $$ y_{ndc} = \frac{y_{idx} - \frac{H-1}{2}}{\frac{H-1}{2}} $$
        where $x_{idx} \in [0, W-1]$ and $y_{idx} \in [0, H-1]$ [S4, unofficial].
    *   Alternatively, using linear spacing: $x = \text{linspace}(0, W-1, W)$, then normalize [S4, unofficial].
3.  **Create Meshgrid:**
    *   Use `meshgrid` to combine the 1D axes.
    *   **Indexing Convention:** Standard `meshgrid` often uses `'xy'` indexing (first output varies along columns, second along rows). However, many neural rendering libraries (like PyTorch) default to `'ij'` indexing (first output varies along rows, second along columns) [S2, S6, unofficial].
    *   If `'xy'` indexing is required but the library defaults to `'ij'`, transpose the resulting grids [S6, unofficial].
    *   Stack the grid components along the last dimension to form a coordinate tensor of shape $[H, W, 2]$ or $[s_x, s_y, s_z, 3]$ [S1, unofficial][S3].
4.  **Reshape and Flatten:**
    *   Reshape the grid to $[-1, \text{dim}]$ to create a list of points suitable for batched neural network input [S2, S4, unofficial].
    *   For 2D, the order typically matches the flattened image layout (row-major) [S2, unofficial].
5.  **Centering and Scaling (for 3D/Physical):**
    *   If generating a physical grid rather than NDC, center the coordinates by subtracting $\lceil \text{dim}/2 \rceil - 1$ from each coordinate index [S5, unofficial].
    *   Scale by `points_distance` (default 0.5) to set physical spacing [S5, unofficial].
6.  **Negation (Optional):**
    *   Some implementations negate the final coordinate tensor (e.g., `return -xy_grid`) to align with specific camera coordinate systems or SDF conventions [S1, unofficial][S3].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np

def generate_2d_ndc_grid(W, H, device='cpu', negate=False):
    """
    Generates a 2D coordinate grid in NDC [-1, 1] for an image of size WxH.
    Returns shape [W*H, 2].
    """
    # Step 1: Generate pixel indices
    x = torch.linspace(0, W - 1, W, device=device)
    y = torch.linspace(0, H - 1, H, device=device)
    
    # Step 2: Normalize to [-1, 1]
    # Formula: (idx - center) / center
    x_ndc = (x - (W - 1) / 2) / ((W - 1) / 2)
    y_ndc = (y - (H - 1) / 2) / ((H - 1) / 2)
    
    # Step 3: Create meshgrid
    # Note: torch.meshgrid defaults to 'ij' indexing.
    # To match image layout (x varies along width, y along height),
    # we often need to swap or transpose depending on the final reshape.
    # S4 suggests stacking y, x then reversing order to (x, y).
    grid_y, grid_x = torch.meshgrid(y_ndc, x_ndc, indexing='ij')
    
    # Stack to [H, W, 2]
    grid = torch.stack((grid_x, grid_y), dim=-1)
    
    # Step 4: Flatten to [W*H, 2]
    # Row-major order matches flattened image
    xy_grid = grid.reshape(-1, 2)
    
    # Step 5: Negate if required by convention
    if negate:
        xy_grid = -xy_grid
        
    return xy_grid

def generate_3d_physical_grid(sx, sy, sz, points_distance=0.5, device='cpu'):
    """
    Generates a 3D coordinate grid centered at origin with physical spacing.
    Returns shape [sx*sy*sz, 3].
    """
    # Step 1: Create index arrays
    x = torch.arange(sx, device=device)
    y = torch.arange(sy, device=device)
    z = torch.arange(sz, device=device)
    
    # Step 2: Meshgrid with 'ij' indexing
    xx, yy, zz = torch.meshgrid(x, y, z, indexing='ij')
    
    # Step 3: Stack and reshape to [sx, sy, sz, 3]
    grid_3d = torch.stack((xx, yy, zz), dim=-1)
    
    # Step 4: Center the grid
    # Subtract ceil(dim/2) - 1 from each coordinate
    center_x = torch.ceil(torch.tensor(sx / 2)).item() - 1
    center_y = torch.ceil(torch.tensor(sy / 2)).item() - 1
    center_z = torch.ceil(torch.tensor(sz / 2)).item() - 1
    
    grid_3d = grid_3d - torch.tensor([center_x, center_y, center_z], device=device)
    
    # Step 5: Scale by points_distance
    grid_3d = grid_3d * points_distance
    
    # Step 6: Flatten
    grid_coords = grid_3d.reshape(-1, 3)
    
    return grid_coords
```

**Parameters:**
*   $W, H$: Image width and height. Typical values: 128, 256, 512 [S1, unofficial][S3].
*   $s_x, s_y, s_z$: Grid resolution per dimension. Default: 128 [S3].
*   `points_distance`: Physical spacing between grid points. Default: 0.5 [S5, unofficial].
*   `info_size`: Number of channels per cell (e.g., 4 for RGB+Alpha). Default: 4 [S5, unofficial].
*   `negate`: Boolean flag to negate coordinates. Default: False (but used in some pipelines) [S1, unofficial][S3].

**Edge Cases and Pitfalls:**
*   **Indexing Mismatch:** `torch.meshgrid` uses `'ij'` indexing by default, while `numpy.meshgrid` uses `'xy'`. Using the wrong indexing leads to transposed coordinates, causing artifacts in rendering [S6, unofficial].
*   **Boundary Precision:** When mapping to NDC, ensure the center calculation uses $(W-1)/2$ to keep the center pixel at 0.0, not $W/2$, which would shift the grid by half a pixel [S4, unofficial].
*   **Memory:** Flattening a high-resolution 3D grid (e.g., $256^3$) creates a large tensor ($16M$ points). Ensure GPU memory is sufficient or use chunked processing [S5, unofficial].

## Subtopics

### [3D Coordinate Grid Generation](2333-3d-coordinate-grid-generation.md)
Produces a regular 3D lattice of sample points within a specified voxel space, primarily to provide input coordinates for Signed Distance Function (SDF) evaluation or density field sampling in neural rendering pipelines. It differs from the parent [Coordinate Grid Generation](1273-coordinate-grid-generation.md), which targets 2D image-plane layouts, by explicitly handling the third dimension and supporting spatial bounds and step sizes for arbitrary voxel resolutions. It serves as the foundational sampling mechanism for methods like [Coarse-to-Fine SDF Sampling for Mesh Creation](3495-coarse-to-fine-sdf-sampling-for-mesh-creation.md) and [Grid-based Density Sampling with Blue Noise](2608-grid-based-density-sampling-with-blue-noise.md).

### [CT Slice Voxel Construction](8351-ct-slice-voxel-construction.md)
Constructs a 3D grid cell (voxel) from two adjacent 2D CT slices by calculating the 3D coordinates of its 8 vertices and retrieving the scalar values (Hounsfield units) for each vertex. It differs from [3D Coordinate Grid Generation](2333-3d-coordinate-grid-generation.md) by explicitly handling the discrete nature of medical imaging data, using slice-specific metadata (pixel spacing, origin) rather than a normalized [-1, 1] domain, and mapping values directly from the source image arrays.

### Meshgrid XY
Mimics `np.meshgrid(..., indexing="xy")` in PyTorch, as `torch.meshgrid` only allows `"ij"` indexing by default [S6, unofficial].
**Steps:**
1.  Call `torch.meshgrid(tensor1, tensor2)` [S6, unofficial].
2.  Return `ii.transpose(-1, -2), jj.transpose(-1, -2)` to swap the axes and achieve `'xy'` behavior [S6, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the total number of points ($W \times H$ or $s_x \times s_y \times s_z$). The generation is vectorized and highly parallel on GPU [S2, S5, unofficial].
*   **Memory Complexity:** $O(N \times D)$ where $D$ is the dimensionality (2 or 3). For 3D grids, memory usage scales cubically with resolution [S5, unofficial].
*   **Performance:** Generating a $128 \times 128$ 2D grid is instantaneous on modern GPUs. A $256^3$ 3D grid requires significant VRAM (~16M points $\times$ 3 floats $\times$ 4 bytes $\approx$ 192 MB) but generation time remains low due to vectorization [S3][S5, unofficial].
*   **Strengths:** Provides exact, differentiable coordinates for neural field training; simple to implement using standard tensor operations [S2, S5, unofficial].
*   **Limitations:** Does not account for geometry; it generates a uniform grid regardless of whether points lie inside or outside the object. This leads to wasted computation on empty space, unlike sparse voxel methods [S1, unofficial].

## Background
Coordinate Grid Generation is a fundamental utility in neural rendering and implicit surface reconstruction. It originated with the rise of Neural Radiance Fields (NeRF) [S3] and subsequent implicit surface methods like NeuS [S1, unofficial], where dense sampling of a 3D volume is required to train the neural network to predict density and color. The use of NDC coordinates $[-1, 1]$ is standard in computer graphics for normalizing view-dependent data [S4, unofficial].

## Sources
[S1] omkarchittar/Neural_Surfaces, https://github.com/omkarchittar/Neural_Surfaces, fetched 2026-10-26 (unofficial)
[S2] Yonghao-Lee/differentiable-rendering, https://github.com/Yonghao-Lee/differentiable-rendering, fetched 2026-10-26 (unofficial)
[S3] learning3d/assignment3, https://github.com/learning3d/assignment3, fetched 2026-10-26
[S4] samirangode/Volume-Rendering-and-Neural-Radiance-Fields, https://github.com/samirangode/Volume-Rendering-and-Neural-Radiance-Fields, fetched 2026-10-26 (unofficial)
[S5] DanJbk/Plenoxels, https://github.com/DanJbk/Plenoxels, fetched 2026-10-27 (unofficial)
[S6] rigved-sanku/NeRF-Implementation-for-Efficient-3D-Scene-Reconstruction, https://github.com/rigved-sanku/NeRF-Implementation-for-Efficient-3D-Scene-Reconstruction, fetched 2026-10-27 (unofficial)
