# Voxel Grid Resolution Calculation

Also called: Sample Point Voxel Count Calculation

## Overview
Calculates the integer dimensions of a 3D voxel grid based on a specified voxel size and the bounding box of the scene geometry. It serves as the foundational step for determining grid capacity before optimization or occupancy counting. Unlike [Grid Resolution Selection via Lambda](3270-grid-resolution-selection-via-lambda.md), which uses a lambda function to balance memory and performance, this method relies on direct geometric division.

## How it works
**Inputs:**
- `boundaryLength`: Width of the bounding box along the X-axis.
- `boundaryWidth`: Depth of the bounding box along the Y-axis.
- `boundaryHeight`: Height of the bounding box along the Z-axis.
- `voxelSize`: The edge length of a single voxel.

**Outputs:**
- `totalVoxel`: The total number of voxels in the grid.
- `gridDims`: The integer dimensions $(D_x, D_y, D_z)$ of the grid.

**Data Structures:**
- `gridDims`: A 3D integer vector or array storing the number of voxels per axis.
- `totalVoxel`: A single integer representing the product of `gridDims`.

**Steps:**
1. Calculate the number of voxels along each axis by performing integer division of the boundary dimensions by the voxel size.
   $$ D_x = \lfloor \frac{\text{boundaryLength}}{\text{voxelSize}} \rfloor $$
   $$ D_y = \lfloor \frac{\text{boundaryWidth}}{\text{voxelSize}} \rfloor $$
   $$ D_z = \lfloor \frac{\text{boundaryHeight}}{\text{voxelSize}} \rfloor $$
2. Compute the total voxel count as the product of the dimensions.
   $$ \text{totalVoxel} = D_x \times D_y \times D_z $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def calculate_voxel_grid_resolution(boundary_length, boundary_width, boundary_height, voxel_size):
    # Integer division to determine grid dimensions
    # Note: Implementation depends on language; here we assume floor division
    dim_x = int(boundary_length // voxel_size)
    dim_y = int(boundary_width // voxel_size)
    dim_z = int(boundary_height // voxel_size)
    
    # Calculate total number of voxels
    total_voxel = dim_x * dim_y * dim_z
    
    return {
        "grid_dims": (dim_x, dim_y, dim_z),
        "total_voxel": total_voxel
    }
```

**Parameters:**
- `voxelSize`: Determines the granularity. Smaller values increase resolution and memory usage.

**Edge Cases:**
- If `voxelSize` is zero or negative, the calculation is undefined.
- If boundary dimensions are smaller than `voxelSize`, the resulting dimension is 0, leading to a `totalVoxel` of 0.

## Subtopics

### Quadrant Occupancy Counting
Counts the maximum number of occupied voxels per quadrant in a subdivided grid using a GPU compute shader. It partitions the grid into `subdivisions` parts per axis.
**Steps:**
1. Calculate grid dimensions, total cells, and number of compute groups. Compute step vector: `step = ceil(gridDims / subdivisions)`.
2. Reset count buffer: `updateReadBufferSubset(_countSSBO, &zero, 0, 1)` with zero.
3. Bind `_ssbo` and `_countSSBO`, set uniforms `gridDims` and `numCells`, execute `_countQuadrantOccupancyShader` with computed groups and max group size.
4. Read back count data from `_countSSBO` and find maximum value across `subdivisions^3` entries.
**Parameters:**
- `subdivisions`: Unsigned integer defining the number of subdivisions per axis for quadrant partitioning.

## Operating characteristics
- **Time Complexity:** $O(1)$ for the basic calculation. The Quadrant Occupancy variant is $O(N)$ where $N$ is the number of voxels, executed on GPU.
- **Memory Complexity:** $O(1)$ for the basic calculation. The Quadrant Occupancy variant requires $O(\text{subdivisions}^3)$ for the count buffer.
- **Strengths:** Simple, deterministic, and fast for initial setup.
- **Limitations:** Does not account for actual geometry occupancy, only bounding box volume. May overestimate memory needs if the scene is sparse.

## Background
Originated from implementations in voxel-based rendering systems where grid dimensions must be derived from scene bounds. The basic calculation is found in [sachPico/lineBasedCave_Simplex](https://github.com/sachPico/lineBasedCave_Simplex) [S1, unofficial]. The Quadrant Occupancy variant is described in [AlfonsoLRz/VoxelFragmentML](https://github.com/AlfonsoLRz/VoxelFragmentML) [S2, unofficial].

## Sources
[S1] sachPico/lineBasedCave_Simplex, https://github.com/sachPico/lineBasedCave_Simplex, fetched 2026-10-27 (unofficial)
[S2] AlfonsoLRz/VoxelFragmentML, https://github.com/AlfonsoLRz/VoxelFragmentML, fetched 2026-10-26 (unofficial)
