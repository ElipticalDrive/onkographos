# 3D Voxel Grid Smoothing
Also called: 3D Box Filter, Voxel Grid Smoothing

## Overview
3D Voxel Grid Smoothing is a preprocessing technique applied to 3D voxel grids to reduce high-frequency noise and aliasing artifacts prior to isosurface extraction or volume rendering [S1, unofficial]. It operates by applying a separable 3D convolution using a specific 1D kernel, effectively averaging voxel values with their immediate neighbors [S1, unofficial]. Unlike its sibling [3D Box Filter for Voxel Averaging](1513-3d-box-filter-for-voxel-averaging.md), which generalizes to arbitrary kernel sizes, this method specifically utilizes a fixed 3-tap triangular filter [S1, unofficial]. It differs from [Iterative Convolution and Thresholding for Noise Removal](1516-iterative-convolution-and-thresholding-for-noise-removal.md) by preserving the continuous scalar field values rather than acting as a binary morphological filter [S1, unofficial].

## How it works
**Inputs:**
- `voxelValues`: A 3D array of floating-point values representing the voxel grid.
- `Resolution`: A 3D vector $(R_x, R_y, R_z)$ defining the dimensions of the grid.
- `SmoothIterations`: An integer specifying the number of sequential passes to apply the filter.

**Outputs:**
- `voxelValues`: The modified grid with smoothed values.

**Data Structures:**
- **Linear Indexing:** The 3D grid is stored in a 1D contiguous buffer. The index for coordinates $(x, y, z)$ is calculated as:
  $$ \text{INDEX}(x, y, z) = x + y \cdot R_x + z \cdot R_x \cdot R_z $$
  where $x$ is the fastest varying dimension and $z$ is the slowest [S1, unofficial].
- **Stencil:** A 1D array of weights defining the convolution kernel. For this method, the stencil is fixed as $\{0.5, 1.0, 0.5\}$ [S1, unofficial]. This corresponds to a triangular filter in 1D.
- **Temporary Buffer:** A float buffer `_voxelValues` of the same size as `voxelValues` is allocated to store intermediate results during each iteration [S1, unofficial].

**Algorithm Steps:**
1. If `SmoothIterations` is 0, terminate immediately [S1, unofficial].
2. Allocate `_voxelValues` with size $R_x \cdot R_y \cdot R_z$ [S1, unofficial].
3. Loop `iter` from 0 to `SmoothIterations - 1`:
   1. Initialize `_voxelValues` to zero.
   2. Initialize `weightSum` to 0.0 for the current voxel (reset per voxel).
   3. Iterate through every voxel $(x, y, z)$ in the grid bounds:
      1. Set `currentSum` = 0.0 and `currentWeight` = 0.0.
      2. Iterate through neighbor offsets $(dx, dy, dz)$ where $dx, dy, dz \in \{-1, 0, 1\}$:
         1. Calculate neighbor coordinates: $nx = x + dx$, $ny = y + dy$, $nz = z + dz$.
         2. Check boundary conditions: if $0 \le nx < R_x$, $0 \le ny < R_y$, and $0 \le nz < R_z$:
            1. Retrieve neighbor value: $v = \text{voxelValues}[\text{INDEX}(nx, ny, nz)]$.
            2. Retrieve stencil weights: $w_x = \text{stencil}[dx + 1]$, $w_y = \text{stencil}[dy + 1]$, $w_z = \text{stencil}[dz + 1]$.
            3. Calculate combined weight: $w = w_x \cdot w_y \cdot w_z$.
            4. Accumulate weighted value: $\text{currentSum} \ += v \cdot w$.
            5. Accumulate weight: $\text{currentWeight} \ += w$.
      3. Normalize the result: $\text{result} = \text{currentSum} / \text{currentWeight}$.
      4. Store in temporary buffer: $\text{_voxelValues}[\text{INDEX}(x, y, z)] = \text{result}$.
   4. Copy `_voxelValues` back to `voxelValues` using a memory copy operation (e.g., `memcpy`) [S1, unofficial].

**Parameters:**
- `stencil`: $\{0.5, 1.0, 0.5\}$ [S1, unofficial].
- `SmoothIterations`: Typically 1 or more. If 0, smoothing is skipped [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Boundary Handling:** Neighbors outside the grid bounds are excluded from the summation and weight calculation. This effectively implements zero-padding or clamping by only summing valid neighbors [S1, unofficial].
- **Normalization:** The division by `weightSum` (or `currentWeight` in the pseudocode) ensures correct normalization even at edges and corners where fewer neighbors are available, preventing intensity bias at the boundaries [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def smooth_voxel_grid(voxel_values, resolution, smooth_iterations):
    if smooth_iterations == 0:
        return voxel_values

    rx, ry, rz = resolution
    size = rx * ry * rz
    temp_values = [0.0] * size
    
    # Stencil weights for 1D kernel {0.5, 1.0, 0.5}
    # Index mapping: offset -1 -> index 0, 0 -> index 1, 1 -> index 2
    stencil = [0.5, 1.0, 0.5]
    
    for _ in range(smooth_iterations):
        # Reset temp buffer
        for i in range(size):
            temp_values[i] = 0.0
            
        for z in range(rz):
            for y in range(ry):
                for x in range(rx):
                    current_sum = 0.0
                    current_weight = 0.0
                    
                    # Iterate over 3x3x3 neighborhood
                    for dx in [-1, 0, 1]:
                        for dy in [-1, 0, 1]:
                            for dz in [-1, 0, 1]:
                                nx = x + dx
                                ny = y + dy
                                nz = z + dz
                                
                                # Boundary check
                                if 0 <= nx < rx and 0 <= ny < ry and 0 <= nz < rz:
                                    # Calculate linear index for neighbor
                                    neighbor_idx = nx + ny * rx + nz * rx * ry
                                    neighbor_val = voxel_values[neighbor_idx]
                                    
                                    # Get stencil weights
                                    wx = stencil[dx + 1]
                                    wy = stencil[dy + 1]
                                    wz = stencil[dz + 1]
                                    
                                    w = wx * wy * wz
                                    current_sum += neighbor_val * w
                                    current_weight += w
                    
                    # Normalize
                    if current_weight > 0:
                        result = current_sum / current_weight
                    else:
                        result = 0.0
                        
                    # Store in temp buffer
                    idx = x + y * rx + z * rx * ry
                    temp_values[idx] = result
        
        # Copy temp back to main buffer
        for i in range(size):
            voxel_values[i] = temp_values[i]
            
    return voxel_values
```

## Subtopics

### [NxNxN Cube Filter for Gradient Smoothing](864-nxnxn-cube-filter-for-gradient-smoothing.md)
Smooths gradient vectors in a 3D voxel grid by averaging gradients within a cubic neighborhood. It solves high-frequency irregularities in gradient fields for isosurface extraction. It differs by operating on vector-valued data rather than scalar intensities and uses a single-pass filter without iterative refinement.

### [3D Box Filter for Voxel Averaging](1513-3d-box-filter-for-voxel-averaging.md)
A 3D convolution using a uniform box filter to average voxel values over a configurable receptive field. It generalizes the fixed 3-tap kernel of the parent method to arbitrary odd-sized cubic kernels.

### [Iterative Convolution and Thresholding for Noise Removal](1516-iterative-convolution-and-thresholding-for-noise-removal.md)
Removes scattered noise voxels by iteratively convolving with a filled circle kernel and zeroing values below a threshold. It acts as a morphological filter for binary/alpha-masked structures, unlike the scalar smoothing of the parent.

## Operating characteristics
- **Time Complexity:** $O(I \cdot N)$, where $I$ is the number of smoothing iterations and $N$ is the total number of voxels ($R_x \cdot R_y \cdot R_z$). Each iteration processes every voxel and its 27 neighbors.
- **Memory Complexity:** $O(N)$ for the input/output grid plus $O(N)$ for the temporary buffer `_voxelValues` [S1, unofficial].
- **Performance:** The method is computationally lightweight due to the small kernel size (3x3x3) and separable nature. However, multiple iterations increase the cost linearly.
- **Strengths:** Simple to implement, effective at reducing aliasing and noise, handles boundaries gracefully via dynamic normalization.
- **Limitations:** Blurs sharp features due to averaging. The fixed kernel size limits flexibility compared to sibling methods.
- **When to use:** Preprocessing step before isosurface extraction or volume rendering when noise reduction is required.
- **When not to use:** When preserving sharp edges or high-frequency details is critical, or when memory bandwidth is a severe bottleneck and multiple iterations are needed.

## Background
The method originates from the IsoSurfaceExtraction project by Michael Kazhdan [S1, unofficial]. It is part of a broader family of voxel grid optimization techniques aimed at reducing rendering costs and improving data quality for subsequent processing steps like marching cubes or ray marching. The specific implementation uses a separable convolution approach to efficiently apply smoothing in 3D space.

## Sources
[S1] mkazhdan/IsoSurfaceExtraction, https://github.com/mkazhdan/IsoSurfaceExtraction, fetched 2026-10-27 (unofficial)
