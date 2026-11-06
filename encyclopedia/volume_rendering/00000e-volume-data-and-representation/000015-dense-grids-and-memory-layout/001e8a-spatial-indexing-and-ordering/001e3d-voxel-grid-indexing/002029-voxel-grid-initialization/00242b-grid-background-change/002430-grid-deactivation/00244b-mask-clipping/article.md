# Mask Clipping
Also called: clip (mask)
## Overview
Mask Clipping returns a new grid containing only the active voxels of a source grid at coordinates where a secondary mask grid is also active [S1, unofficial]. It serves as a spatial filter to isolate a region of interest or reduce memory footprint by discarding voxels outside the target volume [S1, unofficial]. Unlike [Bounding Box Clipping](4643-bounding-box-clipping.md), which uses geometric coordinates, Mask Clipping uses a secondary binary grid to define the active region, making it suitable for arbitrary non-rectangular sub-volumes without requiring a pre-computed mask grid.
## How it works
**Inputs:**
1. `source_grid`: The input volume grid containing active voxels.
2. `mask_grid`: A secondary grid of the same resolution and layout, where active voxels indicate inclusion.

**Outputs:**
1. `clipped_grid`: A new grid containing only the active voxels from `source_grid` that correspond to active voxels in `mask_grid`.

**Data Structures:**
The implementation relies on the standard sparse grid representation (e.g., OpenVDB tree structure) where active voxels are stored in a linked list or hash map indexed by coordinate. The mask grid must support efficient lookup of voxel activity status at arbitrary coordinates.

**Steps:**
1. Initialize an empty `clipped_grid`.
2. Iterate through each active voxel in `source_grid`. Let the current voxel's coordinate be $c$.
3. For each coordinate $c$, check if the corresponding voxel in `mask_grid` is active. This is denoted as `is_active(mask_grid, c)`.
4. If `is_active(mask_grid, c)` is true, copy the value and metadata of the voxel at $c$ from `source_grid` into `clipped_grid`.
5. If `is_active(mask_grid, c)` is false, skip the voxel.
6. Return `clipped_grid`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mask_clip(source_grid, mask_grid):
    clipped_grid = Grid()
    for voxel in source_grid.active_voxels():
        coord = voxel.coordinate
        if mask_grid.is_active(coord):
            clipped_grid.insert(coord, voxel.value, voxel.metadata)
    return clipped_grid
```

**Parameters:**
- None specific to the algorithm logic. The resolution and layout of `mask_grid` must match `source_grid`.

**Edge Cases and Pitfalls:**
- **Resolution Mismatch:** If `mask_grid` has a different resolution or origin than `source_grid`, coordinate mapping must be handled explicitly. Not given in the sources.
- **Empty Mask:** If `mask_grid` has no active voxels, the result is an empty grid.
- **All Active Mask:** If `mask_grid` is fully active, the result is identical to `source_grid`.
- **Precision:** Coordinate comparisons must be exact. Floating-point coordinates should be discretized to integer grid indices before lookup. [G]

## Subtopics

### [Bounding Box Clipping](4643-bounding-box-clipping.md)
Bounding Box Clipping returns a new grid containing only the active voxels of a source grid that lie within a specified axis-aligned bounding box (AABB). It serves as a spatial filter to isolate a region of interest or reduce memory footprint by discarding voxels outside the target volume. Unlike [Mask Clipping](3229-mask-clipping.md), which uses a secondary binary grid to define the active region, Bounding Box Clipping uses geometric coordinates, making it suitable for rectangular sub-volumes without requiring a pre-computed mask grid.

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the number of active voxels in `source_grid`. Each active voxel requires a constant-time lookup in `mask_grid`.
- **Memory Complexity:** $O(M)$, where $M$ is the number of active voxels in the output `clipped_grid`. The mask grid itself must be stored or accessible.
- **Measured Performance:** Not given in the sources.
- **Strengths:** Allows for arbitrary, non-convex, or complex spatial filtering. Can be combined with other mask generation techniques (e.g., from segmentation or procedural noise).
- **Limitations:** Requires storing or accessing an additional grid (`mask_grid`), increasing memory usage. Performance depends on the efficiency of the mask grid's lookup structure.
- **When to Use:** When the region of interest is irregular or defined by another volume dataset.
- **When Not to Use:** When the region is a simple axis-aligned box; use [Bounding Box Clipping](4643-bounding-box-clipping.md) instead for better performance and simplicity.

## Background
Mask Clipping is a standard operation in volume data processing pipelines, particularly in scientific visualization and medical imaging, where regions of interest are often defined by secondary segmentation masks or thresholded volumes. It is commonly used in conjunction with [Grid Deactivation](3214-grid-deactivation.md) to refine the active set of voxels. The method is implemented in various volume rendering frameworks, including Lyr.jl, where it is used to prepare data for rendering by discarding irrelevant voxels [S1, unofficial].

## Sources
[S1] tobiasosborne/Lyr.jl, https://github.com/tobiasosborne/Lyr.jl, fetched 2026-10-26 (unofficial)
