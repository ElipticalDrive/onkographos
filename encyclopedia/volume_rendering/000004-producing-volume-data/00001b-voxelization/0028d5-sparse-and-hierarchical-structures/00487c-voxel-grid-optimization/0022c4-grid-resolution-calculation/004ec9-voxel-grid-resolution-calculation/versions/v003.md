# Voxel Grid Resolution Calculation

Also called: Sample Point Voxel Count Calculation

## Overview
This method calculates the discrete resolution of a 3D voxel grid based on a target voxel count or physical dimensions, ensuring the grid fits a specified bounding box. It serves as the foundational step for voxelization in sparse and hierarchical structures, determining the granularity of the spatial representation. Unlike [Grid Resolution Selection via Lambda](3270-grid-resolution-selection-via-lambda.md), which optimizes resolution via a lambda parameter for neural fields, this method relies on direct geometric division or occupancy counting heuristics.

## How it works
The method comprises two primary variants: a direct geometric calculation for uniform grids and a GPU-accelerated occupancy counting method for adaptive or subdivided grids.

### Inputs and Outputs
**Inputs:**
- `boundaryLength`, `boundaryWidth`, `boundaryHeight`: Physical dimensions of the scene bounding box along X, Y, and Z axes [S1, unofficial].
- `voxelSize`: The target size of a single voxel [S1, unofficial].
- `num_voxels`: Target total number of voxels (used in scaling variants) [S3, unofficial].
- `subdivisions`: Number of subdivisions per axis for quadrant partitioning [S2, unofficial].

**Outputs:**
- `totalVoxel`: The calculated integer count of voxels [S1, unofficial].
- `gridDims`: The dimensions of the voxel grid in cells [S2, unofficial].
- `voxel_size`, `world_size`: Updated grid parameters [S3, unofficial].

### Data Structures
- **Grid Dimensions**: A 3D vector or tuple `(dim_x, dim_y, dim_z)` representing the number of voxels along each axis [S2, unofficial].
- **Count Buffer (`_countSSBO`)**: A Shader Storage Buffer Object (SSBO) storing occupancy counts per quadrant, initialized to zero [S2, unofficial].
- **Grid Data**: Includes `density`, `seg_mask_grid`, `dual_seg_mask_grid`, and `k0` tensors that are rescaled during the process [S3, unofficial].

### Steps

#### Variant 1: Direct Geometric Calculation
1. Calculate the number of voxels along each axis by dividing the boundary dimension by the voxel size and taking the integer part:
   $$ N_x = \lfloor \frac{\text{boundaryLength}}{\text{voxelSize}} \rfloor $$
   $$ N_y = \lfloor \frac{\text{boundaryWidth}}{\text{voxelSize}} \rfloor $$
   $$ N_z = \lfloor \frac{\text{boundaryHeight}}{\text{voxelSize}} \rfloor $$
   [S1, unofficial]
2. Compute the total voxel count:
   $$ \text{totalVoxel} = N_x \times N_y \times N_z $$
   [S1, unofficial]

#### Variant 2: Quadrant Occupancy Counting (GPU)
1. Calculate grid dimensions (`gridDims`), total cells, and the number of compute groups based on `subdivisions` [S2, unofficial].
2. Compute the step vector for the compute shader:
   $$ \text{step} = \lceil \frac{\text{gridDims}}{\text{subdivisions}} \rceil $$
   [S2, unofficial]
3. Reset the count buffer `_countSSBO` to zero for all entries [S2, unofficial].
4. Bind the scene SSBO `_ssbo` and the count buffer `_countSSBO`. Set uniforms `gridDims` and `numCells` [S2, unofficial].
5. Execute the `_countQuadrantOccupancyShader` with the computed number of groups and maximum group size [S2, unofficial].
6. Read back the data from `_countSSBO` and find the maximum value across all `subdivisions^3` entries [S2, unofficial].

#### Variant 3: Scale Volume Grid (Neural Implicit)
1. Store the original world size:
   $$ \text{ori\_world\_size} = \text{world\_size} $$
   [S3, unofficial]
2. Call `_set_grid_resolution(num_voxels)` to update `voxel_size`, `world_size`, and `voxel_size_ratio` [S3, unofficial].
3. Scale the volume grids (`density`, `seg_mask_grid`, `dual_seg_mask_grid`, `k0`) using the new `world_size` [S3, unofficial].
4. Generate a 3D coordinate grid:
   $$ \text{self\_grid\_xyz} = \text{torch.meshgrid}(\dots) $$
   [S3, unofficial]
5. Calculate activation alpha using max pooling on the density grid:
   $$ \text{self\_alpha} = \text{F.max\_pool3d}(\text{activate\_density}(\text{density.get\_dense\_grid()}), \text{kernel\_size}=3, \text{padding}=1, \text{stride}=1)[0,0] $$
   [S3, unofficial]
6. Update the mask cache:
   $$ \text{mask\_cache} = \text{old\_mask} \ \& \ (\text{self\_alpha} > \text{fast\_color\_thres}) $$
   [S3, unofficial]

### Parameters
- `voxelSize`: Physical size of a voxel. Typical values depend on scene scale [S1, unofficial].
- `num_voxels`: Target voxel count. Used to derive `voxel_size` [S3, unofficial].
- `subdivisions`: Unsigned integer for quadrant partitioning. Common values are powers of 2 (e.g., 2, 4, 8) [S2, unofficial].
- `fast_color_thres`: Threshold for alpha activation. Not given in the sources.

### Edge Cases and Pitfalls
- **Precision**: Integer division in Variant 1 may lead to slight undersampling if `boundarySize` is not an exact multiple of `voxelSize` [S1, unofficial].
- **Small Grids**: If the total voxels (product of `world_size`) is less than or equal to $256^3$, the mask cache update logic in Variant 3 may require specific handling or validation [S3, unofficial].
- **GPU Memory**: Variant 2 requires sufficient VRAM for `_ssbo` and `_countSSBO` [S2, unofficial].

## Subtopics

### Quadrant Occupancy Counting
A GPU-based method to determine the maximum number of occupied voxels per quadrant in a subdivided grid. It differs from direct calculation by providing an occupancy heuristic useful for adaptive refinement. Choose this when analyzing spatial distribution for hierarchical structures. [S2, unofficial]

### Scale Volume Grid
A method for dynamically resizing neural implicit volume grids (e.g., in 3D Gaussian Splatting or NeRFs) based on a target voxel count. It updates density, segmentation masks, and coordinate grids. Choose this when training or refining neural radiance fields that require resolution scaling. [S3, unofficial]

## Operating characteristics
- **Time Complexity**:
  - Variant 1: $O(1)$ arithmetic operations [S1, unofficial].
  - Variant 2: $O(N)$ where $N$ is the number of voxels, dominated by GPU kernel execution [S2, unofficial].
  - Variant 3: $O(N)$ for grid rescaling and pooling operations [S3, unofficial].
- **Memory Complexity**:
  - Variant 1: $O(1)$ [S1, unofficial].
  - Variant 2: $O(N)$ for SSBOs [S2, unofficial].
  - Variant 3: $O(N)$ for storing grid tensors [S3, unofficial].
- **Strengths**: Simple to implement (Variant 1); GPU-accelerated for large scenes (Variant 2); Integrates with neural training pipelines (Variant 3).
- **Limitations**: Variant 1 assumes uniform grids; Variant 2 requires GPU infrastructure; Variant 3 is specific to neural implicit representations.

## Background
The concept of voxel grid resolution calculation is fundamental to volume rendering and voxelization. The direct geometric method is a standard technique in computer graphics for mapping continuous space to discrete grids [S1, unofficial]. The quadrant occupancy method appears in modern voxel-based machine learning frameworks for efficient spatial analysis [S2, unofficial]. The scale volume grid method is associated with recent advances in 3D segmentation and neural radiance fields, specifically in projects like SA3D (Segment Anything in 3D) [S3, unofficial].

## Sources
[S1] sachPico/lineBasedCave_Simplex, https://github.com/sachPico/lineBasedCave_Simplex, fetched 2026-10-27 (unofficial)
[S2] AlfonsoLRz/VoxelFragmentML, https://github.com/AlfonsoLRz/VoxelFragmentML, fetched 2026-10-26 (unofficial)
[S3] Jumpat/SegmentAnythingin3D, https://github.com/Jumpat/SegmentAnythingin3D, fetched 2026-10-26 (unofficial)
