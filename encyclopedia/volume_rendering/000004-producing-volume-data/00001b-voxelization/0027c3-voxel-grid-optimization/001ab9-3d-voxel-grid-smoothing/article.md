# 3D Voxel Grid Smoothing

Also called: 3D Box Filter, Voxel Grid Smoothing

## Overview
3D Voxel Grid Smoothing is a post-processing operation applied to a discrete voxel grid to reduce high-frequency noise and aliasing artifacts, typically prior to isosurface extraction or volume rendering [S1, unofficial]. It functions by applying a separable 3D convolution using a triangular kernel, effectively averaging voxel values with their immediate neighbors [S1, unofficial]. Unlike [Binary Voxel RLE Encoding](1071-binary-voxel-rle-encoding.md) which focuses on compression, or [Occupancy Voxel Creation](2628-occupancy-voxel-creation.md) which focuses on generation, this method modifies the scalar field's continuity to ensure smoother geometric transitions [S1, unofficial].

## How it works
**Inputs:**
- `voxelValues`: A 3D array of floating-point values representing the voxel grid.
- `Resolution`: A 3D vector defining the dimensions of the grid $(W, H, D)$.
- `SmoothIterations`: An integer specifying the number of sequential filter passes.
- `stencil`: A 1D array of weights for the convolution kernel.

**Outputs:**
- `voxelValues`: The smoothed voxel grid, updated in place (via a temporary buffer).

**Data Structures:**
- **Linear Indexing**: The 3D grid is stored in row-major order. The linear index for coordinates $(x, y, z)$ is calculated as:
  $$ \text{INDEX}(x, y, z) = x + y \cdot W + z \cdot W \cdot H $$
  where $W$ is `Resolution.values[0]` and $H$ is `Resolution.values[1]` [S1, unofficial].
- **Temporary Buffer**: A float buffer `_voxelValues` of the same size as `voxelValues` is allocated to store intermediate results for each iteration [S1, unofficial].

**Parameters:**
- **Stencil Weights**: The default 1D kernel is $\{0.5, 1.0, 0.5\}$, representing a triangular filter [S1, unofficial].
- **Iterations**: If `SmoothIterations` is 0, the operation is skipped [S1, unofficial].

**Algorithm Steps:**
1. Allocate `_voxelValues` with the same dimensions as `voxelValues`.
2. Loop `i` from 0 to `SmoothIterations.value - 1`:
   1. Initialize `_voxelValues` to zero.
   2. Initialize `weightSum` to 0.0.
   3. Iterate through every voxel $(x, y, z)$ in the grid bounds ($0 \le x < W, 0 \le y < H, 0 \le z < D$):
      1. Initialize `currentSum` to 0.0.
      2. Iterate through neighbor offsets $(xx, yy, zz)$ where $xx, yy, zz \in \{-1, 0, 1\}$:
         1. Check boundary conditions: if $0 \le x+xx < W$, $0 \le y+yy < H$, and $0 \le z+zz < D$:
            1. Calculate neighbor index: $idx_{neighbor} = \text{INDEX}(x+xx, y+yy, z+zz)$.
            2. Calculate weight: $w = \text{stencil}[xx+1] \cdot \text{stencil}[yy+1] \cdot \text{stencil}[zz+1]$.
            3. Accumulate value: `currentSum` += `voxelValues`[$idx_{neighbor}$] $\cdot w$.
            4. Accumulate weight: `weightSum` += $w$.
      3. Normalize: `_voxelValues`[$\text{INDEX}(x, y, z)$] = `currentSum` / `weightSum`.
   4. Copy `_voxelValues` back to `voxelValues` using `memcpy` or equivalent bulk copy operation.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def smooth_voxel_grid(voxel_values, resolution, smooth_iterations, stencil):
    """
    Applies 3D voxel grid smoothing using a separable triangular filter.
    
    Args:
        voxel_values: 1D list or array of floats, size W*H*D.
        resolution: Tuple (W, H, D).
        smooth_iterations: Integer > 0.
        stencil: List of 3 floats [w_-1, w_0, w_1].
        
    Returns:
        Smoothed voxel_values array.
    """
    W, H, D = resolution
    size = W * H * D
    _voxel_values = [0.0] * size
    
    # Precompute stencil products for all 27 neighbors to avoid repeated multiplication
    # stencil_indices map (dx, dy, dz) to stencil index (0, 1, 2)
    stencil_weights = {}
    for dx in [-1, 0, 1]:
        for dy in [-1, 0, 1]:
            for dz in [-1, 0, 1]:
                w = stencil[dx + 1] * stencil[dy + 1] * stencil[dz + 1]
                stencil_weights[(dx, dy, dz)] = w

    for _ in range(smooth_iterations):
        # Reset temporary buffer
        for i in range(size):
            _voxel_values[i] = 0.0
            
        # Process each voxel
        for z in range(D):
            for y in range(H):
                for x in range(W):
                    current_sum = 0.0
                    weight_sum = 0.0
                    
                    # Iterate over 3x3x3 neighborhood
                    for dx in [-1, 0, 1]:
                        for dy in [-1, 0, 1]:
                            for dz in [-1, 0, 1]:
                                nx, ny, nz = x + dx, y + dy, z + dz
                                
                                # Boundary check
                                if 0 <= nx < W and 0 <= ny < H and 0 <= nz < D:
                                    neighbor_idx = nx + ny * W + nz * W * H
                                    w = stencil_weights[(dx, dy, dz)]
                                    current_sum += voxel_values[neighbor_idx] * w
                                    weight_sum += w
                    
                    # Normalize by valid weight sum (handles boundaries)
                    if weight_sum > 0:
                        idx = x + y * W + z * W * H
                        _voxel_values[idx] = current_sum / weight_sum
                    else:
                        # Should not happen if stencil has non-zero center, 
                        # but guard against empty grid or zero stencil
                        idx = x + y * W + z * W * H
                        _voxel_values[idx] = 0.0

        # Copy back to main buffer
        for i in range(size):
            voxel_values[i] = _voxel_values[i]
            
    return voxel_values
```

## Subtopics

### [3D Box Filter for Voxel Averaging](1513-3d-box-filter-for-voxel-averaging.md)
A 3D convolution operation using a uniform box filter to average voxel values over a receptive field, typically used for smoothing or downsampling the grid. It generalizes the parent method's fixed size 3 kernel to an arbitrary odd-sized cubic kernel for configurable smoothing intensity. Choose this when uniform averaging is preferred over the triangular weighting of the standard smoothing filter, or when downsampling is required.

### NxNxN Cube Filter for Gradient Smoothing
A filtering step applied to gradients to smooth them out and prevent high irregularities. It averages the surrounding $N^3$ samples [S2, unofficial]. It differs from scalar smoothing by operating on vector fields (gradients) and includes a normalization step for the resulting vector [S2, unofficial]. Choose this when smoothing gradient data for transfer functions or normal estimation rather than scalar density. Default filter size is 3 [S2, unofficial].

### Iterative Convolution and Thresholding for Noise Removal
Removes scattered noise voxels from the grid by iteratively applying a convolution with a filled circle kernel and zeroing out voxels below a threshold [S3, unofficial]. It differs from standard smoothing by combining convolution with aggressive thresholding to eliminate isolated noise artifacts rather than just blurring them [S3, unofficial]. Choose this for cleaning sparse or noisy occupancy/alpha channels. Default parameters: kernel size 3, radius 1.0, threshold 2, 20 iterations [S3, unofficial].

## Operating characteristics
**Time Complexity:** $O(I \cdot W \cdot H \cdot D \cdot K^3)$, where $I$ is the number of iterations, $W,H,D$ are grid dimensions, and $K$ is the kernel size (3 for standard smoothing, $N$ for variants). The separable nature of the standard triangular filter allows optimization to $O(I \cdot W \cdot H \cdot D \cdot K)$ if implemented as three 1D passes, though the provided implementation uses the direct 3D convolution approach [S1, unofficial].

**Memory Complexity:** $O(W \cdot H \cdot D)$ for the input/output grid plus $O(W \cdot H \cdot D)$ for the temporary buffer `_voxelValues` [S1, unofficial].

**Strengths:**
- Simple to implement and highly parallelizable.
- Effective at removing high-frequency noise and aliasing.
- Boundary handling via dynamic weight normalization prevents edge artifacts caused by zero-padding [S1, unofficial].

**Limitations:**
- Blurs sharp features and reduces resolution.
- Iterative application increases computational cost linearly.
- Not suitable for preserving fine geometric details or sharp edges without additional edge-preserving techniques.

**When to use:** Pre-processing step before isosurface extraction (e.g., Marching Cubes) to ensure smooth surfaces.
**When not to use:** When sharp boundaries are critical, or when real-time performance is constrained and the grid is already clean.

## Background
The method originates from implementations of isosurface extraction pipelines, specifically those utilizing voxel grids for implicit surface representation [S1, unofficial]. The triangular kernel $\{0.5, 1.0, 0.5\}$ is a standard choice for smoothing as it approximates a Gaussian blur with minimal computational overhead [S1, unofficial]. Variants for gradient smoothing and noise removal have been documented in volume rendering tutorials and open-source implementations for plenoptic and neural voxel representations [S2, unofficial][S3, unofficial].

## Sources
[S1] mkazhdan/IsoSurfaceExtraction, https://github.com/mkazhdan/IsoSurfaceExtraction, fetched 2026-10-27 (unofficial)
[S2] Volume Rendering 102: Transfer Functions, http://graphicsrunner.blogspot.com/2009/01/volume-rendering-102-transfer-functions.html, fetched 2026-10-28 (unofficial)
[S3] DanJbk/Plenoxels, https://github.com/DanJbk/Plenoxels, fetched 2026-10-27 (unofficial)
