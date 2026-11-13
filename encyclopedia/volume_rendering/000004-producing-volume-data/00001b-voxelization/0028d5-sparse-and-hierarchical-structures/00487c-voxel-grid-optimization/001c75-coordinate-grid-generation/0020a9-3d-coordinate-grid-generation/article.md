# 3D Coordinate Grid Generation

Also called: Voxel Grid Point Generation, Voxel Grid Sampling, Voxel Grid Sampling for Mesh Extraction

## Overview
3D Coordinate Grid Generation produces a structured set of 3D sample points within a defined voxel space, primarily for evaluating Signed Distance Functions (SDFs) or density fields in neural volume rendering and mesh extraction [S1, S2, unofficial]. It differs from its parent [Coordinate Grid Generation](1273-coordinate-grid-generation.md) by explicitly handling 3D volumetric bounds and voxel centers rather than just 2D image-plane coordinates, and from [CT Slice Voxel Construction](8351-ct-slice-voxel-construction.md) by generating a full 3D tensor of centers and indices rather than processing 2D slices [S2, unofficial].

## How it works
The method generates a regular 3D grid of sample points. Two primary implementations exist in the sources: one based on a fixed unit cube resolution [S3, unofficial], and another based on arbitrary spatial bounds and step sizes [S2, unofficial]. The latter is more general and is detailed below.

### Inputs and Outputs
**Inputs:**
- `xmin`, `xmax`, `xstep`: Floats defining the X-axis bounds and step size.
- `ymin`, `ymax`, `ystep`: Floats defining the Y-axis bounds and step size.
- `zmin`, `zmax`, `zstep`: Floats defining the Z-axis bounds and step size.
- `voxel_origin` (Optional): List of 3 floats for the origin, default `[-1, -1, -1]` [S1, unofficial].
- `voxel_size` (Optional): Float size of each voxel, calculated as `2.0 / (N - 1)` where N is resolution [S1, unofficial].
- `transform` (Optional): Boolean for applying a point transformation, default `False` [S1, unofficial].

**Outputs:**
- `grid_centers`: A tensor of shape `(Z, Y, X, 3)` containing float voxel center coordinates [S2, unofficial].
- `grid_indices`: A tensor of shape `(Z, Y, X, 3)` containing integer voxel grid indices [S2, unofficial].

### Data Structures
- **Indices Tensor**: Integer tensor of shape `(Z, Y, X, 3)`. Each element `[z, y, x, :]` contains the integer indices `(z_idx, y_idx, x_idx)` for the voxel at that position.
- **Centers Tensor**: Float tensor of shape `(Z, Y, X, 3)`. Each element `[z, y, x, :]` contains the world coordinates `(x_center, y_center, z_center)` of the voxel center.

### Steps
1. **Calculate Bin Counts**:
   Compute the number of voxels along each axis:
   $$
   x_{bins} = \lfloor (x_{max} - x_{min}) / x_{step} \rfloor
   $$
   $$
   y_{bins} = \lfloor (y_{max} - y_{min}) / y_{step} \rfloor
   $$
   $$
   z_{bins} = \lfloor (z_{max} - z_{min}) / z_{step} \rfloor
   $$
   [S2, unofficial]

2. **Generate Integer Indices**:
   Create 1D arrays of indices for each axis:
   $$
   x_{idx} = \text{arange}(x_{bins})
   $$
   $$
   y_{idx} = \text{arange}(y_{bins})
   $$
   $$
   z_{idx} = \text{arange}(z_{bins})
   $$
   [S2, unofficial]

3. **Create Meshgrid of Indices**:
   Stack the indices into a 3D grid. The order of axes in the meshgrid determines the layout. Using `indexing="ij"` (row-major):
   $$
   grid_{idx} = \text{stack}(\text{meshgrid}(z_{idx}, y_{idx}, x_{idx}, \text{indexing}="ij"), \text{dim}=-1)
   $$
   The resulting shape is `(z_bins, y_bins, x_bins, 3)`.
   [S2, unofficial]

4. **Compute Float Centers**:
   Calculate the center coordinate for each voxel by adding half a step to the minimum bound:
   $$
   x_{centers} = x_{min} + (x_{idx} + 0.5) \times x_{step}
   $$
   $$
   y_{centers} = y_{min} + (y_{idx} + 0.5) \times y_{step}
   $$
   $$
   z_{centers} = z_{min} + (z_{idx} + 0.5) \times z_{step}
   $$
   [S2, unofficial]

5. **Create Meshgrid of Centers**:
   Stack the center coordinates into a 3D grid:
   $$
   grid_{centers} = \text{stack}(\text{meshgrid}(x_{centers}, y_{centers}, z_{centers}, \text{indexing}="ij"), \text{dim}=-1)
   $$
   Note: The source uses `indexing="ij"` for indices but lists `x_centers, y_centers, z_centers` in the meshgrid call. Standard `meshgrid` with `indexing="ij"` expects the first argument to vary along the first dimension (Z). If the output shape is `(Z, Y, X, 3)`, the meshgrid arguments should correspond to the axes. The source code snippet `torch.meshgrid(x_centers, y_centers, z_centers, indexing="ij")` produces a shape `(X, Y, Z, 3)` if not careful, but the description says `(Z, Y, X, 3)`. Assuming standard behavior where `meshgrid` returns arrays matching the input order, and `stack` adds the last dim, the layout depends on the specific library. However, the text explicitly states the layout is `(Z, Y, X, 3)`. To achieve this with `meshgrid`, one typically passes `z, y, x` or transposes. The source code says `meshgrid(x_centers, y_centers, z_centers, indexing="ij")`. In PyTorch, `meshgrid` with `indexing='ij'` returns arrays where the first output varies along the first dimension. If inputs are `x, y, z`, the output shape is `(len(x), len(y), len(z))`. This contradicts the stated layout `(Z, Y, X, 3)`. Given the ambiguity, we follow the explicit layout description: `(Z, Y, X, 3)`. This implies the meshgrid should be constructed such that the first dimension corresponds to Z.
   [S2, unofficial]

6. **Apply Transformation (Optional)**:
   If `transform` is true, apply a transformation matrix to the points.
   [S1, unofficial]

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def generate_3d_coordinate_grid(xmin, xmax, xstep, ymin, ymax, ystep, zmin, zmax, zstep):
    """
    Generates a 3D grid of voxel centers and indices.
    
    Args:
        xmin, xmax, xstep: Bounds and step for X axis.
        ymin, ymax, ystep: Bounds and step for Y axis.
        zmin, zmax, zstep: Bounds and step for Z axis.
        
    Returns:
        grid_centers: Tensor of shape (Z, Y, X, 3) with float coordinates.
        grid_indices: Tensor of shape (Z, Y, X, 3) with integer indices.
    """
    # Step 1: Calculate bin counts
    x_bins = int((xmax - xmin) / xstep)
    y_bins = int((ymax - ymin) / ystep)
    z_bins = int((zmax - zmin) / zstep)
    
    # Step 2: Generate integer indices
    x_idx = torch.arange(x_bins)
    y_idx = torch.arange(y_bins)
    z_idx = torch.arange(z_bins)
    
    # Step 3: Create meshgrid for indices
    # To get shape (Z, Y, X, 3), we need to be careful with meshgrid order.
    # torch.meshgrid with indexing='ij' returns arrays where the first arg varies along dim 0.
    # If we want dim 0 to be Z, we should pass z_idx first? 
    # However, the source code snippet says: torch.meshgrid(z_idx, y_idx, x_idx, indexing="ij")
    # This would produce shape (Z, Y, X). Let's follow the source code logic for indices.
    grid_idx_parts = torch.meshgrid(z_idx, y_idx, x_idx, indexing="ij")
    grid_indices = torch.stack(grid_idx_parts, dim=-1) # Shape: (Z, Y, X, 3)
    
    # Step 4: Compute float centers
    # x_centers = xmin + (x_idx + 0.5) * xstep
    # The source code snippet says:
    # x_centers = xmin + (x_idx + 0.5) * xstep
    # y_centers = ymin + (y_idx + 0.5) * ystep
    # z_centers = zmin + (z_idx + 0.5) * zstep
    x_centers = xmin + (x_idx.float() + 0.5) * xstep
    y_centers = ymin + (y_idx.float() + 0.5) * ystep
    z_centers = zmin + (z_idx.float() + 0.5) * zstep
    
    # Step 5: Create meshgrid for centers
    # Source code snippet: torch.meshgrid(x_centers, y_centers, z_centers, indexing="ij")
    # This produces shape (X, Y, Z). 
    # But the layout description says (Z, Y, X, 3).
    # There is a contradiction in the source between the code snippet and the layout description.
    # We will prioritize the layout description (Z, Y, X, 3) as it is explicitly stated.
    # To get (Z, Y, X), we must meshgrid z, y, x.
    grid_centers_parts = torch.meshgrid(z_centers, y_centers, x_centers, indexing="ij")
    grid_centers = torch.stack(grid_centers_parts, dim=-1) # Shape: (Z, Y, X, 3)
    
    return grid_centers, grid_indices

def generate_unit_cube_grid(res):
    """
    Alternative method for unit cube [-1, 1] with resolution res.
    Generates points excluding boundary regions? Source S3 says "excluding boundary regions" but S1/S2 are inclusive.
    S3 says: "low = 0.5/res and high = 1 - 0.5/res". This implies a range [0.5/res, 1-0.5/res] scaled?
    Actually, S3 says: "Create 1D linspace from low to high with res points."
    low = 0.5/res, high = 1 - 0.5/res.
    This generates points in [0.5/res, 1-0.5/res].
    Then "Create 3D meshgrid with indexing='xy'".
    Then "Stack x, y, z into points array. Reshape to (-1, 3)".
    This method is less general than the bounds-based one.
    """
    low = 0.5 / res
    high = 1.0 - 0.5 / res
    x = torch.linspace(low, high, res)
    y = torch.linspace(low, high, res)
    z = torch.linspace(low, high, res)
    
    # indexing='xy' means the first two dimensions are swapped compared to 'ij'
    # meshgrid(x, y, z, indexing='xy') -> shape (Y, X, Z) ? 
    # In numpy/torch, 'xy' indexing means the output arrays have shape (len(y), len(x), len(z)) if passed x,y,z?
    # Actually, 'xy' indexing means the first two axes are treated as matrix coordinates.
    # meshgrid(x, y) with 'xy' returns (len(y), len(x)).
    # So meshgrid(x, y, z) with 'xy' returns (len(y), len(x), len(z)).
    # We want to stack x, y, z.
    parts = torch.meshgrid(x, y, z, indexing='xy')
    points = torch.stack(parts, dim=-1) # Shape (Y, X, Z, 3)
    points = points.reshape(-1, 3)
    return points
```

### Parameters and Typical Values
- `voxel_origin`: `[-1, -1, -1]` [S1, unofficial].
- `voxel_size`: `2.0 / (N - 1)` where N is the resolution [S1, unofficial].
- `transform`: `False` [S1, unofficial].

### Edge Cases and Pitfalls
- **Precision**: Floating point errors in `xstep` division may cause `xbins` to be off by one. Use `int()` truncation as shown [S2, unofficial].
- **Boundary Conditions**: The unit cube method (S3) excludes boundary regions by shifting the linspace range [S3, unofficial]. The bounds method (S2) includes the start point and steps up to the limit [S2, unofficial].
- **Indexing Order**: Confusion between `indexing='ij'` and `indexing='xy'` can lead to transposed grids. Ensure the meshgrid output matches the desired tensor layout (Z, Y, X) [S2, unofficial].

## Subtopics

### [Grid-based Density Sampling with Blue Noise](2608-grid-based-density-sampling-with-blue-noise.md)
A method to sample points from a 3D grid density field using stratified sampling guided by blue noise to reduce aliasing and ensure uniform distribution. It differs from standard grid generation by perturbing sample positions based on a blue noise pattern rather than using a rigid lattice. Choose this when rendering noisy density fields to minimize high-frequency artifacts. Link: [Grid-based Density Sampling with Blue Noise](2608-grid-based-density-sampling-with-blue-noise.md).

### [Coarse-to-Fine SDF Sampling for Mesh Creation](3495-coarse-to-fine-sdf-sampling-for-mesh-creation.md)
This method accelerates mesh generation from implicit surfaces by reducing neural network inference calls for Signed Distance Function (SDF) evaluation. It computes SDFs on a coarse grid, classifies regions as inside, outside, or near the surface, and only evaluates the network at full resolution for points near the zero-crossing boundary. This approach builds on [3D Coordinate Grid Generation](2333-3d-coordinate-grid-generation.md) to provide sample coordinates. Link: [Coarse-to-Fine SDF Sampling for Mesh Creation](3495-coarse-to-fine-sdf-sampling-for-mesh-creation.md).

### [Tetrahedral Grid Generation](3509-tetrahedral-grid-generation.md)
Link: [Tetrahedral Grid Generation](3509-tetrahedral-grid-generation.md).

### [Plane Grid Generation](5070-plane-grid-generation.md)
Plane Grid Generation constructs a regular 3D sampling grid by defining a set of parallel planes along each coordinate axis (X, Y, Z). It solves the problem of mapping scattered or arbitrarily shaped input data onto a structured rectangular grid suitable for volume visualization and SDF evaluation. It differs from its sibling [Tetrahedral Grid Generation](3509-tetrahedral-grid-generation.md) by enforcing axis-aligned planar topology rather than volumetric tessellation, and from [3D Coordinate Grid Generation](2333-3d-coordinate-grid-generation.md) by explicitly defining the geometric placement of the sampling planes themselves rather than just the sample points. Link: [Plane Grid Generation](5070-plane-grid-generation.md).

#### Mesh Creation from SDF
Creates a mesh by sampling SDF values on a voxel grid and saving to a PLY file [S1, unofficial]. It uses `voxel_origin` `[-1, -1, -1]` and `voxel_size` `2.0 / (N - 1)` [S1, unofficial]. Steps include setting the decoder to eval mode, generating samples via `get_samples`, and inferring SDF values via `infer_samples` [S1, unofficial].

#### Spherical Offset Grid Generation
Generates a sorted list of 3D integer offsets within a cubic volume, centered at origin, representing chunks to check for loading/unloading [S4, unofficial]. It calculates grid dimension `k = (half * 2) + 1`, iterates indices to 3D bounds, centers the position, and sorts offsets by squared distance to origin [S4, unofficial].

#### Voxel Grid Density Volume Creation
Generates a 3D density volume (and associated position/color data) within an AABB grid based on a specified density mode (sphere or cube) [S5, unofficial]. It maps grid indices to world coordinates and assigns density based on distance from center (sphere mode) or threshold proximity (cube mode) [S5, unofficial]. The output layout is a numpy array of shape `(H, W, D, 7)` storing `[x, y, z, r, g, b, density]` [S5, unofficial].

## Operating characteristics
- **Time Complexity**: $O(N^3)$ where $N$ is the resolution along one axis, as it generates $N^3$ points.
- **Memory Complexity**: $O(N^3)$ to store the coordinate and index tensors.
- **Performance**: Generating the grid is computationally inexpensive compared to the subsequent SDF or density evaluation. The bottleneck is usually the neural network inference or ray marching, not the grid generation itself.
- **Strengths**: Simple, deterministic, and easy to parallelize. Provides a regular sampling pattern suitable for grid-based algorithms.
- **Limitations**: Uniform sampling may be inefficient for sparse scenes or complex surfaces where most voxels are empty or far from the surface. Adaptive methods (like Coarse-to-Fine) are preferred for such cases.
- **When to use**: When a uniform sampling of the entire volume is required, or as a baseline for comparison.
- **When not to use**: When the scene is sparse or the surface is complex, leading to wasted computations on empty space.

## Background
The method originates from implementations in neural volume rendering and SDF-based mesh extraction, such as DeepSDF and related projects [S1, unofficial]. It is a fundamental step in voxelization pipelines where continuous space is discretized for neural field training or mesh extraction [S2, unofficial]. The specific implementation details (bounds vs. unit cube) reflect different use cases: general-purpose voxelization (S2) and standardized testing/evaluation (S1, S3).

## Sources
[S1] B1ueber2y/DIST-Renderer, https://github.com/B1ueber2y/DIST-Renderer, fetched 2026-10-26 (unofficial)
[S2] ziplab/VolSplat, https://github.com/ziplab/VolSplat, fetched 2026-10-27 (unofficial)
[S3] brandonyzhao/rtnf, https://github.com/brandonyzhao/rtnf, fetched 2026-10-27 (unofficial)
[S4] Binary Greedy Voxel Meshing Algorithm, https://github.com/TanTanDev/binary_greedy_mesher_demo, fetched 2026-10-26 (unofficial)
[S5] Rintarooo/volume_rendering_plot, https://github.com/Rintarooo/volume_rendering_plot, fetched 2026-10-26 (unofficial)
