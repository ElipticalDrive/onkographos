# VoxelGridFromList


## Overview
`VoxelGridFromList` initializes a 3D voxel grid by copying boolean values from a flat list into the grid's linear memory layout. It solves the problem of bulk-setting voxel states (e.g., occupancy or SDF signs) without iterating over spatial coordinates, relying instead on a pre-computed or assumed linear index mapping. It differs from [Voxel Grid Construction](3085-voxel-grid-construction.md) by assuming the grid structure already exists and only updating values, and from [Dense Copy Ingestion](3212-dense-copy-ingestion.md) by handling boolean flags rather than dense scalar data.

## How it works
**Inputs:**
1. `grid`: A `VoxelGrid3D` object (or compatible scalar/boolean grid) at input index 0.
2. `values`: A list or array of boolean values at input index 1.

**Outputs:**
1. `modified_grid`: A new `VoxelGrid3D` instance with updated values at input index 0.

**Data Structures:**
- `VoxelGrid3D`: Contains a linear array of values, a size vector `SizeUVW` (width, height, depth), and a bounding box `BBox`. The linear index $i$ maps to voxel coordinates $(u, v, w)$ via $i = u + v \cdot W + w \cdot W \cdot H$, where $W, H, D$ are dimensions.

**Steps:**
1. Retrieve the input grid from index 0 and cast it to `VoxelGrid3D`.
2. Clone the retrieved grid to create `modified_grid`. This ensures the original grid is not mutated.
3. Retrieve the list of boolean values from index 1.
4. Calculate `max_count` as the minimum of the grid's total voxel count ($W \times H \times D$) and the input list's length ($L$).
   $$ \text{max\_count} = \min(W \cdot H \cdot D, L) $$
5. Iterate from index $i = 0$ to `max_count` (exclusive).
6. Set the value at linear index $i$ in `modified_grid` to the value at index $i$ in the input list.
7. Set `modified_grid` as the output at index 0.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def VoxelGridFromList(grid_input, values_input):
    # Step 1: Retrieve and cast grid
    source_grid = grid_input[0]
    # Ensure it is a VoxelGrid3D compatible object
    assert isinstance(source_grid, VoxelGrid3D)
    
    # Step 2: Clone the grid
    modified_grid = source_grid.clone()
    
    # Step 3: Retrieve values
    values = values_input[1]
    
    # Step 4: Calculate max
    grid_voxel_count = source_grid.size_u * source_grid.size_v * source_grid.size_w
    list_count = len(values)
    max_count = min(grid_voxel_count, list_count)
    
    # Step 5 & 6: Iterate and set values
    for i in range(max_count):
        modified_grid[i] = values[i]
        
    # Step 7: Output
    return [modified_grid]
```

**Edge Cases and Pitfalls:**
- **Length Mismatch:** If the input list is shorter than the grid, remaining voxels retain their cloned (original) values. If the list is longer, excess values are ignored.
- **Index Mapping:** This method assumes the input list is already flattened in the same order as the grid's internal memory layout (e.g., row-major or column-major). If the list is spatially ordered (e.g., nested loops), it must be flattened before passing.
- **Type Safety:** The input list must contain boolean-compatible values. Casting errors may occur if floats or integers are passed without explicit conversion.

## Subtopics

### Expanding Voxel Grid
A variant of `VoxelGridFromList` (or the underlying `__setitem__` operation) that automatically expands the grid's bounding box when points are set outside the current range. It preserves existing data by reallocating and copying.
- **Definition:** A `VoxelGrid` variant that supports dynamic resizing.
- **Mechanism:** When setting points `pts` with values:
  1. Calculate `min_pts` and `max_pts` of the new points.
  2. Check if `max_pts` exceeds `range_per_dim[dim][1]` (over) or `min_pts` is below `range_per_dim[dim][0]` (under) for any dimension.
  3. If `over > 0`, expand the upper bound:
     $$ \text{new\_max} = \text{current\_max} + \lceil \frac{\text{over}}{\text{resolution}} \rceil \cdot \text{resolution} $$
  4. If `under > 0`, expand the lower bound:
     $$ \text{new\_min} = \text{current\_min} - \lceil \frac{\text{under}}{\text{resolution}} \rceil \cdot \text{resolution} $$
  5. If the new range differs from the current range (checked via `np.allclose`):
     - Retrieve existing known points and values.
     - Create a new `VoxelGrid` with the expanded range.
     - Insert existing points/values into the new grid.
     - Insert the new `pts` and values.
- **Use Case:** Useful for incremental SDF construction or occupancy mapping where the scene extent is unknown or grows over time.
- **Source:** [S2, unofficial]

## Operating characteristics
- **Time Complexity:** $O(\min(N, M))$, where $N$ is the number of voxels in the grid and $M$ is the number of values in the input list. The cloning step is $O(N)$.
- **Memory Complexity:** $O(N)$ for the cloned grid.
- **Performance:** The operation is memory-bound due to the initial clone. For large grids, this can be expensive. However, the subsequent assignment is a simple linear copy, which is cache-friendly.
- **Strengths:** Simple implementation, no spatial traversal overhead, suitable for bulk updates.
- **Limitations:** Does not handle spatial indexing or sparse updates efficiently. Cloning the entire grid for small updates is wasteful.
- **When to Use:** When initializing a grid from a pre-flattened dataset or when updating a significant portion of the grid.
- **When Not to Use:** For sparse updates or when the grid is very large and memory is constrained. Use [Voxel Grid Construction](3085-voxel-grid-construction.md) or sparse variants instead.

## Background
The `VoxelGridFromList` method is part of the `voxeltools` library, an unofficial toolset for voxel-based operations. It provides a straightforward way to populate voxel grids from linear data structures, facilitating integration with other data formats that store voxel data in flattened arrays. The "Expanding Voxel Grid" variant is associated with `pytorch_volumetric`, which focuses on differentiable volumetric structures for robotics and SDF applications.

## Sources
[S1] voxeltool, https://github.com/arendvw/voxeltools, fetched 2026-10-29 (unofficial)
[S2] GitHub - UM-ARM-Lab/pytorch_volumetric: Volumetric structures such as ..., https://github.com/UM-ARM-Lab/pytorch_volumetric, fetched 2026-10-30 (unofficial)
