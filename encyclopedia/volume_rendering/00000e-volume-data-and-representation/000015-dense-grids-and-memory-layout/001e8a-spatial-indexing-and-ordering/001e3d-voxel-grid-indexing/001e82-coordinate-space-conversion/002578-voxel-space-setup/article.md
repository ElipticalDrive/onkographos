# Voxel Space Setup

Also called: AABB to Voxel Grid Mapping, SetupVoxelization

## Overview
Voxel Space Setup computes the bounding box and transformation matrices required to map world coordinates to voxel grid indices, including padding and cube voxel normalization [S1, unofficial]. It solves the problem of establishing a consistent coordinate system where floating-point world positions are discretized into integer voxel indices for direct memory access [Parent](1785-coordinate-space-conversion.md). It differs from its siblings by focusing on the initial construction of the Cartesian voxel grid from an Axis-Aligned Bounding Box (AABB), whereas [Spherical Voxel Grid Setup](3883-spherical-voxel-grid-setup.md) handles spherical coordinates and [Incremental Back-Rotation for Voxel Mapping](6759-incremental-back-rotation-for-voxel-mapping.md) handles dynamic coordinate updates during rotation.

## How it works
**Inputs:**
- `g_aabbModel`: An array of two 3D vectors representing the minimum and maximum corners of the model's AABB in world space.
- `g_gridSizeX`, `g_gridSizeY`, `g_gridSizeZ`: Integer dimensions of the voxel grid along each axis.
- `g_useCubeVoxels`: A boolean flag indicating whether voxels must be cubic (uniform dimensions across axes).

**Outputs:**
- `g_voxelSpace[0]`, `g_voxelSpace[1]`: The minimum and maximum coordinates of the voxel space in world units.
- `g_matWorldToVoxel`: A 4x4 transformation matrix mapping world coordinates to voxel grid indices.
- `g_matWorldToVoxelProj`: A 4x4 transformation matrix mapping world coordinates to Normalized Device Coordinates (NDC) for projection.
- `g_strideX`, `g_strideY`, `g_dataSize`: Memory layout parameters for the voxel grid buffer.

**Data Structures:**
- The voxel grid is stored in a 1D buffer with a specific stride layout to ensure alignment.
- `g_strideX`: Calculated as `(g_gridSizeZ + 31) / 32`. This pads the Z-dimension stride to 32-bit words (1024 integers if each voxel is 4 bytes, or similar alignment depending on data type).
- `g_strideY`: Calculated as `g_strideX * g_gridSizeX`.
- `g_dataSize`: Calculated as `g_strideY * g_gridSizeY`.

**Steps:**
1. **Calculate Strides:** Compute the memory strides for the voxel grid to ensure proper alignment.
   $$ \text{stride}_X = \lceil \frac{\text{gridSize}_Z}{32} \rceil $$
   $$ \text{stride}_Y = \text{stride}_X \times \text{gridSize}_X $$
   $$ \text{dataSize} = \text{stride}_Y \times \text{gridSize}_Y $$

2. **Compute Model Extent:** Determine the size of the AABB.
   $$ \text{extent} = \text{AABB}_{max} - \text{AABB}_{min} $$

3. **Apply Padding:** Scale the extent by a factor that adds padding to the grid dimensions. The padding is 2 units total (1 unit on each side) to prevent edge cases where triangles exactly on the boundary are missed [S1, unofficial].
   $$ \text{extent}_{padded} = \text{extent} \times \frac{\text{gridSize} + 2}{\text{gridSize}} $$
   Where the division and multiplication are component-wise.

4. **Enforce Cube Voxels (Optional):** If `g_useCubeVoxels` is true, make the extent uniform by taking the maximum of the X, Y, and Z components.
   $$ \text{extent}_{final} = \max(\text{extent}_{padded}.x, \text{extent}_{padded}.y, \text{extent}_{padded}.z) \times [1, 1, 1] $$

5. **Compute Center:** Calculate the center of the original AABB.
   $$ \text{center} = \frac{\text{AABB}_{min} + \text{AABB}_{max}}{2} $$

6. **Compute Voxel Space Bounds:** Determine the minimum coordinate of the voxel space in world units.
   $$ \text{voxelSpaceMin} = \text{center} - 0.5 \times \text{extent}_{final} $$
   $$ \text{voxelSpaceMax} = \text{voxelSpaceMin} + \text{extent}_{final} $$

7. **Compute World-to-Voxel Matrix:** Construct the transformation matrix that maps world coordinates to voxel indices. This is a translation to `voxelSpaceMin` followed by a scaling to `gridSize`.
   $$ \text{g\_matWorldToVoxel} = \text{Translate}(-\text{voxelSpaceMin}) \times \text{Scale}\left(\frac{\text{gridSize}}{\text{extent}_{final}}\right) $$
   Note: The matrix is typically transposed for row-major storage or specific shader expectations [S1, unofficial].

8. **Compute World-to-Projection Matrix:** Construct the transformation matrix for projection purposes. This maps the world to NDC.
   $$ \text{g\_matWorldToVoxelProj} = \text{Translate}(-\text{center}.x, -\text{center}.y, -\text{voxelSpaceMin}.z) \times \text{Scale}\left(\frac{2.0}{\text{extent}.x}, \frac{2.0}{\text{extent}.y}, \frac{1.0}{\text{extent}.z}, 0.0\right) $$
   Note: The Z-scaling uses 1.0 instead of 2.0, likely to map to a [0, 1] range or specific depth buffer requirements [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def setup_voxelization(aabb_min, aabb_max, grid_size_x, grid_size_y, grid_size_z, use_cube_voxels):
    # 1. Calculate Strides
    stride_x = (grid_size_z + 31) // 32
    stride_y = stride_x * grid_size_x
    data_size = stride_y * grid_size_y

    # 2. Compute Model Extent
    extent = aabb_max - aabb_min

    # 3. Apply Padding
    # Padding factor: (gridSize + 2) / gridSize
    padding_factor_x = (grid_size_x + 2) / grid_size_x
    padding_factor_y = (grid_size_y + 2) / grid_size_y
    padding_factor_z = (grid_size_z + 2) / grid_size_z
    
    extent = extent * vector(padding_factor_x, padding_factor_y, padding_factor_z)

    # 4. Enforce Cube Voxels
    if use_cube_voxels:
        max_extent = max(extent.x, max(extent.y, extent.z))
        extent = vector(max_extent, max_extent, max_extent)

    # 5. Compute Center
    center = (aabb_min + aabb_max) / 2.0

    # 6. Compute Voxel Space Bounds
    voxel_space_min = center - 0.5 * extent
    voxel_space_max = voxel_space_min + extent

    # 7. Compute World-to-Voxel Matrix
    # Translate by -voxel_space_min
    # Scale by gridSize / extent
    scale_x = grid_size_x / extent.x
    scale_y = grid_size_y / extent.y
    scale_z = grid_size_z / extent.z
    
    mat_world_to_voxel = identity_matrix()
    # Translation part
    mat_world_to_voxel[3][0] = -voxel_space_min.x * scale_x
    mat_world_to_voxel[3][1] = -voxel_space_min.y * scale_y
    mat_world_to_voxel[3][2] = -voxel_space_min.z * scale_z
    
    # Scale part (diagonal)
    mat_world_to_voxel[0][0] = scale_x
    mat_world_to_voxel[1][1] = scale_y
    mat_world_to_voxel[2][2] = scale_z
    
    # Note: The source code transposes the result of XMMatrixMultiplyTranspose.
    # Assuming standard column-major storage in memory but row-major logic in math:
    # The resulting matrix M such that v_voxel = M * v_world
    # If using row vectors: v_voxel = v_world * M^T
    
    # 8. Compute World-to-Projection Matrix
    # Translate by (-center.x, -center.y, -voxel_space_min.z)
    # Scale by (2.0/extent.x, 2.0/extent.y, 1.0/extent.z, 0.0)
    proj_scale_x = 2.0 / extent.x
    proj_scale_y = 2.0 / extent.y
    proj_scale_z = 1.0 / extent.z
    
    mat_world_to_voxel_proj = identity_matrix()
    # Translation part
    mat_world_to_voxel_proj[3][0] = -center.x * proj_scale_x
    mat_world_to_voxel_proj[3][1] = -center.y * proj_scale_y
    mat_world_to_voxel_proj[3][2] = -voxel_space_min.z * proj_scale_z
    
    # Scale part
    mat_world_to_voxel_proj[0][0] = proj_scale_x
    mat_world_to_voxel_proj[1][1] = proj_scale_y
    mat_world_to_voxel_proj[2][2] = proj_scale_z
    
    return {
        'stride_x': stride_x,
        'stride_y': stride_y,
        'data_size': data_size,
        'voxel_space_min': voxel_space_min,
        'voxel_space_max': voxel_space_max,
        'mat_world_to_voxel': mat_world_to_voxel,
        'mat_world_to_voxel_proj': mat_world_to_voxel_proj
    }
```

**Edge Cases and Pitfalls:**
- **Boundary Misses:** Without the 2-unit padding, triangles exactly on the AABB boundary may be missed during rasterization or traversal [S1, unofficial].
- **Cube Voxel Enforcement:** If `g_useCubeVoxels` is true, the grid becomes isotropic, which may waste memory if the scene is highly anisotropic.
- **Precision:** Floating-point precision issues may arise if the AABB is very large or very small relative to the grid size.

## Subtopics

### [Spherical Voxel Grid Setup](3883-spherical-voxel-grid-setup.md)
Spherical Voxel Grid Setup initializes the data structures required for efficient ray casting traversal within a spherical coordinate volume. It computes discretization steps, precomputes trigonometric values, and constructs geometric primitives (line segments) that define voxel boundaries, enabling the `walkSphericalVolume` algorithm to determine ray-voxel intersections. It differs from Cartesian voxel grid setups by operating in an $(r, \theta, \phi)$ coordinate system tailored for egocentric view synthesis, where resolution requirements are anisotropic and radial distances vary significantly.

### [Incremental Back-Rotation for Voxel Mapping](6759-incremental-back-rotation-for-voxel-mapping.md)
Incremental back-rotation is an algorithm used to map destination voxel coordinates to their corresponding source voxel positions during a room rotation operation, specifically to eradicate holes produced by forward rotation. It sets itself apart from forward mapping by iterating over the destination grid and computing source locations, ensuring every destination voxel is filled. This method builds on [Voxel Space Setup](3520-voxel-space-setup.md) by utilizing the established transformation matrices to perform efficient, incremental coordinate calculations.

### [Coordinate Conversion Utilities](6766-coordinate-conversion-utilities.md)
Coordinate Conversion Utilities provide the mathematical and algorithmic primitives to map positions between world space, chunk space, and local voxel space within a dense voxel grid. They solve the problem of addressing volumetric data stored in spatially partitioned chunks, ensuring that global coordinates are correctly resolved to local indices for memory access and that local indices are correctly mapped back to world coordinates for rendering or physics. This method is distinguished from its siblings by focusing on the fundamental integer arithmetic of chunk-based indexing and axis alignment, rather than spherical mapping [Siblings: Spherical Voxel Grid Setup](3883-spherical-voxel-grid-setup.md), incremental back-rotation [Siblings: Incremental Back-Rotation for Voxel Mapping](6759-incremental-back-rotation-for-voxel-mapping.md), or 2D texture packing [Siblings: PixelGrid2D Construction](9430-pixelgrid2d-construction.md).

### [PixelGrid2D Construction](9430-pixelgrid2d-construction.md)
A 2D grid containing voxel information, implemented as a `Grid2D<BitArray, bool>`, initialized from a bounding box, voxel dimensions, and a BitArray.

### OpenVDB Density Grid Loading
Loads a heterogeneous density field from an OpenVDB file to provide spatial density queries for participating media [S2, unofficial]. It computes the AABB from the active voxel bounding box using `gridPtr->evalActiveVoxelBoundingBox()` and converts indices to world coordinates. Density at a given world position is interpolated using `openvdb::tools::GridSampler` with `openvdb::tools::BoxSampler`. The maximum density is retrieved via `gridPtr->evalMinMax(minValue, maxValue)` [S2, unofficial].

### Ray Tracing Volume Data Setup
Typical simplifications include an integral mapping of voxel address space to pixel address space (1-1 is easiest; usually projection of a voxel map to $k \times k$ pixels); arbitrary mapping requires interpolation [S3, unofficial]. The sampling lattice is rectilinear and uniformly spaced [S3, unofficial]. Pixel spacing must be less than voxel spacing [S3, unofficial]. Use one ray per pixel (no supersampling) and parallel projection [S3, unofficial].

### Volume Normalization to Unit Cube
Computes the bounding box (bb_min, bb_max) of the volume grid by transforming the grid's corner points (0,0,0) and (index_extent, index_extent, index_extent) using the grid's transform matrix [S4, unofficial]. Calculates the extent vector (bb_max - bb_min) and the maximum dimension size [S4, unofficial]. If size is not 1.0, constructs a transformation matrix that scales the volume by 1/size and translates it so its center is at the origin (subtracting bb_min + 0.5 * extent) [S4, unofficial]. Multiplies the density_scale parameter by the original size to compensate for the spatial scaling [S4, unofficial]. Initializes `bb_min` to `glm::vec3(FLT_MAX)` and `bb_max` to `glm::vec3(FLT_MIN)` [S4, unofficial]. Iterates over all frames in `volume->grids`, updating `bb_min` and `bb_max` with the transformed corner points [S4, unofficial].

### Voxel Center Precomputation
Precomputes the 3D coordinates of voxel centers for a regular grid defined by origin, size, and dimensions [S5, unofficial]. Extracts dimensions d, h, w from grid_dims [S5, unofficial]. Computes x-coordinates: $x = grid\_origin[0] + \text{arange}(w) \times voxel\_size[0] + voxel\_size[0]/2$ [S5, unofficial]. Computes y-coordinates: $y = grid\_origin[1] + \text{arange}(h) \times voxel\_size[1] + voxel\_size[1]/2$ [S5, unofficial]. Computes z-coordinates: $z = grid\_origin[2] + \text{arange}(d) \times voxel\_size[2] + voxel\_size[2]/2$ [S5, unofficial]. Stacks coordinates into a grid using meshgrid with 'ij' indexing and returns tensor of shape (D,H,W,3) [S5, unofficial]. Parameters include `grid_origin` (Scene origin coordinates [x_min, y_min, z_min]) and `voxel_size` (Physical size of a single voxel [sx, sy, sz]) [S5, unofficial].

## Operating characteristics
**Time Complexity:**
- The setup process is $O(1)$ relative to the grid size, as it involves a fixed number of arithmetic operations on the AABB and grid dimensions.
- Memory allocation for strides is $O(1)$.

**Memory Complexity:**
- The voxel grid memory usage is $O(\text{gridSize}_X \times \text{gridSize}_Y \times \text{gridSize}_Z)$, with padding overhead.
- The stride calculation ensures memory alignment, which may increase memory usage slightly.

**Performance:**
- The method is efficient for static scenes where the AABB and grid size are known beforehand.
- Dynamic scenes may require recomputation of the transformation matrices if the AABB changes significantly.

**Strengths:**
- Simple and fast setup.
- Handles padding to avoid boundary artifacts.
- Supports cube voxel enforcement for isotropic resolution.

**Limitations:**
- Does not handle non-Cartesian coordinate systems (handled by [Spherical Voxel Grid Setup](3883-spherical-voxel-grid-setup.md)).
- Does not handle dynamic updates (handled by [Incremental Back-Rotation for Voxel Mapping](6759-incremental-back-rotation-for-voxel-mapping.md)).

**When to Use:**
- Use for static volume rendering where the scene geometry is bounded by a known AABB.
- Use when isotropic voxel resolution is required.

**When Not to Use:**
- Do not use for spherical or cylindrical coordinate systems.
- Do not use for dynamic scenes requiring frequent updates to the voxel grid coordinates.

## Background
Voxel Space Setup is a fundamental technique in volume rendering and voxelization, originating from the need to map continuous world coordinates to discrete voxel grids for efficient rendering and processing [S1, unofficial]. It is commonly used in GPU-based voxelization pipelines, such as those described in "GpuPro3Vox" [S1, unofficial], where it serves as the initial step in preparing the voxel grid for rasterization or ray casting. The method ensures that the voxel grid is properly aligned and padded to handle edge cases and maintain numerical stability.

## Sources
[S1] michschw/GpuPro3Vox, https://github.com/michschw/GpuPro3Vox, fetched 2026-10-26 (unofficial)
[S2] GitHub - yumcyaWiz/volpt: Volumetric path tracer written in C++, https://github.com/yumcyaWiz/volpt, fetched 2026-10-27 (unofficial)
[S3] PDF Direct Volume Rendering, https://www.cs.unh.edu/~cs880/lectureNotes/DVR.pdf, fetched 2026-10-28 (unofficial)
[S4] nihofm/volren: Simple OpenGL-based renderer for ..., https://github.com/nihofm/volren, fetched 2026-10-29 (unofficial)
[S5] Yang-SyZng/AVM-GS, https://github.com/Yang-SyZng/AVM-GS, fetched 2026-10-27 (unofficial)
