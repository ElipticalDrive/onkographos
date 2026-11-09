# Mask Clipping
Also called: clip (mask)
## Overview
Mask Clipping returns a new grid containing only the active voxels of a source grid at coordinates where a secondary mask grid is also active [S1, unofficial]. It differs from [Bounding Box Clipping](4643-bounding-box-clipping.md) by using a secondary data grid rather than a geometric primitive to define the region of interest [S1, unofficial].
## How it works
**Inputs:**
1. `source_grid`: A volume grid containing active voxels [S1, unofficial].
2. `mask_grid`: A secondary grid where active voxels indicate inclusion [S1, unofficial].
**Outputs:**
1. `clipped_grid`: A new grid containing only the active voxels from `source_grid` that correspond to active voxels in `mask_grid` [S1, unofficial].
**Data Structures:**
Not given in the sources.
**Steps:**
1. Initialize an empty `clipped_grid` [S1, unofficial].
2. Iterate through each active voxel in `source_grid` [S1, unofficial]. Let the current voxel's coordinate be $c$ [S1, unofficial].
3. Check if the corresponding voxel in `mask_grid` is active [S1, unofficial]. This is denoted as `is_active(mask_grid, c)` [S1, unofficial].
4. If `is_active(mask_grid, c)` is true, add the voxel to `clipped_grid` [S1, unofficial].
5. If `is_active(mask_grid, c)` is false, skip the voxel [S1, unofficial].
6. Return `clipped_grid` [S1, unofficial].
**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mask_clip(source_grid, mask_grid):
    # source_grid and mask_grid are dictionaries mapping coordinates to values
    clipped_grid = {}
    for coord, value in source_grid.items():
        if coord in mask_grid:  # Equivalent to is_active(mask_grid, coord)
            clipped_grid[coord] = value
    return clipped_grid
```
**Parameters:**
Not given in the sources.
**Edge Cases and Pitfalls:**
Not given in the sources.
## Subtopics

### [Bounding Box Clipping](4643-bounding-box-clipping.md)
Bounding Box Clipping returns a new grid containing only the active voxels of the source grid that lie within a specified world-space bounding box. It differs from [Mask Clipping](3229-mask-clipping.md) by using a geometric primitive (axis-aligned box) rather than a secondary data grid to define the region of interest, and from [Grid Clone](7232-grid-clone.md) by filtering spatial extent rather than duplicating the full logical structure.

## Operating characteristics
Not given in the sources.
## Background
Not given in the sources.
## Sources
[S1] tobiasosborne/Lyr.jl, https://github.com/tobiasosborne/Lyr.jl, fetched 2026-10-26 (unofficial)
