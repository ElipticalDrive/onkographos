# 3D Coordinate Grid Generation

Also called: Voxel Grid Point Generation, Voxel Grid Sampling, Voxel Grid Sampling for Mesh Extraction

## Overview
3D Coordinate Grid Generation constructs a regular lattice of 3D sample points within a specified spatial volume, typically defined by bounds and step sizes or a resolution within a canonical unit cube. It serves as the foundational sampling mechanism for evaluating implicit functions (such as Signed Distance Fields) or density fields across a volumetric domain. It differs from [Coordinate Grid Generation](1273-coordinate-grid-generation.md) by explicitly handling 3D spatial bounds and voxel center calculations rather than just generating flattened 2D image-like coordinate arrays, and from [CT Slice Voxel Construction](8351-ct-slice-voxel-construction.md) by generating a full 3D volume rather than processing 2D slices independently.

## How it works
The method generates a set of 3D points representing voxel centers and their corresponding integer indices. Two primary implementations exist in the source literature: one based on a canonical unit cube resolution (common in neural field training) [S1, S3, unofficial] and one based on arbitrary spatial bounds and step sizes (common in voxel-based reconstruction) [S2, unofficial].

### Inputs and Outputs
**Inputs:**
*   `resolution` ($N$): Integer number of voxels per axis (for unit cube method) [S1, S3, unofficial].
*   `voxel_origin`: List of 3 floats $[x_{min}, y_{min}, z_{min}]$, default $[-1, -1, -1]$ [S1, unofficial].
*   `voxel_size`: Float size of each voxel, calculated as $2.0 / (N - 1)$ [S1, unofficial].
*   `xmin, xmax, xstep`: Spatial bounds and step size for X axis [S2, unofficial].
*   `ymin, ymax, ystep`: Spatial bounds and step size for Y axis [S2, unofficial].
*   `zmin, zmax, zstep`: Spatial bounds and step size for Z axis [S2, unofficial].
*   `transform`: Boolean flag to apply a transformation matrix to the generated points, default `False` [S1, unofficial].

**Outputs:**
*   `points`: Tensor of shape $(Z, Y, X, 3)$ or flattened $(-1, 3)$ containing float coordinates of voxel centers [S2, S3, unofficial].
*   `indices`: Tensor of shape $(Z, Y, X, 3)$ containing integer grid indices [S2, unofficial].

### Data Structures
*   **Points Tensor**: Float tensor storing $[x, y, z]$ coordinates.
*   **Indices Tensor**: Integer tensor storing $[i, j, k]$ grid indices.
*   **Overall Index**: A flattened integer tensor from $0$ to $N^3 - 1$ used to derive 3D coordinates [S1, unofficial].

### Steps

#### Method A: Canonical Unit Cube (DeepSDF style) [S1, S3, unofficial]
1.  **Calculate Bounds**: Determine the low and high bounds for the linspace.
    $$ low = \frac{0.5}{res} $$
    $$ high = 1 - \frac{0.5}{res} $$
    Where $res$ is the resolution $N$ [S3, unofficial].
2.  **Generate 1D Coordinates**: Create a 1D linear space array `x_1d` from `low` to `high` with `res` points. Repeat for Y and Z axes [S3, unofficial].
3.  **Create 3D Meshgrid**: Generate a 3D meshgrid from the 1D arrays using `indexing='xy'` [S3, unofficial].
    *   Let $X, Y, Z$ be the resulting 3D coordinate tensors.
4.  **Stack and Reshape**: Stack $X, Y, Z$ into a single array and reshape to $(-1, 3)$ to get the flattened points [S3, unofficial].
5.  **Initialize Samples**: Create a tensor `samples` of shape $(N^3, 4)$ initialized to zeros [S1, unofficial].
6.  **Compute Indices**:
    *   Create `overall_index` tensor from $0$ to $N^3 - 1$ [S1, unofficial].
    *   Derive $x, y, z$ indices from `overall_index` [S1, unofficial].
7.  **Convert to Coordinates**: Convert indices to world coordinates using `voxel_origin` and `voxel_size` [S1, unofficial].
    $$ x_{coord} = x_{idx} \cdot voxel\_size + x_{min} $$
    $$ y_{coord} = y_{idx} \cdot voxel\_size + y_{min} $$
    $$ z_{coord} = z_{idx} \cdot voxel\_size + z_{min} $$
8.  **Transform (Optional)**: If `transform` is true, apply `transform_points_tensor` to the coordinates [S1, unofficial].

#### Method B: Arbitrary Bounds (VolSplat style) [S2, unofficial]
1.  **Unpack Bounds**: Extract $x_{min}, x_{max}, x_{step}, y_{min}, y_{max}, y_{step}, z_{min}, z_{max}, z_{step}$ [S2, unofficial].
2.  **Calculate Bins**: Compute the number of voxels along each axis.
    $$ x_{bins} = \lfloor \frac{x_{max} - x_{min}}{x_{step}} \rfloor $$
    $$ y_{bins} = \lfloor \frac{y_{max} - y_{min}}{y_{step}} \rfloor $$
    $$ z_{bins} = \lfloor \frac{z_{max} - z_{min}}{z_{step}} \rfloor $$
3.  **Generate Integer Indices**: Create integer ranges for each axis.
    $$ x_{idx} = \text{arange}(x_{bins}) $$
    $$ y_{idx} = \text{arange}(y_{bins}) $$
    $$ z_{idx} = \text{arange}(z_{bins}) $$
4.  **Compute Float Centers**: Calculate the center of each voxel.
    $$ x_{center} = x_{min} + (x_{idx} + 0.5) \cdot x_{step} $$
    $$ y_{center} = y_{min} + (y_{idx} + 0.5) \cdot y_{step} $$
    $$ z_{center} = z_{min} + (z_{idx} + 0.5) \cdot z_{step} $$
5.  **Create Meshgrid**: Stack indices and centers into grids using `indexing="ij"`.
    $$ grid_{idx} = \text{stack}(\text{meshgrid}(z_{idx}, y_{idx}, x_{idx}, \text{indexing}="ij")) $$
    $$ grid_{centers} = \text{stack}(\text{meshgrid}(x_{center}, y_{center}, z_{center}, \text{indexing}="ij")) $$
6.  **Output**: Return `grid_centers` (shape $Z, Y, X, 3$) and `grid_idx` (shape $Z, Y, X, 3$) [S2, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_3d_coordinate_grid(bounds_mode="arbitrary", resolution=None, origin=None, 
                                voxel_size=None, bounds=None, steps=None):
    """
    Generates a 3D grid of sample points.
    
    Args:
        bounds_mode: "unit_cube" or "arbitrary"
        resolution: int, N (for unit cube)
        origin: list[float, float, float], default [-1, -1, -1]
        voxel_size: float, calculated as 2.0 / (N - 1) if not provided
        bounds: dict with xmin, xmax, ymin, ymax, zmin, zmax
        steps: dict with xstep, ystep, zstep
    """
    if bounds_mode == "unit_cube":
        N = resolution
        if origin is None:
            origin = [-1.0, -1.0, -1.0]
        if voxel_size is None:
            voxel_size = 2.0 / (N - 1)
            
        # Method A: Canonical Unit Cube
        low = 0.5 / N
        high = 1.0 - 0.5 / N
        
        # Generate 1D lines
        x_1d = linspace(low, high, N)
        y_1d = linspace(low, high, N)
        z_1d = linspace(low, high, N)
        
        # Create 3D meshgrid with 'xy' indexing
        X, Y, Z = meshgrid(x_1d, y_1d, z_1d, indexing='xy')
        
        # Stack and flatten
        points = stack([X, Y, Z], axis=-1).reshape(-1, 3)
        
        # Generate indices
        overall_index = arange(N**3)
        z_idx = overall_index // (N * N)
        y_idx = (overall_index // N) % N
        x_idx = overall_index % N
        
        # Convert indices to coordinates
        coords_x = x_idx * voxel_size + origin[0]
        coords_y = y_idx * voxel_size + origin[1]
        coords_z = z_idx * voxel_size + origin[2]
        
        indices = stack([x_idx, y_idx, z_idx], axis=-1).reshape(-1, 3)
        points = stack([coords_x, coords_y, coords_z], axis=-1).reshape(-1, 3)
        
        return points, indices

    elif bounds_mode == "arbitrary":
        xmin, xmax, xstep = bounds['xmin'], bounds['xmax'], steps['xstep']
        ymin, ymax, ystep = bounds['ymin'], bounds['ymax'], steps['ystep']
        zmin, zmax, zstep = bounds['zmin'], bounds['zmax'], steps['zstep']
        
        # Method B: Arbitrary Bounds
        x_bins = int((xmax - xmin) / xstep)
        y_bins = int((ymax - ymin) / ystep)
        z_bins = int((zmax - zmin) / zstep)
        
        x_idx = arange(x_bins)
        y_idx = arange(y_bins)
        z_idx = arange(z_bins)
        
        # Compute centers
        x_centers = xmin + (x_idx + 0.5) * xstep
        y_centers = ymin + (y_idx + 0.5) * ystep
        z_centers = zmin + (z_idx + 0.5) * zstep
        
        # Stack into grids with 'ij' indexing
        grid_idx = stack(meshgrid(z_idx, y_idx, x_idx, indexing="ij"), dim=-1)
        grid_centers = stack(meshgrid(x_centers, y_centers, z_centers, indexing="ij"), dim=-1)
        
        return grid_centers, grid_idx
```

### Parameters and Typical Values
*   **Resolution ($N$)**: Typically 64, 128, or 256 for neural fields [S1, S3, unofficial].
*   **Voxel Origin**: Default $[-1, -1, -1]$ [S1, unofficial].
*   **Voxel Size**: $2.0 / (N - 1)$ [S1, unofficial].
*   **Transform**: Default `False` [S1, unofficial].

### Edge Cases and Pitfalls
*   **Boundary Precision**: In the unit cube method, using `linspace` from $0.5/N$ to $1 - 0.5/N$ ensures points are centered within the unit cube $[-1, 1]^3$ when scaled, avoiding boundary artifacts [S3, unofficial].
*   **Indexing Convention**: The arbitrary bounds method uses `indexing="ij"` (row-major, Z-Y-X order in meshgrid output), while the unit cube method uses `indexing='xy'` (column-major, X-Y order) [S2, S3, unofficial]. Mixing these conventions leads to transposed grids.
*   **Integer Division**: When calculating bins in arbitrary bounds, integer truncation (`int()`) is used, which may result in the last voxel being slightly smaller or excluded if the bounds are not perfectly divisible by the step size [S2, unofficial].

## Subtopics

### [Grid-based Density Sampling with Blue Noise](2608-grid-based-density-sampling-with-blue-noise.md)
A method to sample points from a 3D grid density field using stratified sampling guided by blue noise to reduce aliasing and ensure uniform distribution. It sets itself apart from standard grid generation by introducing stochasticity to break up regular grid artifacts, while maintaining the structural benefits of a grid. Choose this when rendering density fields where regular grid sampling causes moiré patterns or aliasing. Link: [Grid-based Density Sampling with Blue Noise](2608-grid-based-density-sampling-with-blue-noise.md).

### [Coarse-to-Fine SDF Sampling for Mesh Creation](3495-coarse-to-fine-sdf-sampling-for-mesh-creation.md)
This method accelerates mesh generation from implicit surfaces by reducing neural network inference calls for Signed Distance Function (SDF) evaluation. It computes SDFs on a coarse grid, classifies regions as inside, outside, or near the surface, and only evaluates the network at full resolution for points near the zero-crossing boundary. This approach builds on [3D Coordinate Grid Generation](2333-3d-coordinate-grid-generation.md) to provide sample coordinates. Link: [Coarse-to-Fine SDF Sampling for Mesh Creation](3495-coarse-to-fine-sdf-sampling-for-mesh-creation.md).

### [Tetrahedral Grid Generation](3509-tetrahedral-grid-generation.md)
Tetrahedral Grid Generation converts a standard 3D voxel grid (a lattice of cubes) into a conforming mesh of tetrahedra. This method solves the problem of creating a non-orthogonal sampling domain for Signed Distance Function (SDF) evaluation or volume rendering, which can reduce aliasing artifacts compared to axis-aligned voxel grids. It sets itself apart from sibling methods like [Grid-based Density Sampling with Blue Noise](2608-grid-based-density-sampling-with-blue-noise.md) by providing a deterministic, structured connectivity map rather than stochastic point distributions, and from [Plane Grid Generation](5070-plane-grid-generation.md) by operating in full 3D volume rather than 2D slices. Link: [Tetrahedral Grid Generation](3509-tetrahedral-grid-generation.md).

### [Plane Grid Generation](5070-plane-grid-generation.md)
Plane Grid Generation constructs a regular 3D sampling grid by defining a set of parallel planes along each coordinate axis (X, Y, Z). It solves the problem of mapping scattered or arbitrarily shaped input data onto a structured rectangular grid suitable for volume visualization and SDF evaluation. It differs from its sibling [Tetrahedral Grid Generation](3509-tetrahedral-grid-generation.md) by enforcing axis-aligned planar topology rather than volumetric tessellation, and from [3D Coordinate Grid Generation](2333-3d-coordinate-grid-generation.md) by explicitly defining the geometric placement of the sampling planes themselves rather than just the sample points. Link: [Plane Grid Generation](5070-plane-grid-generation.md).

### Mesh Creation from SDF
Creates a mesh by sampling SDF values on a voxel grid and saving to a PLY file. It uses the standard unit cube grid generation with `voxel_origin` set to $[-1, -1, -1]$ and `voxel_size` calculated as $2.0 / (N - 1)$. It includes a step to set the decoder to evaluation mode and apply optional point transformations. Link: [Coarse-to-Fine SDF Sampling for Mesh Creation](3495-coarse-to-fine-sdf-sampling-for-mesh-creation.md) (as a sub-component).

### Spherical Offset Grid Generation
Generates a sorted list of 3D integer offsets within a cubic volume, centered at origin, representing chunks to check for loading/unloading. It calculates grid dimension $k = (half \times 2) + 1$, iterates through indices, converts to 3D bounds, centers the position, and sorts by squared distance to origin. This is used for spatial chunk management rather than direct SDF evaluation. Link: [Spherical Offset Grid Generation](3495-coarse-to-fine-sdf-sampling-for-mesh-creation.md) (as a sub-component).

### Voxel Grid Density Volume Creation
Generates a 3D density volume (and associated position/color data) within an AABB grid based on a specified density mode (sphere or cube). It maps grid indices to world coordinates and assigns density values based on distance from center (sphere) or threshold proximity (cube). The output layout is a numpy array of shape $(H, W, D, 7)$ storing $[x, y, z, r, g, b, density]$. Link: [Voxel Grid Density Volume Creation](3495-coarse-to-fine-sdf-sampling-for-mesh-creation.md) (as a sub-component).

## Operating characteristics
*   **Time Complexity**: $O(N^3)$ for generating the grid points, where $N$ is the resolution per axis. The meshgrid and stacking operations are linear in the number of points.
*   **Memory Complexity**: $O(N^3)$ to store the coordinates and indices. For high resolutions (e.g., $256^3$), this can consume significant memory (several GBs for float32 tensors).
*   **Performance**: The unit cube method is highly optimized in PyTorch/TensorFlow via `meshgrid` and `linspace`. The arbitrary bounds method requires additional arithmetic for bin calculation.
*   **Strengths**: Provides a deterministic, regular sampling pattern essential for marching cubes and structured volume rendering.
*   **Limitations**: Regular grid sampling is prone to aliasing artifacts (staircasing, moiré) in high-frequency scenes. Does not adapt to surface complexity.
*   **When to use**: When evaluating implicit functions on a regular domain, or as a base for hierarchical sampling methods.
*   **When not to use**: When adaptive sampling is required (use [Coarse-to-Fine SDF Sampling](3495-coarse-to-fine-sdf-sampling-for-mesh-creation.md)) or when aliasing is a critical issue (use [Grid-based Density Sampling with Blue Noise](2608-grid-based-density-sampling-with-blue-noise.md)).

## Background
3D Coordinate Grid Generation is a fundamental technique in volumetric computer graphics, originating from the need to discretize continuous 3D space for numerical evaluation of scalar fields. It is widely used in Signed Distance Field (SDF) reconstruction, neural radiance fields (NeRF), and voxel-based volume rendering. The specific implementation details in [S1, unofficial] are derived from DeepSDF and related neural implicit surface representations, while [S2, unofficial] reflects voxel-based splatting techniques.

## Sources
[S1] B1ueber2y/DIST-Renderer, https://github.com/B1ueber2y/DIST-Renderer, fetched 2026-10-26 (unofficial)
[S2] ziplab/VolSplat, https://github.com/ziplab/VolSplat, fetched 2026-10-27 (unofficial)
[S3] brandonyzhao/rtnf, https://github.com/brandonyzhao/rtnf, fetched 2026-10-27 (unofficial)
[S4] Binary Greedy Voxel Meshing Algorithm, https://github.com/TanTanDev/binary_greedy_mesher_demo, fetched 2026-10-26 (unofficial)
[S5] Rintarooo/volume_rendering_plot, https://github.com/Rintarooo/volume_rendering_plot, fetched 2026-10-26 (unofficial)
