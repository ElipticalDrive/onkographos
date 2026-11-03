# Grid Cell Collection via Corner Indices

Also called: 8-Corner Grid Sampling, Trilinear Corner Indexing

## Overview
This method collects the values at the 8 vertices of the grid cell enclosing each input point, providing the necessary data for trilinear interpolation or similar operations [S1]. It differs from [Nearest Voxel Lookup with Periodic Boundary](1573-nearest-voxel-lookup-with-periodic-boundary.md) by retrieving all 8 surrounding corners rather than a single nearest neighbor, enabling smooth field evaluation [S1]. It serves as the foundational indexing step for [3D Trilinear Interpolation](2613-3d-trilinear-interpolation.md) and its SIMD variant [S1].

## How it works
**Inputs:**
- `samples`: A tensor of shape $[N, 3]$ containing $N$ input points with floating-point coordinates $(x, y, z)$.
- `grid_shape`: A tuple $(D_x, D_y, D_z)$ defining the dimensions of the target grid.
- `periodic`: A boolean flag indicating whether to apply periodic boundary conditions for out-of-bounds indices.

**Outputs:**
- `corner_indices`: A tensor of shape $[N, 8, 3]$ containing the integer indices of the 8 corners for each point.
- `corner_values`: A tensor of shape $[N, 8, C]$ (where $C$ is the channel count) containing the values at those corners, retrieved from the grid [S1].

**Data Structures:**
- The grid is stored as a dense tensor. Indices are 0-based integers.
- Corner indices are stored in a fixed order corresponding to the binary representation of the corner offset: $(i, j, k)$ where $i,j,k \in \{0,1\}$ correspond to floor/ceil respectively [S1].

**Steps:**
1. **Compute Floor and Ceil Coordinates:** For each point $(x, y, z)$, compute the floor and ceil coordinates for each dimension.
   $$
   x_{floor} = \lfloor x \rfloor, \quad x_{ceil} = \lceil x \rceil
   $$
   Repeat for $y$ and $z$ [S1].

2. **Generate Corner Combinations:** Create all 8 combinations of $(x_{floor}/x_{ceil}, y_{floor}/y_{ceil}, z_{floor}/z_{ceil})$ to form the corner indices [S1].

3. **Reshape and Flatten:** Reshape the corner indices tensor to a flat list of points [S1].

4. **Handle Out-of-Bounds Indices:** Fix out-of-bounds indices using periodic boundary conditions (modulo arithmetic) or clamping [S1].
   $$
   idx_{wrapped} = idx \mod D
   $$

5. **Index into Grid:** Use the corrected integer indices to gather values from the grid tensor [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def collect_corner_indices(samples, grid_shape, periodic=True):
    """
    Collects corner indices for trilinear interpolation.
    
    Args:
        samples: List of [N, 3] float coordinates.
        grid_shape: Tuple (Dx, Dy, Dz) of grid dimensions.
        periodic: Boolean, if True, wrap indices; else clamp.
        
    Returns:
        corner_indices: List of [N, 8, 3] integer indices.
    """
    N = len(samples)
    Dx, Dy, Dz = grid_shape
    corners = []
    
    # Step 1 & 2: Compute floor/ceil and generate 8 combinations
    for point in samples:
        x, y, z = point
        xf, xc = math.floor(x), math.ceil(x)
        yf, yc = math.floor(y), math.ceil(y)
        zf, zc = math.floor(z), math.ceil(z)
        
        # Generate all 8 corner combinations
        for cx in (xf, xc):
            for cy in (yf, yc):
                for cz in (zf, zc):
                    corners.append([cx, cy, cz])
    
    # Step 3: Reshape to [N, 8, 3]
    corner_indices = [corners[i*8:(i+1)*8] for i in range(N)]
    
    # Step 4: Handle boundaries
    if periodic:
        corner_indices = [[(idx % D) for idx, D in zip(c, grid_shape)] for c in corner_indices]
    else:
        corner_indices = [[max(0, min(idx, D-1)) for idx, D in zip(c, grid_shape)] for c in corner_indices]
        
    # Step 5: Index into Grid (conceptual; actual gathering occurs in downstream kernel)
    return corner_indices
```

**Parameters:**
- `periodic`: Controls boundary wrapping behavior [S1].

**Edge Cases:**
- **Precision:** Floating-point coordinates exactly on integer boundaries yield identical floor and ceil indices; interpolation weights handle the zero-width interval naturally [G].
- **Out-of-Bounds:** If `periodic` is false and points lie outside the grid, clamping is required to prevent memory access errors [G].

## Subtopics

### [VoxelGrid](2159-voxelgrid.md)
A dense, learnable 3D structure that stores values at voxel centers but uses corner indices for differentiable trilinear interpolation. It replaces coordinate-based MLPs with a fixed, axis-aligned tensor layout for efficient neural rendering. Choose it when differentiable rendering without MLP overhead is required.

### [Curvilinear to Rectilinear Resampling](2248-curvilinear-to-rectilinear-resampling.md)
Converts a curilinear input grid into a rectilinear output grid by estimating function values at new sample points. It differs from sibling methods by explicitly handling geometric distortion, requiring a cell-localization step before interpolation, whereas rectilinear grids assume a fixed, orthogonal topology. Choose it when remapping distorted domains to regular grids.

### [Voxel Trilinear Interpolation (SIMD)](2564-voxel-trilinear-interpolation-simd.md)
Performs trilinear interpolation of scalar voxel values from a 3D brick using AVX/AVX2 128-bit registers. It packs two voxel samples into registers and computes their contributions in parallel, effectively doubling throughput for single-channel data. It builds on this method by consuming the corner indices and values. Choose it when hardware-accelerated throughput is critical.

### [3D Trilinear Interpolation](2613-3d-trilinear-interpolation.md)
Retrieves the value of a scalar or vector field at arbitrary floating-point coordinates by interpolating the values at the 8 integer corners of the enclosing voxel grid cell. It is used in volume rendering and raycasting to achieve higher sampling resolution than nearest-neighbor sampling, providing smooth transitions between discrete voxel values. It differs from its SIMD sibling by focusing on the standard algorithmic definition rather than vectorized hardware optimizations, and from compressed variants by operating on dense, contiguous memory layouts without link indirection. Choose it for standard smooth field evaluation.

### [iso_corners_list_f1v4](2627-iso-corners-list-f1v4.md)
A GPU kernel variant for computing grid cell corner indices from a list of input points, supporting 1, 8, 16, 32, or 64 functions. It builds on this method by processing 256 consecutive lattice points per group across 2–4 neighboring bricks, maximizing thread occupancy by sharing shell setup overhead across the group rather than per-brick. Choose it for high-occupancy GPU rendering pipelines.

### UniformGrid Cell Vertex and Material Retrieval
Retrieves the linear indices of the 8 vertices of a cell given its linear index, and retrieves the materials of those vertices [S2, unofficial]. It uses the formula $dx = m_{size} + 1$, $dy = (m_{size} + 1)^2$, $dz = 1$, with vertices $v[0]=cellIdx$, $v[1]=cellIdx+dz$, $v[2]=cellIdx+dx$, $v[3]=cellIdx+dx+dz$, $v[4]=cellIdx+dy$, $v[5]=cellIdx+dy+dz$, $v[6]=cellIdx+dx+dy$, $v[7]=cellIdx+dx+dy+dz$ [S2, unofficial]. Choose it when working with uniform grid topologies requiring direct vertex/material lookup.

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of input points, with a small constant factor of 8 lookups per point [S1].
- **Memory Complexity:** $O(N)$ to store the corner indices and values [S1].
- **Performance:** Highly efficient on GPUs due to coalesced memory access patterns when points are batched [G]. The SIMD variant [Voxel Trilinear Interpolation (SIMD)](2564-voxel-trilinear-interpolation-simd.md) further accelerates this by packing samples into 128-bit registers [S1, unofficial].
- **Strengths:** Simple, differentiable, and compatible with dense grid layouts [G].
- **Limitations:** Requires dense memory allocation for the grid; not suitable for sparse data without significant overhead [G].

## Background
This method provides the indexing foundation for trilinear interpolation in grid-based neural rendering [S1]. It enables the replacement of coordinate-based MLPs with fixed, axis-aligned tensor layouts, allowing for efficient differentiable rendering [S1, unofficial]. The approach is widely used in volume rendering and raycasting to achieve smooth transitions between discrete voxel values [S1].

## Sources
[S1] DanJbk/Plenoxels, https://github.com/DanJbk/Plenoxels, fetched 2026-10-27 (unofficial)
[S2] Svenny/isomesh, https://github.com/Svenny/isomesh, fetched 2026-10-27 (unofficial)
