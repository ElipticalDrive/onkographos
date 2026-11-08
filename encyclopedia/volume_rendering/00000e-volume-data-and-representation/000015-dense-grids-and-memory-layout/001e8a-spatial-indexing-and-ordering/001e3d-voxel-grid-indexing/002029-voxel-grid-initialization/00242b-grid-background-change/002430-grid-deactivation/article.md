# Grid Deactivation

Also called: deactivate

## Overview
Grid Deactivation removes specific voxels from the active set of a sparse volume grid by filtering out those whose values match a target `value` [S1, unofficial]. It serves as a post-processing step to clean up noise or remove background artifacts that were incorrectly marked as active during initialization [S1, unofficial]. Unlike [Grid Background Change](3210-grid-background-change.md), which updates the *value* of inactive voxels to a new background while preserving active ones, Grid Deactivation removes active voxels entirely if they hold a specific value, effectively shrinking the active volume [S1, unofficial].

## How it works
**Inputs:**
- `grid`: The source sparse volume grid containing an active set of voxels.
- `value`: The scalar value identifying voxels to be deactivated (e.g., the old background value).

**Outputs:**
- `new_grid`: A new sparse volume grid where voxels with `value` are removed from the active set.

**Data Structures:**
- The grid uses a spatial index (e.g., octree or hash map) to store only active voxels. Each active voxel stores its coordinate and value.

**Steps:**
1. Initialize an empty list `data` to hold the surviving voxel entries.
2. Iterate through all active voxels in `grid`.
3. For each active voxel with value `v`:
   - If `v != value`, append the voxel (coordinate and value) to `data`.
   - If `v == value`, discard the voxel (deactivate it).
4. Construct `new_grid` using the filtered `data` and the grid's background value.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def grid_deactivation(grid, value):
    """
    Deactivates active voxels whose value equals the specified target value.
    
    Args:
        grid: Sparse volume grid object with active_voxels() iterator.
        value: Scalar value to filter out.
        
    Returns:
        new_grid: New sparse volume grid with filtered active voxels.
    """
    data = []
    
    # Iterate all active voxels in the source grid
    for voxel in grid.active_voxels():
        v = voxel.value
        # Keep only voxels that do NOT match the target value
        if v != value:
            data.append(voxel)
            
    # Build new grid from filtered data, preserving background
    new_grid = build_grid(data, grid.background)
    return new_grid
```

**Parameters:**
- `value`: The specific scalar value to deactivate. Typical values depend on the background definition (e.g., 0.0 for empty space).

**Edge Cases:**
- If all active voxels match `value`, the resulting grid is empty (no active voxels).
- Precision issues may arise if `value` is a floating-point number; exact equality checks may fail due to rounding errors. Consider using a tolerance if appropriate for the data type [G].

## Subtopics

### Mask Undo
Removes masking bits from voxel values in a Shader Storage Buffer Object (SSBO) using a compute shader subroutine [S2, unofficial]. It specifically clears the `MASK_POSITION` bit from voxel values, effectively undoing a previous masking operation [S2, unofficial]. This is used when the mask information is stored within the voxel value itself rather than in a separate mask grid.

**Definition:** Removes the `MASK_POSITION` bit from voxel values in the SSBO [S2, unofficial].

**Steps:**
1. Bind the SSBO containing the voxel data.
2. Execute the `_undoMaskShader` compute shader with the subroutine `unmaskBit` [S2, unofficial].
3. Update the local grid data from the modified SSBO [S2, unofficial].

**Formula:**
$$ \text{result} = \text{value} \ \& \ \sim(1 \ll \text{MASK\_POSITION}) $$

Where:
- `value`: The original voxel value.
- `MASK_POSITION`: The bit position of the mask flag (e.g., 0, 1, etc.).
- `&`: Bitwise AND.
- `~`: Bitwise NOT.
- `<<`: Bitwise left shift.

**When to choose:** Use Mask Undo when voxel values encode mask status as bits and you need to revert to the raw value without the mask flag [S2, unofficial]. It is distinct from standard Grid Deactivation because it operates on bit-level flags within the value rather than filtering based on the value's magnitude or identity [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the number of active voxels in the source grid, as every active voxel must be visited and checked [S1, unofficial].
- **Memory Complexity:** $O(M)$, where $M$ is the number of surviving active voxels ($M \le N$) [S1, unofficial].
- **Performance:** Performance depends on the efficiency of the active voxel iteration and the grid construction routine. In Lyr.jl, this is implemented in Julia and can run on GPU via CUDA/JIT compilation [S1, unofficial].
- **Strengths:** Simple to implement; effective for removing specific noise or background artifacts from the active set [S1, unofficial].
- **Limitations:** Does not handle spatial coherence; may fragment the active set if the target value is scattered. Precision issues with floating-point values may require tolerance-based comparison [G].

## Background
Grid Deactivation is a utility operation in sparse volume rendering pipelines, often used after volume initialization or masking steps to refine the active voxel set. It is part of the broader family of grid manipulation operations in libraries like Lyr.jl [S1, unofficial]. The concept of deactivating voxels based on value thresholds is common in medical imaging and scientific visualization to isolate specific tissue densities or remove air pockets [S1, unofficial].

## Sources
[S1] Lyr.jl, https://github.com/tobiasosborne/Lyr.jl, fetched 2026-10-26 (unofficial)
[S2] VoxelFragmentML, https://github.com/AlfonsoLRz/VoxelFragmentML, fetched 2026-10-26 (unofficial)
