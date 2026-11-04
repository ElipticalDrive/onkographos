# VoxelGrid Down Sample

## Overview
VoxelGrid Down Sample reduces the resolution of a voxel grid by a specified integer factor using max pooling [S1, unofficial]. It solves the problem of generating lower-resolution representations for multiresolution volume rendering pipelines [S1, unofficial]. It is distinguished from its siblings by its simplicity and deterministic nature, relying on block-wise maximum operations rather than learned parameters or complex geometric decompositions [S1, unofficial][S2, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A `VoxelGrid` object containing a 3D data array `data` of shape $(N_x, N_y, N_z)$, a location `loc`, and a scale `scale`.
- Output: A new `VoxelGrid` object with resolution reduced by `factor` in each dimension, retaining the original `loc` and `scale`.

**Data Structures:**
- `VoxelGrid`:
  - `data`: numpy array of shape $(N_x, N_y, N_z)$.
  - `loc`: 3D vector representing the grid's origin.
  - `scale`: scalar or 3D vector representing voxel size.

**Parameters:**
- `factor`: Integer downsampling factor. Must divide the resolution dimensions evenly.

**Steps:**
1. **Validate Divisibility**: Check if the current resolution dimensions $(N_x, N_y, N_z)$ are divisible by `factor`. If $N_i \pmod{\text{factor}} \neq 0$ for any dimension $i \in \{x, y, z\}$, raise a `ValueError` [S1, unofficial].
2. **Downsample**: Apply a max-pooling operation over non-overlapping blocks of size $(\text{factor}, \text{factor}, \text{factor})$ [S1, unofficial][S2, unofficial]. The new data value for each block is the maximum value within that block [S1, unofficial].
   $$ \text{new\_data}[i, j, k] = \max_{\substack{0 \le dx < \text{factor} \\ 0 \le dy < \text{factor} \\ 0 \le dz < \text{factor}}} \text{data}[i \cdot \text{factor} + dx, j \cdot \text{factor} + dy, k \cdot \text{factor} + dz] $$
3. **Construct Output**: Create a new `VoxelGrid` instance. Set its `data` to the downsampled array. Copy `loc` and `scale` from the input grid [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def voxel_grid_down_sample(voxel_grid, factor):
    """
    Downsample a VoxelGrid by a factor using max pooling.
    
    Args:
        voxel_grid: An object with attributes .data (numpy array), .loc, .scale.
        factor: Integer downsampling factor.
        
    Returns:
        A new VoxelGrid with reduced resolution.
    """
    data = voxel_grid.data
    shape = data.shape
    
    # Step 1: Validate divisibility
    for dim_size in shape:
        if dim_size % factor != 0:
            raise ValueError(f"Resolution {dim_size} is not divisible by factor {factor}")
            
    # Step 2: Downsample using max pooling
    # Reshape to group voxels into blocks of size (factor, factor, factor)
    # Shape becomes (Nx/factor, factor, Ny/factor, factor, Nz/factor, factor)
    reshaped = data.reshape(
        shape[0] // factor, factor,
        shape[1] // factor, factor,
        shape[2] // factor, factor
    )
    
    # Take max over the pooling dimensions (axes 1, 3, 5)
    new_data = np.max(reshaped, axis=(1, 3, 5))
    
    # Step 3: Return new VoxelGrid
    # Assuming VoxelGrid is a simple class or namedtuple-like structure
    return VoxelGrid(data=new_data, loc=voxel_grid.loc, scale=voxel_grid.scale)
```

**Edge Cases and Pitfalls:**
- **Non-divisible Resolution**: If the grid dimensions are not multiples of the factor, the method fails. Implementers must ensure input grids are padded or cropped before downsampling [S1, unofficial].
- **Data Type**: The output data type should match the input to preserve precision, though max pooling is generally stable [S1, unofficial].

## Subtopics

### [Volume Resampling and Overflow Guard](1772-volume-resampling-and-overflow-guard.md)
A procedure to resample volumes to a target shape and enforce a maximum voxel count to prevent downstream memory overflow [1772-volume-resampling-and-overflow-guard.md]. It addresses memory constraints that simple downsampling might not explicitly handle if the target resolution is not predefined by a factor.

### [Grid Deformation](5372-grid-deformation.md)
Grid Deformation is a preprocessing technique applied prior to data reduction in volume rendering compression. It transforms the input data grid according to user-defined regions to better preserve salient features during subsequent downsampling. It is an orthogonal approach to hierarchical models like wavelets, reporting improved visual results compared to pure downsampling and wavelet-based methods.

### [Optimization-based Downsampling](5825-optimization-based-downsampling.md)
Optimization-based downsampling is a method for generating low-resolution microflake volume representations that preserve the visual appearance of the reference high-resolution volume, specifically accounting for microscopic self-shadowing effects. It distinguishes itself from simple max-pooling or linear resampling by iteratively optimizing scattering parameters via large-scale training rendering to match the target appearance.

## Operating characteristics
- **Time Complexity**: $O(N)$, where $N$ is the number of voxels in the input grid, as each voxel is visited exactly once during the max operation.
- **Memory Complexity**: $O(N / \text{factor}^3)$ for the output grid. The input grid must remain in memory during the operation.
- **Strengths**: Deterministic, fast, and simple to implement. Preserves the maximum intensity, which is often desirable for density or opacity fields [S1, unofficial][S2, unofficial].
- **Limitations**: Loses all information except the maximum value in each block. Not suitable for data where mean or average intensity is critical. Requires divisible resolutions [S1, unofficial].
- **When to use**: When generating MIPmaps or lower LOD levels for volume rendering where the highest density/opacity is the most important feature to preserve [S1, unofficial].

## Background
The method is derived from the `differentiable_volumetric_rendering` repository by autonomousvision [S1, unofficial]. It is also conceptually aligned with max-pooling layers used in deep learning autoencoders for volume compression, where max-pooling is applied to non-overlapping sub-regions to downsample input and reduce parameters in subsequent encoder layers [S2, unofficial].

## Sources
[S1] autonomousvision/differentiable_volumetric_rendering, https://github.com/autonomousvision/differentiable_volumetric_rendering, fetched 2026-10-26 (unofficial)
[S2] Compressed volume rendering using deep learning, https://www.cs.umd.edu/sites/default/files/scholarly_papers/Jain.pdf, fetched 2026-10-26 (unofficial)
