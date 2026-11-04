# Voxel Grid Initialization

Also called: Grid Activation, Grid Initialization, activate, vTable1 initialization

## Overview
Voxel Grid Initialization returns a new grid where any voxel (active or inactive) whose stored value equals a specific `value` is made active, effectively expanding the active set [S1, unofficial]. It differs from sibling methods like [Coordinate Space Conversion](1785-coordinate-space-conversion.md) by focusing on data population and memory allocation rather than coordinate transformation, and from [Spatial Hashing](5002-spatial-hashing.md) by operating on a dense grid structure before potential sparsification [G].

## How it works
**Inputs:**
- `tree`: The sparse voxel tree structure containing leaf nodes and a `background` value [S1, unofficial].
- `value`: The specific scalar value to match for activation [S1, unofficial].
- `linearGridDim`: The linear dimension of the grid, used to calculate the voxel table size [S2, unofficial].

**Outputs:**
- A new grid structure with an initialized `vTable1` containing active voxel coordinates [S2, unofficial].

**Data Structures:**
- **Grid Structure**: Contains `vTable1` (`unsigned int*`), `gridSize` (`int`), and `gridVoxelSize` (`float`) [S2, unofficial].
- **Voxel Table (`vTable1`)**: A 1D array of unsigned integers. Its dimension is calculated via `vTableDimension(linearGridDim)` [S2, unofficial]. The specific formula for `vTableDimension` is not provided in the sources [S2, unofficial].

**Steps:**
1. Initialize `data` with all currently active voxels from `active_voxels(tree)` [S1, unofficial].
2. Allocate memory for the voxel table (`vTable1`) using `vTableMalloc`, which calculates the dimension based on `linearGridDim` [S2, unofficial].
3. Iterate all leaves. For each leaf, iterate offsets in `off_indices(leaf.value_mask)`. If `leaf.values[offset + 1] == value`, decode coordinate `c` and add `data[c] = value` [S1, unofficial].
4. Initialize the allocated voxel table to all zeros (empty) using `vTableInitialEmpty` [S2, unofficial].
5. Build the new grid using `build_grid(data, tree.background; ...)` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxel_grid_initialization(tree, value, linearGridDim):
    # Step 1: Initialize data with currently active voxels
    data = {}
    for c in tree.active_voxels():
        data[c] = tree.background

    # Step 2: Allocate memory for vTable1 based on linearGridDim
    vTable1 = [0] * (linearGridDim ** 3)  # vTableMalloc implementation [G]

    # Step 3: Iterate all leaves and activate matching voxels
    for leaf in tree.leaves:
        offsets = leaf.value_mask.get_set_bits()  # off_indices implementation [G]
        for offset in offsets:
            # Source uses offset + 1 indexing [S1, unofficial]
            if offset + 1 < len(leaf.values) and leaf.values[offset + 1] == value:
                c = leaf.decode_coordinate(offset)  # decode_coordinate implementation [G]
                data[c] = value

    # Step 4: Initialize the allocated voxel table to all zeros
    vTable1 = [0] * len(vTable1)  # vTableInitialEmpty implementation [G]

    # Step 5: Build the new grid
    grid = Grid()
    grid.vTable1 = vTable1
    grid.gridSize = linearGridDim
    grid.gridVoxelSize = tree.voxel_size
    # build_grid(data, tree.background) populates grid from data [S1, unofficial]

    return grid

# Helper definitions (standard operations not fully detailed in sources)
class Grid:
    def __init__(self):
        self.vTable1 = None
        self.gridSize = 0
        self.gridVoxelSize = 0.0
```

**Parameters:**
- `value`: The target scalar value to match for activation [S1, unofficial].
- `linearGridDim`: The linear dimension of the grid, used for memory allocation [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Precision**: Floating-point inaccuracies in coordinate decoding may lead to duplicate or missing voxels if not handled by integer quantization (see [Voxel Grid Construction](3085-voxel-grid-construction.md)) [G].
- **Boundaries**: Ensure `offset + 1` does not exceed the bounds of `leaf.values` [G].
- **Degenerate Input**: If `tree` has no leaves, the grid will be empty but still allocated [G].

## Subtopics

### [Voxel Grid Construction](3085-voxel-grid-construction.md)
Constructs a 3D voxel grid where each cube has 8 corner vertices, deduplicating shared corners to store them only once. The grid is centered at the origin with unit length along each dimension. It differs from naive grid generation by using integer quantization for vertex deduplication to handle floating-point inaccuracies.

### [Grid Background Change](3210-grid-background-change.md)
Updates the background value of inactive voxels in a volume grid while preserving the values of active voxels. It returns a new grid where inactive voxels holding the old background value are updated to a new background value, while active voxels remain unchanged.

### Dense Copy Ingestion
Builds a VDB grid from a dense 3D array. Only values differing from `background` are stored as active voxels [S1, unofficial].
- Iterate array indices `ix, iy, iz`.
- If `array[ix, iy, iz] != background`, compute world coordinate `c` using `bbox_min` and add to `data` [S1, unofficial].
- Build grid using `build_grid(data, background; ...)` [S1, unofficial].

## Operating characteristics
Not given in the sources.

## Background
The method originates from the Lyr.jl library [S1, unofficial] and is used in the context of GPU-accelerated volume rendering. It is part of the pipeline that converts level set data into fog for volume rendering [S1, unofficial]. The voxel table initialization is a critical step in preparing the data structure for the rendering kernel [S2, unofficial].

## Sources
[S1] Lyr.jl, https://github.com/tobiasosborne/Lyr.jl, fetched 2026-10-26 (unofficial)
[S2] gpu-voxelizer, https://github.com/ivandrodri/gpu-voxelizer, fetched 2026-10-26 (unofficial)
