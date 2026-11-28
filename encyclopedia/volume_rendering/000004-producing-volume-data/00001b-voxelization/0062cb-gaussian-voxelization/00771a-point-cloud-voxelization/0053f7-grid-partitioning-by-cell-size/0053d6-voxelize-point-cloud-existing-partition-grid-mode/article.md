# Voxelize Point Cloud (Existing Partition Grid Mode)
Also called: Existing Image Geometry, Existing Partition Grid

## Overview
This method voxelizes a point cloud by mapping points into a pre-existing destination grid geometry defined by its origin, spacing, and dimensions [S1, unofficial]. It solves the problem of integrating point data into a specific, pre-defined volumetric coordinate system [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A point cloud geometry and an existing image geometry defining the destination grid [S1, unofficial].
- Output: A mask or data array associated with the grid geometry [S1, unofficial].

**Data Structures:**
- The destination grid is defined by origin $(o_x, o_y, o_z)$, uniform spacing $s$, and dimensions $D_x, D_y, D_z$ [Grid Partitioning by Cell Size](15111-grid-partitioning-by-cell-size.md)[G].
- The output array uses a tuple shape of $\{dims[2], dims[1], dims[0]\}$ [S1, unofficial].

**Steps:**
1. **Validate Grid Spacing:** Check that the grid spacing is strictly greater than zero [S1, unofficial].
2. **Map Points to Indices:** For each point $p$ with coordinates $(p_x, p_y, p_z)$, compute the voxel indices $(v_x, v_y, v_z)$ using the parent's shared mapping formula:
   $$ v_i = \lfloor (p_i - o_i) / s \rfloor $$
   for each axis $i$ [Grid Partitioning by Cell Size](15111-grid-partitioning-by-cell-size.md).
3. **Clamp/Check Bounds:** Verify that $0 \le v_i < D_i$ for all axes [G].
4. **Write to Grid:** Set the corresponding voxel in the output array to the active value (e.g., `True` or `1`) [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def map_points_to_voxels(points, origin, spacing, dimensions):
    """
    Map 3D points to voxel indices using fixed cell spacing.
    Args:
        points: List of 3D coordinates (x, y, z).
        origin: Tuple (ox, oy, oz) of grid origin.
        spacing: Float cell size s.
        dimensions: Tuple (Dx, Dy, Dz) of grid dimensions.
    Returns:
        mask: 3D boolean array of shape (Dx, Dy, Dz).
    """
    if spacing <= 0:
        raise ValueError("Spacing must be strictly positive for standard mapping.")
    
    inv_spacing = 1.0 / spacing
    mask = [[[False for _ in range(dimensions[2])] for _ in range(dimensions[1])] for _ in range(dimensions[0])]
    
    for p in points:
        vx = int(math.floor((p[0] - origin[0]) * inv_spacing))
        vy = int(math.floor((p[1] - origin[1]) * inv_spacing))
        vz = int(math.floor((p[2] - origin[2]) * inv_spacing))
        
        vx = max(0, min(vx, dimensions[0] - 1))
        vy = max(0, min(vy, dimensions[1] - 1))
        vz = max(0, min(vz, dimensions[2] - 1))
        
        mask[vx][vy][vz] = True
    return mask
```

**Parameters:**
- Grid Origin: Defined by the pre-existing destination grid geometry [S1, unofficial].
- Grid Spacing: Defined by the pre-existing destination grid geometry [S1, unofficial].
- Grid Dimensions: Defined by the pre-existing destination grid geometry [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Invalid Spacing:** If the grid spacing is not larger than zero, the method rejects it with error code -45983 [S1, unofficial].
- **Shape Verification:** The output tuple shape must be explicitly checked as $\{dims[2], dims[1], dims[0]\}$ to ensure correct memory layout [S1, unofficial].
- **Origin Precision:** The grid origin may be padded (e.g., $\approx -0.01$) due to prior resizing operations; this padding is critical for correct alignment [S1, unofficial].

## Subtopics

### Point-Cloud Splatting for Grid Initialization
A strategy to initialize the occupancy grid by treating geometry points as finite-sized entities that splat to nearby grid cells within a radius, rather than as infinitesimal points, to complete small missing regions [S2]. It addresses incomplete occupancy grids resulting from sparsity and errors in the geometry prior [S2]. The key parameter is the splatting radius, which defines the spatial extent for spreading point influence [S2]. A pitfall is the addition of empty grid cells, which is a trade-off for filling missing regions [S2]. This approach yields significant quantitative and qualitative performance improvements, though it cannot completely remove missing regions [S2].

## Operating characteristics
- **Time Complexity:** Not given in the sources. [G]
- **Memory Complexity:** Not given in the sources. [G]
- **Strengths:** Integrates point data into a pre-defined coordinate system; supports out-of-core processing with chunked data stores; validates grid spacing to prevent invalid mappings [S1, unofficial].
- **Limitations:** Requires a valid pre-existing grid geometry; sensitive to grid spacing validation [S1, unofficial].
- **When to Use:** When the target voxel grid geometry is already known or defined by another process [S1, unofficial].
- **When Not to Use:** Not given in the sources. [G]

## Background
This method is implemented as the `VoxelizePointCloudFilter` in the `simplnx` library, supporting geometry types including Edge, Triangle, Quad, Tetrahedral, and Hexahedral [S1, unofficial]. It is designed to work with chunked data stores for out-of-core processing [S1, unofficial]. The validation logic rejecting invalid spacing with error code -45983 is specific to this implementation [S1, unofficial].

## Sources
[S1] FILT: Voxelize Point Cloud, https://github.com/BlueQuartzSoftware/simplnx/pull/1724, fetched 2026-10-27 (unofficial)
[S2] INGeo: Accelerating Instant Neural Scene Reconstruction with Noisy Geometry Priors, https://doi.org/10.1007/978-3-031-25066-8_40, fetched 2026-10-26
