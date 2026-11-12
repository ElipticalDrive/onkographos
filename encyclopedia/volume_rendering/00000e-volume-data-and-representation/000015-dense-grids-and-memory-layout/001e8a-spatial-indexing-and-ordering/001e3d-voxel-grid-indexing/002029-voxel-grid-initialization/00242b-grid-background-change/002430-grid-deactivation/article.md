# Grid Deactivation

Also called: deactivate

## Overview
Grid Deactivation removes active voxels from a sparse volume grid whose values match a target `value` [S1, unofficial]. Unlike [Grid Background Change](3210-grid-background-change.md), which updates inactive voxels to a new background value while preserving active ones, Grid Deactivation removes active voxels entirely if they hold the target value, shrinking the active set [S1, unofficial].

## How it works
**Inputs:**
- `grid`: The source sparse volume grid containing an active set of voxels.
- `value`: The scalar value identifying voxels to be deactivated.

**Outputs:**
- `new_grid`: A new sparse volume grid where voxels with `value` are removed from the active set.

**Data Structures:**
- The grid stores active voxels in a spatial index [G]. Each active voxel stores its coordinate and value [G].

**Steps:**
1. Initialize an empty list `data` to hold the surviving voxel entries.
2. Iterate through all active voxels in `grid` [S1, unofficial].
3. For each active voxel with value `v`:
   - If `v != value`, append the voxel (coordinate and value) to `data` [S1, unofficial].
   - If `v == value`, discard the voxel (deactivate it) [S1, unofficial].
4. Construct `new_grid` using the filtered `data` and the grid's background value [S1, unofficial].

**Formulas:**
Not given in the sources.

**Parameters:**
- `value`: The specific scalar value to deactivate. Typical values: Not given in the sources.

**Edge Cases and Pitfalls:**
- If all active voxels match `value`, the resulting grid is empty (no active voxels) [S1, unofficial].
- Precision issues may arise if `value` is a floating-point number; exact equality checks may fail due to rounding errors. Consider using a tolerance if appropriate for the data type [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def grid_deactivation(grid, value):
    data = []
    for voxel in grid.active_voxels:
        if voxel.value != value:
            data.append((voxel.coordinate, voxel.value))
    return build_sparse_grid(data, grid.background)

def build_sparse_grid(data, background):
    new_grid = SparseGrid(background)
    for coord, val in data:
        new_grid.set_active(coord, val)
    return new_grid
```

## Subtopics

### Mask Undo
**Definition:** Removes the `MASK_POSITION` bit from voxel values [S2, unofficial].

**Steps:**
1. Execute the `_undoMaskShader` with subroutine `unmaskBit` [S2, unofficial].
2. Update the local grid data from the modified values [S2, unofficial].

**Formula:**
$$ \text{result} = \text{value} \ \& \ \sim(1 \ll \text{MASK\_POSITION}) $$

Where:
- `value`: The original voxel value.
- `MASK_POSITION`: The bit position of the mask flag.
- `&`: Bitwise AND.
- `~`: Bitwise NOT.
- `<<`: Bitwise left shift.

**When to choose:** Use Mask Undo when voxel values encode mask status as bits and you need to revert to the raw value without the mask flag [S2, unofficial]. It operates on bit-level flags within the value rather than filtering based on magnitude [S2, unofficial].

## Operating characteristics
- Time Complexity: Not given in the sources.
- Memory Complexity: Not given in the sources.
- Performance: Not given in the sources.
- Strengths: Not given in the sources.
- Limitations: Not given in the sources.

## Background
Origin and history: Not given in the sources.

## Sources
[S1] Lyr.jl, https://github.com/tobiasosborne/Lyr.jl, fetched 2026-10-26 (unofficial)
[S2] VoxelFragmentML, https://github.com/AlfonsoLRz/VoxelFragmentML, fetched 2026-10-26 (unofficial)
