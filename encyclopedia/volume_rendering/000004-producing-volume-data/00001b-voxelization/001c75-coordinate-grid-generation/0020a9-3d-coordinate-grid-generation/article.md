# 3D Coordinate Grid Generation

Also called: Voxel Grid Point Generation, Voxel Grid Sampling, Voxel Grid Sampling for Mesh Extraction

## Overview
3D Coordinate Grid Generation produces a regular 3D lattice of sample points within a specified voxel space, primarily to provide input coordinates for Signed Distance Function (SDF) evaluation or density field sampling in neural rendering pipelines [S1, S2, unofficial]. It differs from its parent [Coordinate Grid Generation](1273-coordinate-grid-generation.md), which targets 2D image-plane layouts, by explicitly handling the third dimension and supporting spatial bounds and step sizes for arbitrary voxel resolutions [S2, unofficial]. It serves as the foundational sampling mechanism for methods like [Coarse-to-Fine SDF Sampling for Mesh Creation](3495-coarse-to-fine-sdf-sampling-for-mesh-creation.md) and [Grid-based Density Sampling with Blue Noise](2608-grid-based-density-sampling-with-blue-noise.md) [S1, unofficial].

## How it works
The method generates a flattened grid of 3D coordinates and corresponding integer indices. Two primary implementation patterns exist: one based on fixed resolution within a unit cube (common in DeepSDF-style pipelines) and one based on explicit spatial bounds and step sizes (common in sparse voxel or grid-based splatting pipelines) [S1, S2, unofficial].

### Inputs and Outputs
**Inputs:**
- `resolution` ($N$): Integer, number of voxels per axis (for fixed resolution method) [S1, unofficial].
- `voxel_origin`: List of 3 floats $[o_x, o_y, o_z]$, default $[-1, -1, -1]$ [S1, unofficial].
- `voxel_size`: Float, size of each voxel edge. Calculated as $2.0 / (N - 1)$ for unit cube bounds [S1, unofficial].
- `xmin, xmax, xstep`: Floats defining X-axis bounds and step size [S2, unofficial].
- `ymin, ymax, ystep`: Floats defining Y-axis bounds and step size [S2, unofficial].
- `zmin, zmax, zstep`: Floats defining Z-axis bounds and step size [S2, unofficial].
- `transform`: Boolean, whether to apply a coordinate transformation matrix (default `False`) [S1, unofficial].

**Outputs:**
- `points`: Tensor of shape $(N^3, 3)$ or $(Z, Y, X, 3)$ containing voxel center coordinates [S1, S2, unofficial].
- `indices`: Tensor of shape $(Z, Y, X, 3)$ containing integer grid indices $(i, j, k)$ [S2, unofficial].

### Data Structures
1. **Overall Index Tensor**: A 1D tensor of integers from $0$ to $N^3 - 1$ (or $X_{bins} \times Y_{bins} \times Z_{bins} - 1$) used to iterate over all voxels [S1, S2, unofficial].
2. **Coordinate Grid**: A 3D tensor (or flattened 2D tensor) where the last dimension is 3, storing $(x, y, z)$ coordinates [S1, S2, unofficial].
3. **Index Grid**: A 3D tensor where the last dimension is 3, storing integer voxel indices $(i, j, k)$ [S2, unofficial].

### Steps (Fixed Resolution / Unit Cube Method)
This method, used in [S1, unofficial], assumes a unit cube $[-1, 1]^3$ centered at the origin.

1. **Calculate Voxel Size**:
   $$ v_{size} = \frac{2.0}{N - 1} $$
   where $N$ is the resolution [S1, unofficial].

2. **Define Bounds**:
   $$ low = \frac{0.5}{N} $$
   $$ high = 1 - \frac{0.5}{N} $$
   These bounds ensure points are centered within the unit cube cells, excluding the exact boundary edges if desired [S3, unofficial]. Note: [S1, unofficial] uses `voxel_origin` and `voxel_size` directly, while [S3, unofficial] calculates `low` and `high` based on resolution. We follow [S1, unofficial]'s explicit parameterization for generality, but [S3, unofficial] implies centering. The standard centering formula for a voxel $i$ in range $[min, max]$ with $N$ steps is $min + (i + 0.5) \times step$.

3. **Generate 1D Coordinates**:
   Create a 1D array `x_1d` of $N$ points linearly spaced from $low$ to $high$ (or using `voxel_origin` and `voxel_size`).
   $$ x_i = voxel\_origin_x + (i + 0.5) \times voxel\_size - 1.0 \quad \text{(if origin is -1)} $$
   More generally, using [S1, unofficial]'s parameters:
   $$ x_i = voxel\_origin_x + i \times voxel\_size $$
   However, [S3, unofficial] and [S2, unofficial] emphasize centering. The robust implementation uses centering:
   $$ x_i = xmin + (i + 0.5) \times xstep $$

4. **Create 3D Meshgrid**:
   Generate a 3D meshgrid from `x_1d`, `y_1d`, `z_1d` with `indexing='xy'` (or `'ij'` depending on axis ordering convention).
   Let $X, Y, Z$ be the resulting 3D tensors of shape $(N, N, N)$ [S1, S2, unofficial].

5. **Stack and Reshape**:
   Stack $X, Y, Z$ into a tensor `points` of shape $(N, N, N, 3)$.
   Reshape `points` to $(-1, 3)$ to get the flattened list of sample points [S1, S2, unofficial].

6. **Generate Integer Indices**:
   Create integer ranges for each axis:
   $$ i_{idx} = \text{arange}(N) $$
   $$ j_{idx} = \text{arange}(N) $$
   $$ k_{idx} = \text{arange}(N) $$
   Stack them into `grid_idx` of shape $(N, N, N, 3)$ [S2, unofficial].

7. **Transform (Optional)**:
   If `transform` is `True`, apply a transformation matrix to `points` [S1, unofficial].

### Steps (Explicit Bounds / Step Method)
This method, used in [S2, unofficial], allows arbitrary spatial bounds.

1. **Unpack Parameters**:
   Extract `xmin, xmax, xstep, ymin, ymax, ystep, zmin, zmax, zstep` [S2, unofficial].

2. **Calculate Bin Counts**:
   $$ x_{bins} = \lfloor \frac{x_{max} - x_{min}}{x_{step}} \rfloor $$
   $$ y_{bins} = \lfloor \frac{y_{max} - y_{min}}{y_{step}} \rfloor $$
   $$ z_{bins} = \lfloor \frac{z_{max} - z_{min}}{z_{step}} \rfloor $$
   [S2, unofficial] uses `int()` conversion, implying truncation.

3. **Generate Integer Indices**:
   $$ x_{idx} = \text{arange}(x_{bins}) $$
   $$ y_{idx} = \text{arange}(y_{bins}) $$
   $$ z_{idx} = \text{arange}(z_{bins}) $$
   Stack into `grid_idx` using `meshgrid(z_idx, y_idx, x_idx, indexing="ij")` to match the $(Z, Y, X)$ layout [S2, unofficial].

4. **Calculate Float Centers**:
   $$ x_{centers} = x_{min} + (x_{idx} + 0.5) \times x_{step} $$
   $$ y_{centers} = y_{min} + (y_{idx} + 0.5) \times y_{step} $$
   $$ z_{centers} = z_{min} + (z_{idx} + 0.5) \times z_{step} $$
   [S2, unofficial]

5. **Generate Coordinate Grid**:
   Stack `x_centers, y_centers, z_centers` using `meshgrid(..., indexing="ij")` into `grid_centers` of shape $(Z, Y, X, 3)$ [S2, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np

def generate_3d_coordinate_grid_fixed(resolution, voxel_origin=None, voxel_size=None, transform=False):
    """
    Generates a 3D grid of sample points for SDF evaluation within a unit cube [-1, 1]^3.
    Based on [S1, S3, unofficial].
    """
    if voxel_origin is None:
        voxel_origin = [-1.0, -1.0, -1.0]
    if voxel_size is None:
        voxel_size = 2.0 / (resolution - 1)
    
    # Calculate bounds for centering
    # Using the formula from [S2, unofficial] adapted for fixed bounds:
    # center = min + (index + 0.5) * step
    # For unit cube [-1, 1], min=-1, step=2/(N-1)
    
    # Generate 1D coordinates
    # Using torch.linspace for precision
    x_1d = torch.linspace(-1.0 + voxel_size/2, 1.0 - voxel_size/2, resolution)
    y_1d = torch.linspace(-1.0 + voxel_size/2, 1.0 - voxel_size/2, resolution)
    z_1d = torch.linspace(-1.0 + voxel_size/2, 1.0 - voxel_size/2, resolution)
    
    # Create 3D meshgrid with 'xy' indexing (x varies fastest in memory if flattened row-major? 
    # Actually meshgrid with 'xy' returns X, Y, Z where X varies along columns, Y along rows.
    # For 3D, we need to be careful with stacking.
    # [S1, unofficial] uses indexing='xy'. [S2, unofficial] uses indexing='ij'.
    # Let's follow [S1, unofficial] for the fixed resolution case as it's the primary source for SDF sampling.
    X, Y, Z = torch.meshgrid(x_1d, y_1d, z_1d, indexing='xy')
    
    # Stack into points array (N, N, N, 3)
    points = torch.stack((X, Y, Z), dim=-1)
    
    # Reshape to (-1, 3)
    points_flat = points.reshape(-1, 3)
    
    # Generate integer indices
    indices_flat = torch.arange(resolution**3)
    z_idx = indices_flat // (resolution * resolution)
    y_idx = (indices_flat // resolution) % resolution
    x_idx = indices_flat % resolution
    grid_idx = torch.stack((x_idx, y_idx, z_idx), dim=-1).float() # Shape (N^3, 3)
    
    # Apply transform if requested
    if transform:
        # Placeholder for transform_points_tensor logic [S1, unofficial]
        # Typically a 3x3 or 4x4 matrix multiplication
        pass 
        
    return points_flat, grid_idx

def generate_3d_coordinate_grid_bounds(xmin, xmax, xstep, ymin, ymax, ystep, zmin, zmax, zstep):
    """
    Generates a 3D grid of voxel center coordinates and integer indices within specified bounds.
    Based on [S2, unofficial].
    """
    # Calculate number of bins
    xbins = int((xmax - xmin) / xstep)
    ybins = int((ymax - ymin) / ystep)
    zbins = int((zmax - zmin) / zstep)
    
    # Create integer indices
    x_idx = torch.arange(xbins)
    y_idx = torch.arange(ybins)
    z_idx = torch.arange(zbins)
    
    # Stack indices into grid: (Z, Y, X, 3)
    # Using 'ij' indexing as per [S2, unofficial]
    grid_idx = torch.stack(torch.meshgrid(z_idx, y_idx, x_idx, indexing="ij"), dim=-1)
    
    # Compute float centers
    x_centers = xmin + (x_idx + 0.5) * xstep
    y_centers = ymin + (y_idx + 0.5) * ystep
    z_centers = zmin + (z_idx + 0.5) * zstep
    
    # Stack centers into grid: (Z, Y, X, 3)
    grid_centers = torch.stack(torch.meshgrid(x_centers, y_centers, z_centers, indexing="ij"), dim=-1)
    
    return grid_centers, grid_idx
```

### Parameters with Typical Values
- `resolution` ($N$): 64, 128, 256 [S1, unofficial].
- `voxel_origin`: $[-1, -1, -1]$ [S1, unofficial].
- `voxel_size`: $2.0 / (N - 1)$ [S1, unofficial].
- `transform`: `False` [S1, unofficial].
- `xstep`, `ystep`, `zstep`: Determined by scene scale and desired voxel count [S2, unofficial].

### Edge Cases and Pitfalls
- **Boundary Precision**: Using `linspace` from `low` to `high` (where `low = 0.5/res`, `high = 1 - 0.5/res`) excludes the exact boundaries of the unit cube, which is often desired to avoid double-counting or edge artifacts in SDF evaluation [S3, unofficial].
- **Indexing Convention**: `meshgrid` supports `'xy'` and `'ij'` indexing. `'xy'` treats the first two dimensions as columns and rows (matrix-like), while `'ij'` treats them as matrix indices (array-like). Mismatching indexing between coordinate generation and index generation leads to misaligned samples [S1, S2, unofficial].
- **Integer Division**: When calculating bin counts from bounds, truncation (`int()`) may leave unused space if the bounds are not exact multiples of the step size [S2, unofficial].

## Subtopics

### [Grid-based Density Sampling with Blue Noise](2608-grid-based-density-sampling-with-blue-noise.md)
A method to sample points from a 3D grid density field using stratified sampling guided by blue noise to reduce aliasing and ensure uniform distribution. It differs from standard grid generation by perturbing the regular grid points or selecting a subset based on blue noise patterns, rather than using all grid points. It is chosen when reducing computational cost for density estimation while maintaining visual quality. Link: [Grid-based Density Sampling with Blue Noise](2608-grid-based-density-sampling-with-blue-noise.md).

### [Coarse-to-Fine SDF Sampling for Mesh Creation](3495-coarse-to-fine-sdf-sampling-for-mesh-creation.md)
This method accelerates mesh generation from implicit surfaces by reducing neural network inference calls for Signed Distance Function (SDF) evaluation. It computes SDFs on a coarse grid, classifies regions as inside, outside, or near the surface, and only evaluates the network at full resolution for points near the zero-crossing boundary. This approach builds on [3D Coordinate Grid Generation](2333-3d-coordinate-grid-generation.md) to provide sample coordinates. Link: [Coarse-to-Fine SDF Sampling for Mesh Creation](3495-coarse-to-fine-sdf-sampling-for-mesh-creation.md).

### [Plane Grid Generation](5070-plane-grid-generation.md)
Plane Grid Generation precomputes vertex index arrays for triangle lists representing surfaces parallel to the xy, yz, and xz planes on a regular 3D grid. It solves the problem of efficiently indexing grid vertices for rendering or processing planar slices without dynamic calculation during the rendering pass. It differs from its siblings by generating static, axis-aligned planar meshes rather than density-based sampling or coarse-to-fine hierarchical structures. Link: [Plane Grid Generation](5070-plane-grid-generation.md).

#### Tetrahedral Grid Generation
Generates a tetrahedral decomposition of a 3D voxel grid for mesh extraction. It initializes a tetrahedron definition matrix `TF` of size $((n_x-1)(n_y-1)(n_z-1) \times 6, 4)$ [S4, unofficial]. It iterates over grid indices $i \in [0, n_x-2]$, $j \in [0, n_y-2]$, $k \in [0, n_z-2]$. For each cube, it calculates vertex indices $v_1 \dots v_8$ using linear indexing: $v_1 = i + n_x(j + n_y k)$, etc. It defines 6 tetrahedra per cube by assigning vertex indices to `TF` rows: $(v_1, v_3, v_8, v_7)$, $(v_1, v_8, v_5, v_7)$, $(v_1, v_3, v_4, v_8)$, $(v_1, v_4, v_2, v_8)$, $(v_1, v_6, v_5, v_8)$, $(v_1, v_2, v_6, v_8)$ [S4, unofficial]. Pitfall: Parallelization of the grid generation loop may cause race conditions if not handled correctly [S4, unofficial].

#### Mesh Creation from SDF
Creates a mesh by sampling SDF values on a voxel grid and saving to a PLY file [S1, unofficial]. Steps: Record start time, set decoder to eval mode, set `voxel_origin` to $[-1, -1, -1]$ and `voxel_size` to $2.0 / (N - 1)$, get samples using `get_samples`, infer SDF values using `infer_samples` [S1, unofficial]. Parameter: `transform` (Boolean, default `False`) [S1, unofficial].

#### Spherical Offset Grid Generation
Generates a sorted list of 3D integer offsets within a cubic volume, centered at origin, representing chunks to check for loading/unloading [S5, unofficial]. Steps: Calculate grid dimension $k = (\text{half} \times 2) + 1$. Iterate $i$ from $0$ to $k^3 - 1$. Convert index to 3D bounds: `pos = index_to_ivec3_bounds(i, k)`. Center the position: `pos -= IVec3::splat((k * 0.5) as i32)`. Add to sampling offsets vector. Sort offsets by squared distance to origin [S5, unofficial].

#### Voxel Grid Density Volume Creation
Generates a 3D density volume (and associated position/color data) within an AABB grid based on a specified density mode (sphere or cube) [S6, unofficial]. Steps: Iterate $i \in [0, H-1]$, $k \in [0, W-1]$, $j \in [0, D-1]$. Map grid indices to world coordinates: $x = \frac{i}{H-1}(max_x - min_x) + min_x$, etc. If `density_mode` is 'sphere': calculate $r^2 = (x-x_c)^2 + (y-y_c)^2 + (z-z_c)^2$. If $r^2 < 1.0$, density = 1.0. If $r^2 < 0.25$, density = `high_density_val`. Else density = 0. If `density_mode` is 'cube': density = `min_density_val`. If $x, y, z$ are all within $[center - thres\_pos, center + thres\_pos]$, density = `high_density_val`. Else density = `min_density_val` [S6, unofficial]. Parameters: `thres_pos` (typical 0.8), `high_density_val` (typical 0.95), `min_density_val` (typical 0.2) [S6, unofficial]. Layout: `density_volume` is a numpy array of shape $(H, W, D, 7)$. Indices $[i, k, j]$ correspond to grid coordinates. Last dimension stores: $[0]=x, [1]=y, [2]=z$ (world position), $[3]=r, [4]=g, [5]=b$ (color, always 255), $[6]=$ density (float) [S6, unofficial].

## Operating characteristics
- **Time Complexity**: $O(N^3)$ for generating the grid, where $N$ is the resolution per axis. The generation itself is linear in the number of voxels.
- **Memory Complexity**: $O(N^3)$ to store the coordinate and index tensors. For high resolutions (e.g., $256^3$), this can consume significant memory (e.g., $256^3 \times 3 \times 4$ bytes $\approx 24$ MB for floats, plus indices).
- **Measured Performance**: Not given in the sources.
- **Scaling**: Scales linearly with the number of voxels. For very large grids, sparse representations or hierarchical methods (like [Coarse-to-Fine SDF Sampling](3495-coarse-to-fine-sdf-sampling-for-mesh-creation.md)) are preferred to avoid memory exhaustion.
- **Strengths**: Simple, deterministic, and easy to implement. Provides uniform coverage of the volume.
- **Limitations**: Inefficient for sparse volumes where most voxels are empty. Fixed resolution methods require recomputation if the resolution changes.
- **When to use**: When uniform sampling is required for SDF evaluation, density estimation, or when the volume is dense and fits in memory.
- **When not to use**: For very large sparse volumes, or when adaptive sampling is needed.

## Background
The method originates from neural implicit surface reconstruction pipelines, particularly DeepSDF and its derivatives, where regular grids are used to sample SDF values for training and inference [S1, unofficial]. It is also used in voxel-based rendering and splatting methods like VolSplat for generating voxel centers [S2, unofficial]. The concept of generating coordinate grids for neural fields is a standard preprocessing step in 3D deep learning.

## Sources
[S1] B1ueber2y/DIST-Renderer, https://github.com/B1ueber2y/DIST-Renderer, fetched 2026-10-26 (unofficial)
[S2] ziplab/VolSplat, https://github.com/ziplab/VolSplat, fetched 2026-10-27 (unofficial)
[S3] brandonyzhao/rtnf, https://github.com/brandonyzhao/rtnf, fetched 2026-10-27 (unofficial)
[S4] mwrazam/marchingtets-reconstruction, https://github.com/mwrazam/marchingtets-reconstruction, fetched 2026-10-27 (unofficial)
[S5] Binary Greedy Voxel Meshing Algorithm, https://github.com/TanTanDev/binary_greedy_mesher_demo, fetched 2026-10-26 (unofficial)
[S6] Rintarooo/volume_rendering_plot, https://github.com/Rintarooo/volume_rendering_plot, fetched 2026-10-26 (unofficial)
