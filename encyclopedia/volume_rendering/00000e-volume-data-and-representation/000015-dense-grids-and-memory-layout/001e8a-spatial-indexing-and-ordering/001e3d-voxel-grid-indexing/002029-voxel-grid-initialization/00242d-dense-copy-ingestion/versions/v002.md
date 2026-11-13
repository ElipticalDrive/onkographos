# Dense Copy Ingestion
## Overview
Dense Copy Ingestion constructs a VDB (OpenVDB) grid from a dense 3D array, storing only values that differ from a specified background value as active voxels [S1, unofficial]. It differs from [Voxel Grid Construction](3085-voxel-grid-construction.md) by accepting pre-existing dense memory layouts rather than generating voxels from geometric primitives, and from [VoxelGridFromList](7224-voxelgridfromlist.md) by operating on contiguous arrays rather than sparse coordinate lists.
## How it works
**Inputs:**
- `array`: A dense 3D array of values (e.g., `Float32`).
- `background`: The scalar value representing empty space.
- `bbox_min`: The world-space minimum coordinate of the grid's bounding box.
- `voxel_size`: The physical size of a single voxel.

**Outputs:**
- A VDB grid object containing active voxels.

**Data Structures:**
- The source `array` is indexed by integer coordinates `(ix, iy, iz)`.
- The target VDB grid stores only non-background values.

**Steps:**
1. Initialize an empty list `data` to hold voxel entries.
2. Iterate through all indices `ix, iy, iz` of the input `array`.
3. For each index, retrieve the value `v = array[ix, iy, iz]`.
4. If `v != background`:
   a. Compute the world-space center coordinate `c` of the voxel. Assuming a regular grid aligned with axes, the coordinate is calculated as:
      $$ c = \text{bbox\_min} + (ix + 0.5) \cdot \text{voxel\_size} $$
      (Note: The exact component-wise addition depends on the axis alignment; typically `ix` maps to X, `iy` to Y, `iz` to Z).
   b. Append `(c, v)` to `data`.
5. Construct the VDB grid using `build_grid(data, background; ...)`. This function allocates the sparse tree structure and inserts the active voxels [S1, unofficial].

**Parameters:**
- `background`: Must match the type of `array`. Typical value: `0.0`.
- `voxel_size`: Physical dimension per voxel.

**Edge Cases:**
- If all values equal `background`, the resulting grid has no active voxels.
- Precision issues may arise if `v` is very close to `background` but not exactly equal due to floating-point arithmetic. A tolerance check might be required in practice, though the source specifies exact inequality [S1, unofficial].

## Subtopics
## Operating characteristics
**Time Complexity:** $O(N)$ where $N$ is the total number of voxels in the dense array, as every element must be checked.
**Memory Complexity:** $O(A)$ where $A$ is the number of active voxels (those differing from background). The dense array is read but not necessarily stored in the final grid.
**Performance:** The method is efficient for sparse data within a dense container, as it avoids storing empty space. However, it requires reading the entire dense array, which can be memory-intensive if the dense array is large and sparse.
**Strengths:** Simple implementation; leverages existing dense data structures.
**Limitations:** Inefficient for highly sparse data if the dense array is very large, as it still iterates over all empty voxels.

## Background
This method is part of the Lyr.jl volume rendering library, specifically wired into its preview kernel for building scene data from dense inputs [S1, unofficial]. It serves as a bridge between dense computational outputs and sparse VDB-based rendering pipelines.

## Sources
[S1] tobiasosborne/Lyr.jl, https://github.com/tobiasosborne/Lyr.jl, fetched 2026-10-26 (unofficial)
