# Voxel Space Setup

Also called: AABB to Voxel Grid Mapping, SetupVoxelization

## Overview
Voxel Space Setup computes the bounding box and transformation matrices required to map continuous world coordinates to discrete voxel grid indices, handling padding and cube voxel normalization [S1, unofficial]. It establishes the geometric foundation for subsequent coordinate conversions and ray traversals by defining the active volume extent in voxel space [S1, unofficial]. It differs from [Coordinate Conversion Utilities](1785-coordinate-space-conversion.md) by performing the static initialization of the bounding volume and matrices, whereas utilities perform the per-point arithmetic [S1, unofficial].

## How it works
**Inputs:**
- Axis-Aligned Bounding Box (AABB) in world space: `min_world` and `max_world` (3D vectors).
- Target grid resolution: `gridSize` (3D integer vector, typically $N_x \times N_y \times N_z$).
- Boolean flag `g_useCubeVoxels`: if true, enforces uniform voxel dimensions across all axes [S1, unofficial].

**Outputs:**
- `voxelSpaceMin` and `voxelSpaceMax`: The bounding box in voxel space (3D vectors) [S1, unofficial].
- `g_matWorldToVoxel`: Transformation matrix mapping world coordinates to voxel indices [S1, unofficial].
- `g_matWorldToVoxelProj`: Transformation matrix mapping world coordinates to Normalized Device Coordinates (NDC) for projection [S1, unofficial].
- Stride values for memory layout: `strideX`, `strideY`, `dataSize` [S1, unofficial].

**Data Structures:**
- `g_voxelSpace[2]`: Array of two 3D vectors storing `voxelSpaceMin` and `voxelSpaceMax` [S1, unofficial].
- `g_matWorldToVoxel`: 4x4 matrix (column-major or row-major depending on implementation, typically stored as 16 floats) [S1, unofficial].
- `g_matWorldToVoxelProj`: 4x4 matrix [S1, unofficial].

**Steps:**
1. **Calculate Memory Strides:** Compute the stride for the Z-axis padded to 32-bit words, then the Y-axis stride, and total data size [S1, unofficial].
   $$ \text{strideX} = \lceil \frac{N_z + 31}{32} \rceil $$
   $$ \text{strideY} = \text{strideX} \times N_x $$
   $$ \text{dataSize} = \text{strideY} \times N_y $$
   [S1, unofficial]

2. **Compute Model Extent:** Calculate the physical extent of the AABB in world space [S1, unofficial].
   $$ \text{extent}_{world} = \text{max\_world} - \text{min\_world} $$
   [S1, unofficial]

3. **Apply Padding:** Scale the extent by a factor that adds 2 units of padding to the grid dimensions [S1, unofficial].
   $$ \text{extent}_{padded} = \text{extent}_{world} \times \frac{N + 2}{N} $$
   where $N$ is the component-wise grid size vector $(N_x, N_y, N_z)$ [S1, unofficial].
   *Pitfall:* This padding prevents edge cases where triangles exactly on the boundary are missed [S1, unofficial].

4. **Normalize to Cube Voxels (Optional):** If `g_useCubeVoxels` is true, make the extent uniform by taking the maximum component of the padded extent [S1, unofficial].
   $$ \text{extent}_{final} = (\max(\text{extent}_{padded}), \max(\text{extent}_{padded}), \max(\text{extent}_{padded})) $$
   [S1, unofficial]

5. **Compute Center and Voxel Space Bounds:** Calculate the center of the original AABB and derive the voxel space minimum [S1, unofficial].
   $$ \text{center} = \frac{\text{min\_world} + \text{max\_world}}{2} $$
   $$ \text{voxelSpaceMin} = \text{center} - 0.5 \times \text{extent}_{final} $$
   $$ \text{voxelSpaceMax} = \text{voxelSpaceMin} + \text{extent}_{final} $$
   [S1, unofficial]

6. **Compute World-to-Voxel Matrix:** Construct the matrix that translates the voxel space minimum to the origin and scales the extent to the grid size [S1, unofficial].
   $$ \text{g\_matWorldToVoxel} = \text{Translate}(-\text{voxelSpaceMin}) \times \text{Scale}\left(\frac{N}{\text{extent}_{final}}\right) $$
   [S1, unofficial]

7. **Compute World-to-Projection Matrix:** Construct the matrix for NDC projection, translating the center to the origin (with Z offset) and scaling to NDC range [-1, 1] for X/Y and [0, 1] for Z (or similar, depending on API) [S1, unofficial].
   $$ \text{g\_matWorldToVoxelProj} = \text{Translate}(-\text{center}.x, -\text{center}.y, -\text{voxelSpaceMin}.z) \times \text{Scale}\left(\frac{2.0}{\text{extent}_{final}.x}, \frac{2.0}{\text{extent}_{final}.y}, \frac{1.0}{\text{extent}_{final}.z}, 0.0\right) $$
   [S1, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def setup_voxelization(aabb_min, aabb_max, grid_size, use_cube_voxels):
    """
    Computes voxel space bounding box and transformation matrices.
    
    Args:
        aabb_min: (float, float, float) World space minimum of AABB.
        aabb_max: (float, float, float) World space maximum of AABB.
        grid_size: (int, int, int) Target resolution (Nx, Ny, Nz).
        use_cube_voxels: bool. If True, voxels are cubic.
        
    Returns:
        voxel_space_min: (float, float, float)
        voxel_space_max: (float, float, float)
        mat_world_to_voxel: 4x4 matrix
        mat_world_to_voxel_proj: 4x4 matrix
        stride_x: int
        stride_y: int
        data_size: int
    """
    # 1. Calculate Memory Strides
    # Pad Z stride to 32-bit words (assuming 1 bit per voxel or similar compact layout)
    nz = grid_size[2]
    stride_x = (nz + 31) // 32
    nx = grid_size[0]
    ny = grid_size[1]
    stride_y = stride_x * nx
    data_size = stride_y * ny
    
    # 2. Compute Model Extent
    extent_world = [aabb_max[i] - aabb_min[i] for i in range(3)]
    
    # 3. Apply Padding
    # Scale extent by (gridSize + 2) / gridSize
    extent_padded = [0.0, 0.0, 0.0]
    for i in range(3):
        gs = float(grid_size[i])
        extent_padded[i] = extent_world[i] * (gs + 2.0) / gs
        
    # 4. Normalize to Cube Voxels
    if use_cube_voxels:
        max_extent = max(extent_padded)
        extent_final = [max_extent, max_extent, max_extent]
    else:
        extent_final = extent_padded
        
    # 5. Compute Center and Voxel Space Bounds
    center = [(aabb_min[i] + aabb_max[i]) / 2.0 for i in range(3)]
    voxel_space_min = [center[i] - 0.5 * extent_final[i] for i in range(3)]
    voxel_space_max = [voxel_space_min[i] + extent_final[i] for i in range(3)]
    
    # 6. Compute World-to-Voxel Matrix
    # Translate(-voxelSpaceMin) * Scale(gridSize / extent)
    # Note: Matrix multiplication order depends on convention (row vs column vectors).
    # Assuming column vectors: M = Scale * Translate
    # However, source code shows XMMatrixMultiplyTranspose(Translate, Scale), 
    # which implies row-vector convention or specific HLSL/DirectX matrix layout.
    # We will define the transformation logic explicitly.
    
    # Translation component
    t_x = -voxel_space_min[0]
    t_y = -voxel_space_min[1]
    t_z = -voxel_space_min[2]
    
    # Scale component
    s_x = grid_size[0] / extent_final[0]
    s_y = grid_size[1] / extent_final[1]
    s_z = grid_size[2] / extent_final[2]
    
    # Construct 4x4 Matrix (Row-Major for simplicity, adjust if Column-Major required)
    # [ s_x  0    0    0 ]
    # [ 0    s_y  0    0 ]
    # [ 0    0    s_z  0 ]
    # [ t_x  t_y  t_z  1 ]
    mat_world_to_voxel = [
        [s_x, 0.0, 0.0, 0.0],
        [0.0, s_y, 0.0, 0.0],
        [0.0, 0.0, s_z, 0.0],
        [t_x, t_y, t_z, 1.0]
    ]
    
    # 7. Compute World-to-Projection Matrix
    # Translate(-center.x, -center.y, -voxelSpaceMin.z) * Scale(2.0/extent.x, 2.0/extent.y, 1.0/extent.z, 0.0)
    t_proj_x = -center[0]
    t_proj_y = -center[1]
    t_proj_z = -voxel_space_min[2]
    
    s_proj_x = 2.0 / extent_final[0]
    s_proj_y = 2.0 / extent_final[1]
    s_proj_z = 1.0 / extent_final[2]
    
    mat_world_to_voxel_proj = [
        [s_proj_x, 0.0,      0.0,      0.0],
        [0.0,      s_proj_y, 0.0,      0.0],
        [0.0,      0.0,      s_proj_z, 0.0],
        [t_proj_x, t_proj_y, t_proj_z, 1.0]
    ]
    
    return voxel_space_min, voxel_space_max, mat_world_to_voxel, mat_world_to_voxel_proj, stride_x, stride_y, data_size
```

**Parameters:**
- `g_useCubeVoxels`: Boolean flag to enforce uniform voxel dimensions across all axes [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Boundary Misses:** Grid padding of 2 units is added to the extent to prevent edge cases where triangles exactly on the boundary are missed [S1, unofficial].
- **Precision:** Ensure floating-point precision is sufficient when computing `voxelSpaceMin` and matrices to avoid artifacts in ray traversal [G].

## Subtopics

### [Spherical Voxel Grid Setup](3883-spherical-voxel-grid-setup.md)
Spherical Voxel Grid Setup initializes the data structures required for efficient ray casting traversal in a spherical coordinate system, specifically designed for egocentric omnidirectional view synthesis. It differs from Cartesian voxel grids by aligning the grid topology with the radial nature of egocentric rays, thereby reducing wasted memory on distant objects and avoiding the non-uniform ray-grid hit distribution inherent in uniform Cartesian resolutions. It builds on [Voxel Space Setup](3520-voxel-space-setup.md) by defining a spherical bounding volume and precomputing geometric constants for traversal.

### [Incremental Back-Rotation for Voxel Mapping](6759-incremental-back-rotation-for-voxel-mapping.md)
Incremental back-rotation is an algorithm used to map destination voxel coordinates to their corresponding source voxel positions during a room rotation operation, specifically to eradicate holes produced by forward rotation. It sets itself apart from forward mapping by iterating over the destination grid and computing source locations, ensuring every destination voxel is filled. This method builds on [Voxel Space Setup](3520-voxel-space-setup.md) by utilizing the established transformation matrices to perform efficient, incremental coordinate calculations.

### [Coordinate Conversion Utilities](6766-coordinate-conversion-utilities.md)
Coordinate Conversion Utilities provide the bidirectional mapping between continuous world-space positions and discrete voxel grid indices, serving as the fundamental interface between geometric reasoning and volumetric data access. While [Voxel Space Setup](3520-voxel-space-setup.md) establishes the bounding box and transformation matrices, these utilities perform the actual arithmetic to translate a point in space into a specific cell index and vice versa. They differ from [Spherical Voxel Grid Setup](3883-spherical-voxel-grid-setup.md) by operating on Cartesian grids rather than spherical shells, and from [Incremental Back-Rotation for Voxel Mapping](6759-incremental-back-rotation-for-voxel-mapping.md) by handling static linear transformations rather than iterative rotational adjustments.

### [PixelGrid2D Construction](9430-pixelgrid2d-construction.md)
Constructs a 2D grid structure (`Grid2D<BitArray, bool>`) that maps voxel coordinates to boolean occupancy flags, initialized from a bounding box and a pre-computed bit array. It differs from [Incremental Back-Rotation for Voxel Mapping](6759-incremental-back-rotation-for-voxel-mapping.md) by assuming a static, pre-calculated set of active voxels provided as a flat bit stream, rather than computing occupancy dynamically during ray traversal. It serves as the foundational data structure for [Voxel Center Precomputation](10731-voxel-center-precomputation.md) by providing the spatial index for active cells.

### [Voxel Center Precomputation](10731-voxel-center-precomputation.md)
Precomputes the 3D world coordinates of voxel centers for a regular, axis-aligned voxel grid defined by an origin, dimensions, and physical voxel size. It solves the problem of mapping discrete grid indices to continuous world space coordinates efficiently, avoiding per-lookup computation during rendering or training. It differs from [Incremental Back-Rotation for Voxel Mapping](6759-incremental-back-rotation-for-voxel-mapping.md) by assuming a fixed, non-rotated grid, and from [Coordinate Conversion Utilities](6766-coordinate-conversion-utilities.md) by providing the full 3D tensor of centers rather than generic conversion functions.

### [Voxel Grid Construction from AABB](12627-voxel-grid-construction-from-aabb.md)
Constructs a uniform voxel grid from an Axis-Aligned Bounding Box (AABB) and a target resolution. It ensures uniform voxel size by scaling based on the longest axis and clamping zero-extent axes.

### OpenVDB Density Grid Loading
Loads a heterogeneous density field from an OpenVDB file to provide spatial density queries for participating media [S2, unofficial]. It computes the AABB from the active voxel bounding box using `gridPtr->evalActiveVoxelBoundingBox()` and converts indices to world coordinates [S2, unofficial]. Density at a given world position is interpolated using `openvdb::tools::GridSampler` with `openvdb::tools::BoxSampler` [S2, unofficial]. The maximum density is retrieved via `gridPtr->evalMinMax(minValue, maxValue)` [S2, unofficial].

### Ray Tracing Volume Data Setup
Typical simplifications include an integral mapping of voxel address space to pixel address space (1-1 is easiest; usually projection of a voxel map to $k \times k$ pixels); arbitrary mapping requires interpolation [S3, unofficial]. The sampling lattice is rectilinear and uniformly spaced [S3, unofficial]. Pixel spacing must be less than voxel spacing [S3, unofficial]. The setup typically uses one ray per pixel (no supersampling) and parallel projection [S3, unofficial].

### Volume Normalization to Unit Cube
Computes the bounding box (bb_min, bb_max) of the volume grid by transforming the grid's corner points (0,0,0) and (index_extent, index_extent, index_extent) using the grid's transform matrix [S4, unofficial]. It calculates the extent vector (bb_max - bb_min) and the maximum dimension size [S4, unofficial]. If size is not 1.0, it constructs a transformation matrix that scales the volume by 1/size and translates it so its center is at the origin (subtracting bb_min + 0.5 * extent) [S4, unofficial]. It multiplies the density_scale parameter by the original size to compensate for the spatial scaling [S4, unofficial]. Initialization sets `bb_min` to `glm::vec3(FLT_MAX)` and `bb_max` to `glm::vec3(FLT_MIN)` [S4, unofficial]. It iterates over all frames in `volume->grids`, updating bounds with `grid->transform * glm::vec4(0, 0, 0, 1)` and `grid->transform * glm::vec4(glm::vec3(grid->index_extent()), 1)` [S4, unofficial].

## Operating characteristics
- **Time Complexity:** The setup process is $O(1)$ with respect to the number of voxels, as it involves a fixed number of arithmetic operations on the bounding box and grid dimensions [S1, unofficial].
- **Memory Complexity:** The memory required for the transformation matrices is constant (2x 4x4 matrices) [S1, unofficial]. The memory for the voxel grid itself is determined by the grid dimensions and padding [S1, unofficial].
- **Strengths:** Provides a robust mapping from world space to voxel space, handling padding and cube normalization to ensure consistent traversal [S1, unofficial].
- **Limitations:** The setup assumes a static AABB. Dynamic changes to the scene geometry require re-computation of the setup [G].
- **When to Use:** Use when initializing a voxel-based rendering pipeline that requires mapping world-space geometry to a discrete grid [S1, unofficial].
- **When Not to Use:** Do not use if the voxel grid is dynamic or if non-Cartesian coordinate systems are required (use [Spherical Voxel Grid Setup](3883-spherical-voxel-grid-setup.md) instead) [S1, unofficial].

## Background
Voxel Space Setup is a fundamental step in GPU-based voxelization and volume rendering pipelines. It originates from techniques described in "Voxelization via Rendering" and related GPU Pro articles, where efficient mapping of world-space primitives to voxel grids is critical for subsequent rasterization or ray-tracing steps [S1, unofficial]. The method is widely used in real-time rendering engines and offline volume renderers that rely on dense voxel representations [S1, unofficial].

## Sources
[S1] michschw/GpuPro3Vox, https://github.com/michschw/GpuPro3Vox, fetched 2026-10-26 (unofficial)
[S2] GitHub - yumcyaWiz/volpt: Volumetric path tracer written in C++, https://github.com/yumcyaWiz/volpt, fetched 2026-10-27 (unofficial)
[S3] PDF Direct Volume Rendering, https://www.cs.unh.edu/~cs880/lectureNotes/DVR.pdf, fetched 2026-10-28 (unofficial)
[S4] nihofm/volren: Simple OpenGL-based renderer for ..., https://github.com/nihofm/volren, fetched 2026-10-29 (unofficial)
