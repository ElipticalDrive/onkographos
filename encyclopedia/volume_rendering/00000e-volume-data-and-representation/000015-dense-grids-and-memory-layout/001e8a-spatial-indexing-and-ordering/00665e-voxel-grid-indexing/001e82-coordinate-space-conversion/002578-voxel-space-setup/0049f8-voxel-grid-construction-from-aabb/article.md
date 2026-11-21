# Voxel Grid Construction from AABB

Also called: VoxelGrid.fromAABB

## Overview
Constructs a uniform voxel grid from an Axis-Aligned Bounding Box (AABB) and a target resolution, ensuring uniform voxel size by scaling based on the longest axis and clamping zero-extent axes [S1, unofficial]. It differs from its sibling [Spherical Voxel Grid Setup](3883-spherical-voxel-grid-setup.md) by enforcing axis-aligned uniformity rather than radial discretization, and from [Incremental Back-Rotation for Voxel Mapping](6759-incremental-back-rotation-for-voxel-mapping.md) by computing the static grid geometry rather than mapping dynamic rotations.

## How it works
**Inputs:**
- `bbox`: An AABB defined by minimum corner `min` and maximum corner `max` (3D vectors).
- `resolution`: A positive integer scalar representing the target number of voxels along the longest axis.

**Outputs:**
- `voxelSize`: The uniform side length of each voxel.
- `gridDimensions`: A 3D vector `(countX, countY, countZ)` representing the number of voxels along each axis.
- `voxelGrid`: The initialized data structure (details depend on the specific implementation, e.g., [Voxel Grid Initialization & Clamping](13807-voxel-grid-initialization-clamping.md)).

**Data Structures:**
The core geometric parameters are stored as:
- `voxelSize`: `float`
- `gridDimensions`: `vec3` (uint32 or int32)

**Steps:**
1. **Validate Input:**
   - Check that `resolution` is a positive integer. If not, reject and throw an error [S1, unofficial].
   - Check that `bbox` is valid (i.e., `min[i] <= max[i]` for all axes $i \in \{x, y, z\}$). If `min > max` (inverted AABB), reject and throw an error [S1, unofficial].

2. **Calculate AABB Dimensions:**
   Compute the extent of the bounding box along each axis:
   $$
   d_i = \max(0, \text{max}_i - \text{min}_i) \quad \text{for } i \in \{x, y, z\}
   $$
   [S1, unofficial]

3. **Determine Voxel Size:**
   Find the maximum dimension extent to ensure uniformity:
   $$
   d_{\text{max}} = \max(d_x, d_y, d_z)
   $$
   Calculate the uniform voxel size:
   $$
   \text{voxelSize} = \frac{d_{\text{max}}}{\text{resolution}}
   $$
   [S1, unofficial]

4. **Calculate Voxel Counts:**
   For each axis $i$, calculate the number of voxels required to cover the extent $d_i$:
   $$
   \text{count}_i = \left\lceil \frac{d_i}{\text{voxelSize}} \right\rceil
   $$
   [S1, unofficial]

5. **Handle Zero-Extent Axes:**
   If any $d_i = 0$ (degenerate axis), clamp the corresponding count to 1 to avoid division by zero or empty grids:
   $$
   \text{if } d_i = 0, \text{ then } \text{count}_i = 1
   $$
   [S1, unofficial]

6. **Construct Grid:**
   Initialize the voxel grid with dimensions `(countX, countY, countZ)` and voxel size `voxelSize`. Apply any necessary initialization or clamping steps as defined in [Voxel Grid Initialization & Clamping](13807-voxel-grid-initialization-clamping.md) to ensure hardware compatibility [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def construct_voxel_grid_from_aabb(bbox_min, bbox_max, resolution):
    # 1. Validate resolution
    if not isinstance(resolution, int) or resolution <= 0:
        raise ValueError("Resolution must be a positive integer")

    # 2. Validate AABB
    for i in range(3):
        if bbox_min[i] > bbox_max[i]:
            raise ValueError("Inverted AABB: min must be <= max for all axes")

    # 3. Calculate dimensions
    dims = [max(0.0, bbox_max[i] - bbox_min[i]) for i in range(3)]
    max_dim = max(dims)

    # 4. Calculate voxel size
    if max_dim == 0:
        # Degenerate AABB with zero volume
        voxel_size = 0.0
    else:
        voxel_size = max_dim / resolution

    # 5. Calculate counts
    counts = []
    for d in dims:
        if d == 0:
            counts.append(1)
        else:
            # Ceiling division: ceil(a / b) = (a + b - 1) // b for integers, 
            # but here we use float math then ceil
            import math
            counts.append(math.ceil(d / voxel_size) if voxel_size > 0 else 1)

    # 6. Return parameters
    return {
        "voxel_size": voxel_size,
        "dimensions": counts
    }
```

## Subtopics

### [Voxel Grid Initialization & Clamping](13807-voxel-grid-initialization-clamping.md)
Initializes dimensional parameters while enforcing strict hardware vertex limits to prevent GPU buffer overflows. Choose this when memory constraints require dynamic adjustment of voxel counts based on a maximum vertex budget.

## Operating characteristics
- **Time Complexity:** $O(1)$ arithmetic operations for grid construction.
- **Memory Complexity:** $O(1)$ for storing grid parameters; actual grid memory depends on output dimensions.
- **Strengths:** Simple, deterministic, and ensures uniform voxel size which simplifies subsequent sampling and hashing [S1, unofficial].
- **Limitations:** Does not account for aspect ratio optimization; may waste memory if the AABB is highly elongated. Zero-extent axes are clamped to 1, which may not be desired for all applications [S1, unofficial].
- **When to use:** When a uniform grid is required for spatial indexing of splats or points within a known bounding volume.

## Background
Originating from implementations like `splatcarve` by stevekwon211, this method is used in 3D Gaussian Splatting pipelines to map splat centers to a discrete voxel grid for efficient carving and rendering [S1, unofficial]. It builds on the coordinate space setup defined in [Voxel Space Setup](3520-voxel-space-setup.md).

## Sources
[S1] stevekwon211/splatcarve, https://github.com/stevekwon211/splatcarve, fetched 2026-10-27 (unofficial)
