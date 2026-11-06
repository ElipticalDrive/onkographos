# Bounding Box Clipping
Also called: clip (bbox)
## Overview
Bounding Box Clipping returns a new grid containing only the active voxels of a source grid that lie within a specified spatial bounding box [S1, unofficial]. It solves the problem of isolating a specific region of interest or reducing memory footprint by discarding voxels outside a defined extent. Unlike [Mask Clipping](3229-mask-clipping.md), which filters voxels based on the activity state of a secondary mask grid, Bounding Box Clipping filters based on geometric coordinates [S1, unofficial].
## How it works
**Inputs:**
- `source_grid`: The input voxel grid containing active and inactive voxels.
- `bbox`: A bounding box defined by minimum and maximum coordinates (e.g., `min_x, min_y, min_z` and `max_x, max_y, max_z`).

**Outputs:**
- `clipped_grid`: A new grid instance containing only the voxels from `source_grid` whose coordinates satisfy the bounding box constraint.

**Data Structures:**
- The output grid retains the spatial metadata (origin, resolution, voxel size) of the source grid, but its active voxel list is filtered.
- Internal representation of the bounding box is typically axis-aligned (AABB).

**Steps:**
1. Initialize an empty list or buffer `data` to store the active voxels for the new grid.
2. Iterate through all active voxels in the `source_grid`.
3. For each active voxel at coordinate $c$:
   - Check if $c$ is contained within `bbox`.
   - If `contains(bbox, c)` is true, add the voxel (value and coordinate) to `data`.
4. Construct `clipped_grid` using `data` and the original grid's metadata.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def bounding_box_clip(source_grid, bbox):
    """
    Returns a new grid containing only active voxels of source_grid 
    that lie within the specified bbox.
    
    Args:
        source_grid: Object with method get_active_voxels() returning list of (coord, value)
        bbox: Object with method contains(coord) returning boolean
    Returns:
        new_grid: Instance of the grid class with filtered active voxels
    """
    clipped_voxels = []
    
    # Iterate over all active voxels in the source grid
    for coord, value in source_grid.get_active_voxels():
        # Check if the voxel coordinate is inside the bounding box
        if bbox.contains(coord):
            clipped_voxels.append((coord, value))
            
    # Create and return the new grid with the filtered voxels
    # The new grid inherits metadata (origin, resolution) from source_grid
    new_grid = source_grid.create_empty_copy()
    new_grid.set_active_voxels(clipped_voxels)
    return new_grid
```

**Edge Cases and Pitfalls:**
- If no active voxels lie within the bounding box, the resulting grid may be empty or null, depending on implementation [S2, unofficial].
- Coordinate system alignment: The bounding box coordinates must be in the same space as the voxel grid coordinates (e.g., voxel indices vs. world units). Mismatched spaces lead to incorrect clipping [G].
- Precision: Floating-point bounding box boundaries should be handled carefully when comparing against integer voxel indices to avoid off-by-one errors [G].

## Subtopics

### Voxel Body Copying with Sub-Voxel Clipping
This variant handles clipping at the level of solid blocks or "voxel bodies" rather than individual voxels. It calculates the integer bounding box of the volume within the world grid using `TryGetRange`. If no solid range exists, it returns null [S2, unofficial]. It iterates over all solid blocks using `ForEachSolidIn`. For each block, if it is not fully inside the target range, it calls `ClipBlock` with `keepInside: true` to generate a clipped field. If the resulting field is empty, the block is skipped; if it is full, the field is set to null. Otherwise, a new `VoxelBody.Voxel` with the clipped field is added to a `kept` list. If the `kept` list is empty, it returns null; otherwise, it creates a `VoxelBody` using `VoxelBody.Cropped` with the name and the list of voxels [S2, unofficial]. This approach is useful for optimizing memory by preserving block-level structure while trimming extents.

## Operating characteristics
**Time Complexity:** Linear with respect to the number of active voxels in the source grid, $O(N)$, where $N$ is the count of active voxels. The check `contains(bbox, c)` is $O(1)$.
**Memory Complexity:** Proportional to the number of voxels retained in the clipped region.
**Strengths:** Simple to implement; effective for isolating regions of interest; reduces memory usage for large sparse grids.
**Limitations:** Does not compress data; if the bounding box is large, the output grid may still be large. Not suitable for hierarchical compression.
**When to use:** When a specific spatial subset of a volume is needed for rendering or analysis.
**When not to use:** When hierarchical access patterns are required (use [Spatial Indexing](link) instead) or when data needs to be preserved in its original structure.

## Background
Bounding Box Clipping is a fundamental operation in voxel-based data processing, often used in preprocessing steps for volume rendering or spatial queries. It is implemented in various voxel engines and libraries to manage memory and focus computation.
## Sources
[S1] Lyr.jl, https://github.com/tobiasosborne/Lyr.jl, fetched 2026-10-26 (unofficial)
[S2] VoxelTerraformer, https://github.com/GhostlyActive/VoxelTerraformer, fetched 2026-10-27 (unofficial)
