# Voxel Grid Construction from AABB

Also called: Grid Resolution Setting, VoxelGrid.fromAABB

## Overview
Constructs a uniform voxel grid from an Axis-Aligned Bounding Box (AABB) and a target resolution, ensuring uniform voxel size by scaling based on the longest axis and clamping zero-extent axes [S2, unofficial]. It differs from its sibling [Spherical Voxel Grid Setup](3883-spherical-voxel-grid-setup.md) by enforcing axis-aligned uniformity rather than radial discretization, and from [Incremental Back-Rotation for Voxel Mapping](6759-incremental-back-rotation-for-voxel-mapping.md) by computing the static grid geometry rather than mapping dynamic rotations [G].

## How it works
**Inputs:**
- `bbox_min`, `bbox_max`: 3D vectors defining the minimum and maximum corners of the AABB.
- `resolution`: A positive integer scalar representing the target number of voxels along the longest axis [S2, unofficial].

**Outputs:**
- `voxel_size`: The uniform side length of each voxel.
- `grid_dimensions`: A 3D vector `(count_x, count_y, count_z)` representing the number of voxels along each axis.
- `voxel_grid`: The initialized data structure containing the grid parameters [G].

**Data Structures:**
The core geometric parameters are stored as:
- `voxel_size`: `float`
- `grid_dimensions`: `vec3` (integer type)

**Steps:**
1. **Validate Input:** Verify that `resolution` is a positive integer and that `bbox_min[i] <= bbox_max[i]` for all axes $i \in \{x, y, z\}$ [S2, unofficial].
2. **Calculate AABB Extents:** Compute the extent of the bounding box along each axis:
   $$
   d_i = \max(0, \text{bbox\_max}_i - \text{bbox\_min}_i) \quad \text{for } i \in \{x, y, z\}
   $$
   [S2, unofficial]
3. **Determine Voxel Size:** Find the maximum dimension extent to ensure uniformity:
   $$
   d_{\text{max}} = \max(d_x, d_y, d_z)
   $$
   Calculate the uniform voxel size:
   $$
   \text{voxel\_size} = \frac{d_{\text{max}}}{\text{resolution}}
   $$
   [S2, unofficial]
4. **Calculate Voxel Counts:** For each axis $i$, calculate the number of voxels required to cover the extent $d_i$:
   $$
   \text{count}_i = \left\lceil \frac{d_i}{\text{voxel\_size}} \right\rceil
   $$
   [S2, unofficial]
5. **Handle Zero-Extent Axes:** If any $d_i = 0$ (degenerate axis), clamp the corresponding count to 1 to avoid division by zero or empty grids:
   $$
   \text{if } d_i = 0, \text{ then } \text{count}_i = 1
   $$
   [S2, unofficial]
6. **Construct Grid:** Initialize the voxel grid with dimensions `(count_x, count_y, count_z)` and voxel size `voxel_size` [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def construct_voxel_grid_from_aabb(bbox_min, bbox_max, resolution):
    if not isinstance(resolution, int) or resolution <= 0:
        raise ValueError("Resolution must be a positive integer")
    for i in range(3):
        if bbox_min[i] > bbox_max[i]:
            raise ValueError("Inverted AABB: min must be <= max for all axes")

    dims = [max(0.0, bbox_max[i] - bbox_min[i]) for i in range(3)]
    max_dim = max(dims)

    voxel_size = max_dim / resolution if max_dim > 0 else 1.0

    counts = []
    for d in dims:
        if d == 0:
            counts.append(1)
        else:
            counts.append(math.ceil(d / voxel_size))

    return {"voxel_size": voxel_size, "dimensions": counts}
```

## Subtopics

### [Voxel Grid Initialization & Clamping](13807-voxel-grid-initialization-clamping.md)
Initializes dimensional parameters while enforcing strict hardware vertex limits to prevent GPU buffer overflows [G]. Choose this when memory constraints require dynamic adjustment of voxel counts based on a maximum vertex budget [G].

## Operating characteristics
- **Time Complexity:** $O(1)$ arithmetic operations for grid construction [G].
- **Memory Complexity:** $O(1)$ for storing grid parameters; actual grid memory depends on output dimensions [G].
- **Strengths:** Simple, deterministic, and ensures uniform voxel size which simplifies subsequent sampling and hashing [S2, unofficial].
- **Limitations:** Does not account for aspect ratio optimization; may waste memory if the AABB is highly elongated. Zero-extent axes are clamped to 1, which may not be desired for all applications [S2, unofficial].
- **When to use:** When a uniform grid is required for spatial indexing of splats or points within a known bounding volume [G].

## Background
Originating from implementations like `splatcarve` by stevekwon211, this method is used in 3D Gaussian Splatting pipelines to map splat centers to a discrete voxel grid for efficient carving and rendering [S2, unofficial]. It builds on the coordinate space setup defined in [Voxel Space Setup](3520-voxel-space-setup.md) [G].

## Sources
[S1] Jumpat/SegmentAnythingin3D, https://github.com/Jumpat/SegmentAnythingin3D, fetched 2026-10-26 (unofficial)
[S2] stevekwon211/splatcarve, https://github.com/stevekwon211/splatcarve, fetched 2026-10-27 (unofficial)
