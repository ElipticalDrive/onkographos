# VoxelizationGrid
## Overview
`VoxelizationGrid` is a data structure representing an explicit voxelization grid, defining bounds, grid dimensions, and voxel sizes in 3D space [S1, unofficial]. It solves the problem of converting arbitrary mesh bounding boxes into a discrete, aligned 3D grid suitable for dense volume storage, ensuring that grid indices map consistently to world coordinates [S1, unofficial].
## How it works
**Inputs and Outputs:** The method takes geometric constraints (mesh vertices or explicit bounding box coordinates) and resolution parameters (`voxel_size` or `grid_size`) as input, and outputs an instance of the `VoxelizationGrid` dataclass containing validated, contiguous arrays defining the grid's spatial extent and discretization [S1, unofficial].

**Data Structures:** The core structure is a dataclass `VoxelizationGrid` with fields stored in contiguous NumPy arrays: `bounds_min_xyz` (`Float32Array`, shape `(3,)`), `bounds_max_xyz` (`Float32Array`, shape `(3,)`), `grid_size_xyz` (`Int32Array`, shape `(3,)`), `voxel_size_xyz` (`Float32Array`, shape `(3,)`), and `pixel_size` (`float` or `None`) [S1, unofficial].

**Coordinate Conventions:** Bounds and voxel sizes are stored in $(x, y, z)$ world order [S1, unofficial]. Dense volume arrays associated with this grid are stored in $(z, y, x)$ order [S1, unofficial]. `grid_size_xyz` corresponds to OpenGL texture size $(x, y, z)$ [S1, unofficial].

**Parameters:** `align_bounds_max` (`bool`, default `True`) determines if the maximum bound is aligned to the grid cell boundary (`computed_max`) or kept at the requested padded maximum (`padded_requested_max`) [S1, unofficial]. `padding` (`float` or `tuple`, default `0.0`) is added to the bounding box extent during grid construction [S1, unofficial].

**Numbered Steps:**
1. **Initialization (`__post_init__`)**: Convert `bounds_min_xyz`, `bounds_max_xyz`, and `voxel_size_xyz` to contiguous `Float32Array` reshaped to `(3,)`. Convert `grid_size_xyz` to contiguous `Int32Array` reshaped to `(3,)`. Validate that `grid_size_xyz` elements are $> 0$, `voxel_size_xyz` elements are $> 0$, and `bounds_max_xyz` is strictly larger than `bounds_min_xyz` [S1, unofficial].
2. **Construction from mesh (`from_mesh`)**: Apply an optional transform matrix to the input mesh vertices. Compute `bounds_min_xyz` as the axis-wise minimum of the transformed vertices. Compute `bounds_max_xyz` as the axis-wise maximum of the transformed vertices. Call `from_bounds` with these computed bounds, the provided `voxel_size`, `padding`, and `pixel_size` [S1, unofficial].
3. **Construction from bounds (`from_bounds`)**: Convert inputs to `Float32`/`Int32` arrays. Expand `padding` to a 3D vector. Compute `padded_min = bounds_min - padding` and `padded_requested_max = bounds_max + padding`. Compute `extent = padded_requested_max - padded_min` [S1, unofficial].
   - If `voxel_size` is provided and `grid_size_xyz` is `None`: compute `voxel_size_xyz`, `safe_extent = np.maximum(extent, voxel_size_xyz)`, `grid_size = np.maximum(np.ceil(safe_extent / voxel_size_xyz).astype(np.int32), 1)`, `aligned_max = padded_min + grid_size.astype(np.float32) * voxel_size_xyz` [S1, unofficial].
   - If `grid_size_xyz` is provided and `voxel_size` is `None`: validate `grid_size > 0`, compute `voxel_size_xyz = extent / grid_size.astype(np.float32)`, `aligned_max = padded_requested_max` [S1, unofficial].
   - If both are provided: validate `grid_size > 0`, compute `computed_max = padded_min + grid_size.astype(np.float32) * voxel_size_xyz`, set `aligned_max` to `computed_max` if `align_bounds_max` is `True`, else `padded_requested_max` [S1, unofficial].
   - Return instance with `padded_min`, `aligned_max`, `grid_size`, `voxel_size_xyz`, and `pixel_size` [S1, unofficial].
4. **Legacy construction (`from_legacy`)**: Convert `num_voxels_zyx` to `Int32Array`. Map legacy ZYX order to `grid_size_xyz` (X, Y, Z) by reversing indices: `grid_size_xyz = [legacy_num_voxels[2], legacy_num_voxels[1], legacy_num_voxels[0]]` [S1, unofficial]. If `voxel_size` is `None`, call `from_bounds` with `grid_size_xyz` [S1, unofficial]. If `voxel_size` is provided, call `from_bounds` with both `voxel_size` and `grid_size_xyz`, setting `align_bounds_max=True` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from dataclasses import dataclass
from typing import Optional, Union, Tuple

@dataclass
class VoxelizationGrid:
    bounds_min_xyz: np.ndarray  # Float32, shape (3,)
    bounds_max_xyz: np.ndarray  # Float32, shape (3,)
    grid_size_xyz: np.ndarray   # Int32, shape (3,)
    voxel_size_xyz: np.ndarray  # Float32, shape (3,)
    pixel_size: Optional[float]

    def __post_init__(self):
        self.bounds_min_xyz = np.asarray(self.bounds_min_xyz, dtype=np.float32).reshape(3)
        self.bounds_max_xyz = np.asarray(self.bounds_max_xyz, dtype=np.float32).reshape(3)
        self.grid_size_xyz = np.asarray(self.grid_size_xyz, dtype=np.int32).reshape(3)
        self.voxel_size_xyz = np.asarray(self.voxel_size_xyz, dtype=np.float32).reshape(3)
        if np.any(self.grid_size_xyz <= 0):
            raise ValueError("grid_size_xyz elements must be > 0")
        if np.any(self.voxel_size_xyz <= 0):
            raise ValueError("voxel_size_xyz elements must be > 0")
        if np.any(self.bounds_max_xyz <= self.bounds_min_xyz):
            raise ValueError("bounds_max_xyz must be strictly larger than bounds_min_xyz")

    @classmethod
    def from_mesh(cls, mesh_vertices: np.ndarray, voxel_size: Optional[Union[float, np.ndarray]] = None,
                  padding: Union[float, Tuple[float, float, float]] = 0.0,
                  transform: Optional[np.ndarray] = None,
                  pixel_size: Optional[float] = None) -> 'VoxelizationGrid':
        if transform is not None:
            if transform.shape == (4, 4):
                vertices_transformed = (mesh_vertices @ transform[:3, :3].T) + transform[:3, 3]
            else:
                vertices_transformed = mesh_vertices @ transform.T
        else:
            vertices_transformed = mesh_vertices
        bounds_min_xyz = np.min(vertices_transformed, axis=0)
        bounds_max_xyz = np.max(vertices_transformed, axis=0)
        return cls.from_bounds(bounds_min_xyz, bounds_max_xyz, voxel_size, padding, pixel_size)

    @classmethod
    def from_bounds(cls, bounds_min_xyz: np.ndarray, bounds_max_xyz: np.ndarray,
                    voxel_size: Optional[Union[float, np.ndarray]] = None,
                    padding: Union[float, Tuple[float, float, float]] = 0.0,
                    grid_size: Optional[np.ndarray] = None,
                    align_bounds_max: bool = True,
                    pixel_size: Optional[float] = None) -> 'VoxelizationGrid':
        bounds_min_xyz = np.asarray(bounds_min_xyz, dtype=np.float32).reshape(3)
        bounds_max_xyz = np.asarray(bounds_max_xyz, dtype=np.float32).reshape(3)
        if isinstance(padding, (int, float)):
            padding_vec = np.array([padding, padding, padding], dtype=np.float32)
        else:
            padding_vec = np.asarray(padding, dtype=np.float32).reshape(3)
        padded_min = bounds_min_xyz - padding_vec
        padded_requested_max = bounds_max_xyz + padding_vec
        extent = padded_requested_max - padded_min

        if voxel_size is not None and grid_size is None:
            voxel_size_xyz = np.asarray(voxel_size, dtype=np.float32).reshape(3)
            safe_extent = np.maximum(extent, voxel_size_xyz)
            grid_size_xyz = np.maximum(np.ceil(safe_extent / voxel_size_xyz).astype(np.int32), 1)
            aligned_max = padded_min + grid_size_xyz.astype(np.float32) * voxel_size_xyz
            return cls(padded_min, aligned_max, grid_size_xyz, voxel_size_xyz, pixel_size)
        elif grid_size is not None and voxel_size is None:
            grid_size_xyz = np.asarray(grid_size, dtype=np.int32).reshape(3)
            if np.any(grid_size_xyz <= 0):
                raise ValueError("grid_size must be > 0")
            voxel_size_xyz = extent / grid_size_xyz.astype(np.float32)
            aligned_max = padded_requested_max
            return cls(padded_min, aligned_max, grid_size_xyz, voxel_size_xyz, pixel_size)
        elif voxel_size is not None and grid_size is not None:
            grid_size_xyz = np.asarray(grid_size, dtype=np.int32).reshape(3)
            voxel_size_xyz = np.asarray(voxel_size, dtype=np.float32).reshape(3)
            if np.any(grid_size_xyz <= 0):
                raise ValueError("grid_size must be > 0")
            computed_max = padded_min + grid_size_xyz.astype(np.float32) * voxel_size_xyz
            aligned_max = computed_max if align_bounds_max else padded_requested_max
            return cls(padded_min, aligned_max, grid_size_xyz, voxel_size_xyz, pixel_size)
        else:
            raise ValueError("Either voxel_size or grid_size must be provided")

    @classmethod
    def from_legacy(cls, num_voxels_zyx: Union[Tuple[int, int, int], np.ndarray],
                    voxel_size: Optional[Union[float, np.ndarray]] = None,
                    padding: Union[float, Tuple[float, float, float]] = 0.0,
                    pixel_size: Optional[float] = None) -> 'VoxelizationGrid':
        legacy_arr = np.asarray(num_voxels_zyx, dtype=np.int32).reshape(3)
        grid_size_xyz = np.array([legacy_arr[2], legacy_arr[1], legacy_arr[0]], dtype=np.int32)
        if voxel_size is None:
            # [G] Source note says call from_bounds with grid_size_xyz, but bounds are required.
            # Assuming default unit bounds [0,0,0] to [1,1,1] to satisfy signature.
            return cls.from_bounds(np.zeros(3, dtype=np.float32), np.ones(3, dtype=np.float32),
                                   voxel_size=None, padding=padding, grid_size=grid_size_xyz, pixel_size=pixel_size)
        else:
            return cls.from_bounds(np.zeros(3, dtype=np.float32), np.ones(3, dtype=np.float32),
                                   voxel_size=voxel_size, padding=padding, grid_size=grid_size_xyz,
                                   align_bounds_max=True, pixel_size=pixel_size)
```

**Edge Cases and Pitfalls:**
*   **Precision:** Floating-point division in `voxel_size_xyz` calculation may introduce small errors. Using `np.maximum` with `voxel_size_xyz` in Case A ensures `safe_extent` is never smaller than the voxel size, preventing division by zero or negative grid sizes [S1, unofficial].
*   **Alignment:** When `align_bounds_max` is `False`, the grid bounds may not align with integer voxel boundaries relative to the original request [S1, unofficial].
*   **Degenerate Input:** If `bounds_max` equals `bounds_min`, the extent is zero. The validation step catches this, but if padding is large enough to create a non-zero extent, the grid will be generated based on the padded bounds [S1, unofficial].

## Operating characteristics
*   **Time and Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Provides a validated coordinate system for voxelization, handles padding and alignment explicitly [S1, unofficial].
*   **Limitations:** Does not perform actual voxelization (sampling/rasterization); requires downstream components for dense array storage [S1, unofficial].
*   **When to use:** Defining the spatial domain for a volume renderer or voxelizer [S1, unofficial].
*   **When not to use:** When sparse representations are required or grid dimensions are dynamic [G].

## Background
The `VoxelizationGrid` structure is part of the `Voxelerate` library [S1, unofficial]. It addresses the common need in computer graphics to map continuous mesh geometry to discrete voxel grids [S1, unofficial].

## Sources
[S1] Voxelerate, https://github.com/hsafari68/Voxelerate, fetched 2026-10-26 (unofficial)
