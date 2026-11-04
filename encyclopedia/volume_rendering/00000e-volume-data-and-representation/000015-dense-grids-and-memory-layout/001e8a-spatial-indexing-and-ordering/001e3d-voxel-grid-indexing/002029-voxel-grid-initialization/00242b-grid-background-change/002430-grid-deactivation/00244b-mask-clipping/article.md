# Mask Clipping
Also called: clip (mask)
## Overview
Mask Clipping returns a new grid containing only active voxels of a source grid at coordinates where a mask grid is also active [S1, unofficial].
## How it works
**Inputs:**
1. `source_grid`: A grid containing active voxels [S1, unofficial].
2. `mask_grid`: A grid defining the spatial inclusion criterion [S1, unofficial].
**Outputs:**
1. `clipped_data`: A list of `(coordinate, value)` tuples representing the active voxels for the new grid [S1, unofficial].
**Data Structures:**
The method operates on sparse grid representations where active voxels are stored in a traversable structure [G]. The mask grid must support efficient coordinate queries [G].
**Steps:**
1. Initialize an empty `clipped_data` list [G].
2. Iterate through every active voxel in `source_grid` [S1, unofficial].
3. For each voxel at coordinate $c$ with value $v$, query the mask grid [S1, unofficial].
4. If `is_active(mask_grid, c)` is true, append $(c, v)$ to `clipped_data` [S1, unofficial].
5. Return `clipped_data` [G].
**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mask_clip(source_grid, mask_grid):
    # Initialize empty list for clipped voxel data [G]
    clipped_data = []
    # Assume source_grid yields (coord, value) pairs for active voxels [G]
    for coord, value in source_grid.iterate_active():
        # Query mask grid for coordinate inclusion [S1, unofficial]
        if mask_grid.is_active(coord):
            clipped_data.append((coord, value))  # Add to data [S1, unofficial]
    return clipped_data  # Represents the active voxels for the new grid [G]
```
**Edge Cases:**
Not given in the sources.
## Subtopics

### Bounding Box Clipping
Returns a new grid containing only active voxels of the source grid that lie within a specified bounding box [S1, unofficial]. Choose this when the region of interest is a simple rectangular volume, as it avoids the overhead of loading and querying a separate mask grid [G]. The step involves iterating active voxels and checking if `contains(bbox, c)` is true [S1, unofficial].

## Operating characteristics
Not given in the sources.
## Background
Not given in the sources.
## Sources
[S1] tobiasosborne/Lyr.jl, https://github.com/tobiasosborne/Lyr.jl, fetched 2026-10-26 (unofficial)
