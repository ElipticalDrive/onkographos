# VoxelizationGrid

Also called: Voxel Grid Data Structure and I/O, VoxelGridBase

## Overview
`VoxelizationGrid` is a data structure that represents an explicit voxelization grid, defining spatial bounds, grid dimensions, and voxel sizes in 3D space [S2, S3, unofficial]. It serves as a base class for managing a 3D voxel grid in world space, providing utilities for coordinate conversion, ray intersection, and mesh generation [S3, unofficial]. It is distinct from sibling methods like [Instanced Voxel Rendering](10592-instanced-voxel-rendering.md) by focusing on the foundational storage and indexing logic rather than the rendering pipeline itself.

## How it works
**Inputs and Outputs:**
The primary input is a set of spatial parameters: bounding box (`bounds_min`, `bounds_max`), grid dimensions (`grid_size` or `voxel_size`), and optional padding [S2, S3, unofficial]. The output is an initialized grid object capable of mapping between world coordinates, integer cell indices, and linear array indices [S3, unofficial].

**Data Structures:**
There are two primary implementations described in the sources, which differ in storage and field layout:

1.  **Python/Dataclass Implementation (`VoxelizationGrid`)** [S2, unofficial]:
    *   `bounds_min_xyz`: `Float32Array`, shape `(3,)`. World coordinates of the minimum corner.
    *   `bounds_max_xyz`: `Float32Array`, shape `(3,)`. World coordinates of the maximum corner.
    *   `grid_size_xyz`: `Int32Array`, shape `(3,)`. Number of voxels along X, Y, Z. Corresponds to OpenGL texture size `(x, y, z)`.
    *   `voxel_size_xyz`: `Float32Array`, shape `(3,)`. World space size of a single voxel.
    *   `pixel_size`: `float` or `None`.
    *   *Note:* Dense volume arrays associated with this grid are stored in `(z, y, x)` order [S2, unofficial].

2.  **C++/Class Implementation (`VoxelGrid`)** [S1, S3, unofficial]:
    *   `dims` (`sibr::Vector3i`): Integer grid dimensions.
    *   `cellSize` (`sibr::Vector3f`): World space voxel size.
    *   `cellSizeNorm` (`float`): Euclidean norm (diagonal length) of `cellSize`.
    *   `box` (`Box`): The 3D bounding box delimiting the voxelized region.
    *   `voxelGrid`: A pointer array `vector<Voxel*>` storing voxel data.
    *   `voxelSize` (`double`): Uniform voxel size (in [S1, unofficial]).
    *   `defaultDensity` (`double`): Default density value (in [S1, unofficial]).
    *   `xCount, yCount, zCount` (`int`): Dimensions (in [S1, unofficial]).
    *   Transformation matrices: `mat`, `t_inv`, `tstar`, `tstar_inv` for world-to-grid conversion (in [S1, unofficial]).
    *   `baseCellMesh`, `baseCellMeshFilled`: Pre-generated wireframe and filled triangle meshes for a single voxel unit.
    *   `_generator`, `_distribution`: Random number generator (seeded at 0) and distribution for sampling (in [S3, unofficial]).

**Numbered Steps:**

1.  **Initialization from Bounds (`from_bounds`)** [S2, unofficial]:
    *   Convert `bounds_min_xyz` and `bounds_max_xyz` to contiguous `Float32Array` reshaped to 3.
    *   Convert `grid_size_xyz` to contiguous `Int32Array` reshaped to 3.
    *   Validate: `grid_size_xyz` elements > 0, `voxel_size_xyz` elements > 0, `bounds_max_xyz` strictly larger than `bounds_min_xyz`.
    *   Expand padding to a 3D vector. Compute `padded_min = bounds_min - pad` and `padded_requested_max = bounds_max + pad`.
    *   Compute `extent = padded_requested_max - padded_min`.
    *   *Case A (voxel_size provided, grid_size None):*
        *   `safe_extent = np.maximum(extent, voxel_size_xyz)`
        *   `grid_size = np.maximum(np.ceil(safe_extent / voxel_size_xyz).astype(np.int32), 1)`
        *   `aligned_max = padded_min + grid_size.astype(np.float32) * voxel_size_xyz`
    *   *Case B (grid_size provided, voxel_size None):*
        *   Validate `grid_size > 0`.
        *   `voxel_size_xyz = extent / grid_size.astype(np.float32)`
        *   `aligned_max = padded_requested_max`
    *   *Case C (both provided):*
        *   Validate `grid_size > 0`.
        *   `computed_max = padded_min + grid_size.astype(np.float32) * voxel_size_xyz`
        *   If `align_bounds_max` is True, `aligned_max = computed_max`; else `aligned_max = padded_requested_max`.
    *   Return instance with `padded_min`, `aligned_max`, `grid_size`, `voxel_size_xyz`, `pixel_size`.

2.  **Initialization from Legacy Format (`from_legacy`)** [S2, unofficial]:
    *   Convert `num_voxels_zyx` to `Int32Array`.
    *   Map legacy ZYX order to `grid_size_xyz` by reversing indices: `grid_size_xyz = [legacy_num_voxels[2], legacy_num_voxels[1], legacy_num_voxels[0]]`.
    *   If `voxel_size` is None, call `from_bounds` with `grid_size_xyz`.
    *   If `voxel_size` is provided, call `from_bounds` with both `voxel_size` and `grid_size_xyz`, setting `align_bounds_max=True`.

3.  **Initialization from Mesh (`from_mesh`)** [S2, unofficial]:
    *   Apply optional transform to mesh vertices.
    *   Compute `bounds_min_xyz` as min of transformed vertices along axis 0.
    *   Compute `bounds_max_xyz` as max of transformed vertices along axis 0.
    *   Call `from_bounds` with these bounds, provided `voxel_size`, `padding`, `transform`, and `pixel_size`.

4.  **C++ Specific Initialization (`forceCube`)** [S3, unofficial]:
    *   If `forceCube` is true, calculate `maxDimSize` as the maximum component of `box.sizes()` divided by `numPerDim` cast to float.
    *   For each axis `c`, update `dims[c]` to `round(box.sizes()[c] / maxDimSize)`.
    *   Calculate `cellSize` as `box.sizes()` divided element-wise by `dims` cast to float.
    *   Calculate `cellSizeNorm` as the Euclidean norm of `cellSize`.

5.  **Coordinate Conversion:**
    *   **World to Grid:** Construct a 4x4 matrix `mat` with scaling by dimensions and translation to center. Precompute inverse matrices `t_inv` and `tstar_inv` [S1, unofficial].
    *   **Linear ID to 3D Indices:** `getCell(cellId)` converts linear ID to 3D indices. Throws error if `cellId >= getNumCells()` [S3, unofficial].
    *   **3D Indices to Linear ID:** `getCellId(cell)` converts 3D integer coordinates to linear ID [S3, unofficial].
    *   **World Position to Linear ID:** `getCellId(world_pos)` converts 3D world position to linear ID of containing voxel [S3, unofficial].
    *   **Linear ID to World Position:** `getCellCenter(cellId)` returns the center 3D position of the voxel at linear ID `cellId` [S3, unofficial].

6.  **Voxel Storage Layout:**
    *   In the C++ implementation [S1, unofficial], voxels are stored in a `std::vector<Voxel*>`. The order is Z-major: outer loop `kk` (0..zCount-1), middle `jj` (0..yCount-1), inner `ii` (0..xCount-1).
    *   The linear index is calculated as `x + y*xCount + z*xCount*yCount` [S1, unofficial].
    *   Each `Voxel` struct contains `vec3 color` (RGB), `double density`, and `double transmissivity` [S1, unofficial].

7.  **Mesh Generation:**
    *   Initialize `baseCellMesh` and `baseCellMeshFilled` [S3, unofficial].
    *   Create a base cell box extending from `box.min()` to `box.min() + getCellSize()` [S3, unofficial].
    *   Generate 8 vertices for cube corners.
    *   Wireframe uses predefined triangle indices for edges. Filled uses predefined triangle indices for faces [S3, unofficial].
    *   `getCellMesh(cell)` generates wireframe for voxel at integer coordinates `cell` [S3, unofficial].
    *   `getAllCellMesh()` generates wireframe for all voxels [S3, unofficial].
    *   `getCellMeshFilled(cell)` generates filled triangle mesh for voxel at `cell` [S3, unofficial].
    *   `getAllCellMeshFilled()` generates filled triangle mesh for all voxels [S3, unofficial].

8.  **Ray Intersection:**
    *   `rayMarch(ray)` intersects a ray with the voxel grid and returns a vector of linear IDs of intersected voxels [S3, unofficial].
    *   `intersectionWithBox(ray, intersection)` checks if ray intersects the voxel grid. If so, sets `intersection` to the position and returns true [S3, unofficial].

9.  **Neighbor and Boundary Checks:**
    *   `outOfBounds(v)` returns true if any component of `v` is less than 0 or greater than or equal to the corresponding component of `dims` [S3, unofficial].
    *   `isInside(worldPos)` returns true if `box.contains(worldPos)` [S3, unofficial].
    *   `getNeighbors(cellId)` returns a vector of linear IDs of the neighbors of the voxel at `cellId` [S3, unofficial].
    *   `getCell(worldPos)` returns the integer coordinates of the cell containing `worldPos` [S3, unofficial].
    *   `getCellInclusive(worldPos)` returns the integer coordinates of the cell containing `worldPos`. Positions on boundaries belong to the closest cell [S3, unofficial].

10. **Grid Extension:**
    *   `extend(numCells)` extends the voxel grid along all dimensions by adding `numCells` to each dimension (new count N+2*numCells) [S3, unofficial].

11. **Sampling:**
    *   `sampleCell(cellId)` samples a random position in the voxel. Implementation: `center + random(-1,1)^3 * cellSize`. Random generator is seeded at 0 [S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

class VoxelizationGrid:
    def __init__(self, bounds_min_xyz, bounds_max_xyz, grid_size_xyz=None, voxel_size_xyz=None, padding=0.0, align_bounds_max=True):
        self.bounds_min_xyz = np.array(bounds_min_xyz, dtype=np.float32)
        self.bounds_max_xyz = np.array(bounds_max_xyz, dtype=np.float32)
        self.padding = padding
        self.align_bounds_max = align_bounds_max
        
        # Validation
        if not np.all(self.bounds_max_xyz > self.bounds_min_xyz):
            raise ValueError("bounds_max must be strictly larger than bounds_min")
            
        self._initialize_grid(grid_size_xyz, voxel_size_xyz)

    def _initialize_grid(self, grid_size_xyz, voxel_size_xyz):
        # Convert padding to 3D vector
        if isinstance(self.padding, (int, float)):
            pad = np.array([self.padding, self.padding, self.padding], dtype=np.float32)
        else:
            pad = np.array(self.padding, dtype=np.float32)
            
        padded_min = self.bounds_min_xyz - pad
        padded_requested_max = self.bounds_max_xyz + pad
        extent = padded_requested_max - padded_min
        
        if grid_size_xyz is not None and voxel_size_xyz is None:
            grid_size_xyz = np.array(grid_size_xyz, dtype=np.int32)
            if not np.all(grid_size_xyz > 0):
                raise ValueError("grid_size must be > 0")
            voxel_size_xyz = extent / grid_size_xyz.astype(np.float32)
            aligned_max = padded_requested_max
        elif grid_size_xyz is None and voxel_size_xyz is not None:
            voxel_size_xyz = np.array(voxel_size_xyz, dtype=np.float32)
            if not np.all(voxel_size_xyz > 0):
                raise ValueError("voxel_size must be > 0")
            safe_extent = np.maximum(extent, voxel_size_xyz)
            grid_size_xyz = np.maximum(np.ceil(safe_extent / voxel_size_xyz).astype(np.int32), 1)
            aligned_max = padded_min + grid_size_xyz.astype(np.float32) * voxel_size_xyz
        elif grid_size_xyz is not None and voxel_size_xyz is not None:
            grid_size_xyz = np.array(grid_size_xyz, dtype=np.int32)
            voxel_size_xyz = np.array(voxel_size_xyz, dtype=np.float32)
            if not np.all(grid_size_xyz > 0):
                raise ValueError("grid_size must be > 0")
            computed_max = padded_min + grid_size_xyz.astype(np.float32) * voxel_size_xyz
            if self.align_bounds_max:
                aligned_max = computed_max
            else:
                aligned_max = padded_requested_max
        else:
            raise ValueError("Either grid_size_xyz or voxel_size_xyz must be provided")
            
        self.grid_size_xyz = grid_size_xyz
        self.voxel_size_xyz = voxel_size_xyz
        self.aligned_max = aligned_max
        self.padded_min = padded_min

    def get_cell_id_from_world(self, world_pos):
        """Converts world position to linear cell ID."""
        # Calculate relative position within the grid bounds
        rel_pos = (world_pos - self.padded_min) / self.voxel_size_xyz
        cell_coords = np.floor(rel_pos).astype(np.int32)
        
        # Check bounds
        if not self._is_inside_bounds(cell_coords):
            return -1
            
        # Convert 3D coords to linear ID (X-major in Python logic, though storage may vary)
        # Note: Source [S1, unofficial] uses X + Y*XCount + Z*XCount*YCount for linear indexing
        x, y, z = cell_coords
        x_count, y_count, z_count = self.grid_size_xyz
        return x + y * x_count + z * x_count * y_count

    def _is_inside_bounds(self, cell_coords):
        x, y, z = cell_coords
        x_count, y_count, z_count = self.grid_size_xyz
        return (0 <= x < x_count) and (0 <= y < y_count) and (0 <= z < z_count)

    def get_cell_center(self, cell_id):
        """Returns the center 3D position of the voxel at linear ID cell_id."""
        x_count, y_count, z_count = self.grid_size_xyz
        x = cell_id % x_count
        y = (cell_id // x_count) % y_count
        z = cell_id // (x_count * y_count)
        
        # Calculate min corner of the cell
        cell_min = self.padded_min + np.array([x, y, z], dtype=np.float32) * self.voxel_size_xyz
        # Center is min + half voxel size
        return cell_min + self.voxel_size_xyz / 2.0

    def sample_cell(self, cell_id):
        """Samples a random position in the voxel."""
        center = self.get_cell_center(cell_id)
        # Random generator seeded at 0, range -1 to 1
        # Using numpy default generator for simplicity, assuming seed 0
        np.random.seed(0)
        random_offset = (np.random.rand(3) * 2 - 1) * self.voxel_size_xyz
        return center + random_offset
```

**Parameters:**
*   `align_bounds_max` (bool, default True): In `from_bounds`, determines if the maximum bound is aligned to the grid cell boundary (`computed_max`) or kept at the requested padded maximum (`padded_requested_max`) [S2, unofficial].
*   `padding` (float or tuple, default 0.0): Added to the bounding box extent during grid construction in `from_bounds` [S2, unofficial].
*   `forceCube` (bool): If true, the largest dimension is split into `numPerDim` voxels and others adjusted so voxels are cubes in world space [S3, unofficial].

**Edge Cases and Pitfalls:**
*   Accessing voxels outside the grid bounds `[0, xCount)`, `[0, yCount)`, `[0, zCount)` returns default values (0 for color/transmissivity, `defaultDensity` for density) or ignores the set operation [S1, unofficial].
*   `getCell` throws an error if `cellId >= getNumCells()` [S3, unofficial].
*   Precision issues may arise when `bounds_max` is not strictly larger than `bounds_min` or when `voxel_size` is extremely small [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Coordinate conversion is O(1). Ray marching is O(N) where N is the number of voxels intersected. Mesh generation for all cells is O(N).
*   **Memory Complexity:** O(N) where N is the total number of voxels (`xCount * yCount * zCount`). The C++ implementation stores pointers to `Voxel` structs, adding overhead [S1, unofficial].
*   **Strengths:** Provides a unified interface for coordinate transformation, ray intersection, and mesh generation. Supports flexible grid construction from bounds, mesh, or legacy formats.
*   **Limitations:** The C++ implementation uses a flat array of pointers, which may have cache locality issues compared to contiguous arrays. The Python implementation relies on NumPy arrays, which may have memory overhead for very large grids.
*   **When to use:** When a explicit voxel grid representation is needed for coordinate mapping, ray casting, or visualization.
*   **When not to use:** When memory efficiency is critical and sparse representations (like [Instanced Voxel Rendering](10592-instanced-voxel-rendering.md)) are more appropriate.

## Background
The `VoxelizationGrid` concept appears in multiple open-source projects. The Python implementation is found in `Voxelerate` [S2, unofficial], while the C++ implementation is part of the `FDS-GS` project [S3, unofficial]. The underlying voxelization and rendering techniques are also present in `VolumetricClouds` [S1, unofficial].

## Sources
[S1] dotcamdev/VolumetricClouds, https://github.com/dotcamdev/VolumetricClouds, fetched 2026-10-26 (unofficial)
[S2] hsafari68/Voxelerate, https://github.com/hsafari68/Voxelerate, fetched 2026-10-26 (unofficial)
[S3] whoiszzj/FDS-GS, https://github.com/whoiszzj/FDS-GS, fetched 2026-10-27 (unofficial)
