# SparseGrid

Also called: SparseGrid Data Structure

## Overview
SparseGrid is a sparse adaptive grid structure that allocates memory only for cells explicitly added to the domain, making it suitable for large 3D spaces containing small iso-surface regions [S1, unofficial]. It differs from hash grids by maintaining an explicit lattice structure defined by a shape and axis-aligned bounding box (AABB), rather than relying solely on hash collisions for spatial mapping [S1, unofficial].

## How it works
**Inputs and Outputs:**
The structure is initialized with a 3D integer `shape` (number of sample points per dimension), an AABB (`aabb_min`, `aabb_max`), and a `default_value` for unset regions [S1, unofficial]. It accepts 1D `int32` tensors of linear cell indices for adding or removing cells [S1, unofficial]. It outputs tensors of corner coordinates, scalar values, and connectivity data for active cells [S1, unofficial].

**Data Structures:**
The core storage resides on the GPU device using Thrust device vectors [S1, unofficial]:
- `thrust::device_vector<float> values`: Stores scalar values at cell corners. Layout is $(N, 8)$, where $N$ is the number of active cells.
- `thrust::device_vector<uint> cell_indices`: Stores linear indices of active cells. Layout is 1D, size $N$.
- `uint3 shape`: Dimensions of the implied lattice.
- `float3 aabb_min`: Minimum corner of the bounding box. Default: $[-1, -1, -1]$ [S1, unofficial].
- `float3 aabb_max`: Maximum corner of the bounding box. Default: $[1, 1, 1]$ [S1, unofficial].
- `float default_value`: Value for unset points. Default: `FLT_MAX` [S1, unofficial].

**Linear Indexing:**
Cells are identified by linear indices within the implied lattice defined by `shape`. The linear index $L$ for a cell at grid coordinates $(x, y, z)$ is computed as:
$$ L = x \cdot (shape.y \cdot shape.z) + y \cdot shape.z + z $$
where $0 \le x < shape.x$, $0 \le y < shape.y$, and $0 \le z < shape.z$ [S1, unofficial].

**Algorithm Steps:**

1.  **Initialization:** Construct the grid with `shape`, `aabb_min`, `aabb_max`, and `default_value`. Initialize `values` and `cell_indices` as empty device vectors [S1, unofficial].
2.  **Add Cells:** When `add_cells` is called with a tensor of linear indices:
    -   For each index $L$ in the input tensor:
        -   Check if $L$ is already in `cell_indices`. If not, append $L$ to `cell_indices`.
        -   Compute the 8 corner coordinates of the cell in world space using the lattice geometry and AABB.
        -   Initialize the corresponding 8 entries in `values` to `default_value` [S1, unofficial].
3.  **Remove Cells:** When `remove_cells` is called with a tensor of linear indices:
    -   For each index $L$ in the input tensor:
        -   Find $L$ in `cell_indices`.
        -   Remove it from `cell_indices` (e.g., swap with last element and pop back, or use a compacting filter).
        -   Remove the corresponding 8 entries from `values` [S1, unofficial].
4.  **Get Points:** `get_points_by_cell_indices` takes a tensor of cell indices and returns an $(N, 8, 3)$ `float32` tensor of corner coordinates. For each cell index, compute the 8 corners in world space [S1, unofficial].
5.  **Get Values:** `get_values` returns an $(N, 8)$ `float32` tensor of scalar values at the corners of all active cells [S1, unofficial].
6.  **Get Cell Indices:** `get_cell_indices` returns the 1D `int32` tensor of linear indices of all active cells [S1, unofficial].
7.  **Filter Iso-surface:** `filter_cell_indices` takes `cell_indices`, `values` (N, 8), and a `level` (default 0.0). It returns a 1D `int32` tensor of indices for cells where the scalar values cross the iso-surface level. A cell crosses the surface if the values at its corners are not all greater than or equal to the level, or not all less than or equal to the level [S1, unofficial].
8.  **Get Dual Quads:** `get_dual_quads` takes edge data (`NDArray<uint2> edges`) and an `is_out` mask (`NDArray<bool>`). It returns a tuple of:
    -   `thrust::device_vector<int4>`: Quad definitions.
    -   `thrust::device_vector<bool>`: Output flags.
    -   `thrust::device_vector<uint2>`: Deduplicated edges [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

class SparseGrid:
    def __init__(self, shape: tuple, aabb_min: tuple = (-1, -1, -1), 
                 aabb_max: tuple = (1, 1, 1), default_value: float = float('inf')):
        self.shape = shape
        self.aabb_min = np.array(aabb_min, dtype=np.float32)
        self.aabb_max = np.array(aabb_max, dtype=np.float32)
        self.default_value = default_value
        
        # Active cells stored as a set for O(1) lookup, converted to list for tensor ops
        self.active_cell_indices = set()
        # Values stored as a dict mapping linear_index -> list of 8 floats
        self.cell_values = {}

    def _linear_index_to_coords(self, idx: int) -> tuple:
        """Convert linear index to (x, y, z) grid coordinates."""
        x = idx // (self.shape[1] * self.shape[2])
        rem = idx % (self.shape[1] * self.shape[2])
        y = rem // self.shape[2]
        z = rem % self.shape[2]
        return (x, y, z)

    def _coords_to_world(self, x: int, y: int, z: int) -> np.ndarray:
        """Convert grid coordinates to world space coordinates."""
        # Assuming uniform grid spacing implied by shape and AABB
        # Cell size in each dimension
        dx = (self.aabb_max[0] - self.aabb_min[0]) / self.shape[0]
        dy = (self.aabb_max[1] - self.aabb_min[1]) / self.shape[1]
        dz = (self.aabb_max[2] - self.aabb_min[2]) / self.shape[2]
        
        # Corner coordinates (8 corners of the cell)
        corners = []
        for cx in [x, x + 1]:
            for cy in [y, y + 1]:
                for cz in [z, z + 1]:
                    wx = self.aabb_min[0] + cx * dx
                    wy = self.aabb_min[1] + cy * dy
                    wz = self.aabb_min[2] + cz * dz
                    corners.append([wx, wy, wz])
        return np.array(corners, dtype=np.float32)

    def add_cells(self, indices: np.ndarray):
        """Add cells to the grid."""
        for idx in indices:
            if idx not in self.active_cell_indices:
                self.active_cell_indices.add(idx)
                # Initialize values for 8 corners
                self.cell_values[idx] = [self.default_value] * 8

    def remove_cells(self, indices: np.ndarray):
        """Remove cells from the grid."""
        for idx in indices:
            if idx in self.active_cell_indices:
                self.active_cell_indices.remove(idx)
                del self.cell_values[idx]

    def get_points(self) -> np.ndarray:
        """Return (N, 8, 3) tensor of corner coordinates for active cells."""
        if not self.active_cell_indices:
            return np.empty((0, 8, 3), dtype=np.float32)
        
        points_list = []
        for idx in sorted(self.active_cell_indices):
            x, y, z = self._linear_index_to_coords(idx)
            corners = self._coords_to_world(x, y, z)
            points_list.append(corners)
        return np.array(points_list, dtype=np.float32)

    def get_values(self) -> np.ndarray:
        """Return (N, 8) tensor of scalar values at active cell corners."""
        if not self.active_cell_indices:
            return np.empty((0, 8), dtype=np.float32)
        
        values_list = []
        for idx in sorted(self.active_cell_indices):
            values_list.append(self.cell_values[idx])
        return np.array(values_list, dtype=np.float32)

    def get_cell_indices(self) -> np.ndarray:
        """Return 1D int32 tensor of linear indices of active cells."""
        return np.array(sorted(self.active_cell_indices), dtype=np.int32)

    def filter_cell_indices(self, level: float = 0.0) -> np.ndarray:
        """Return indices of cells crossing the iso-surface."""
        crossing_indices = []
        for idx in self.active_cell_indices:
            vals = self.cell_values[idx]
            # Check if values cross the level
            # A cell crosses if not all values >= level and not all values <= level
            all_ge = all(v >= level for v in vals)
            all_le = all(v <= level for v in vals)
            if not all_ge and not all_le:
                crossing_indices.append(idx)
        return np.array(crossing_indices, dtype=np.int32)

    def get_num_cells(self) -> int:
        return len(self.active_cell_indices)

    def get_num_points(self) -> int:
        return len(self.active_cell_indices) * 8
```

**Edge Cases and Pitfalls:**
-   **Precision:** Linear indexing assumes the grid fits in memory addressable by `uint`. For very large `shape`, overflow may occur.
-   **Boundaries:** The AABB defines the world space extent. Cells are added based on linear indices within the `shape`. If `shape` is large but only a few cells are added, the structure remains sparse.
-   **Degenerate Input:** If `shape` contains zeros, division by zero may occur in coordinate conversion. [G] Guard against zero dimensions in `shape`.

## Subtopics

### Hash Grids
Hash Grids use hash functions to map 3D coordinates to memory locations, offering O(1) average access time but no inherent spatial locality. Choose SparseGrid when explicit lattice structure and AABB-based world space mapping are required, or when traversal order matters. [Link](936-hash-grids.md)

### Stack-Based Representation (SBR)
SBR stores active voxels in a linear stack, often with run-length encoding. It is highly compact for contiguous regions. Choose SparseGrid when random access to specific cell indices is needed or when dynamic addition/removal of arbitrary cells is frequent. [Link](1081-stack-based-representation-sbr.md)

### Pointer Compression
Pointer Compression reduces memory overhead by compressing pointers in tree structures. SparseGrid does not use pointers for spatial traversal, relying instead on linear indices. Choose Pointer Compression for deep tree structures where pointer overhead dominates. [Link](1770-pointer-compression.md)

### Brick-based Hierarchical Storage
Brick-based storage divides the volume into bricks (sub-grids), allowing for hierarchical compression. Choose this for large volumes with multi-resolution requirements. SparseGrid is flat and adaptive at the cell level. [Link](2793-brick-based-hierarchical-storage.md)

### Brickmap SDF Volume Rendering
Brickmap SDF uses brick-based storage specifically for Signed Distance Fields. Choose this for SDF rendering where hierarchical LOD is beneficial. SparseGrid is a general-purpose sparse grid for scalar fields. [Link](3310-brickmap-sdf-volume-rendering.md)

### Compaction of Update Information
Compaction optimizes the storage of changes to sparse volumes. SparseGrid supports dynamic add/remove operations, which may benefit from compaction techniques. [Link](3364-compaction-of-update-information.md)

### Tree-based structures (Octree, Pointer Tree, Sparse Voxel DAG)
Tree-based structures recursively subdivide space. They offer hierarchical traversal and efficient empty space skipping. Choose SparseGrid when a flat, adaptive grid with explicit linear indexing is preferred over recursive traversal. [Link](4139-tree-based-structures-octree-pointer-tree-sparse-voxel-dag.md)

### Out-of-Core Rendering with Virtual Page Tables
Out-of-core rendering manages volumes larger than RAM using virtual memory. SparseGrid can be adapted for out-of-core use by paging `cell_indices` and `values` to disk. [Link](5359-out-of-core-rendering-with-virtual-page-tables.md)

## Operating characteristics
-   **Time Complexity:** Adding/removing cells is O(1) per cell on average (using hash set for `active_cell_indices`). Getting points/values is O(N) where N is the number of active cells. Filtering iso-surface is O(N).
-   **Memory Complexity:** O(N) where N is the number of active cells. Each active cell stores 8 scalar values and its linear index. The overhead is minimal compared to dense grids for sparse data.
-   **Measured Performance:** Not given in the sources.
-   **Scaling:** Scales linearly with the number of active cells. Performance is independent of the total volume size defined by `shape`, as long as the number of active cells remains small.
-   **Strengths:** Efficient for large domains with small active regions. Explicit lattice structure allows easy coordinate conversion. Dynamic add/remove operations.
-   **Limitations:** No inherent spatial locality for traversal (unlike trees). Hash set overhead for `active_cell_indices` lookup. Not suitable for dense volumes.
-   **When to use:** Large sparse volumes, dynamic updates, explicit lattice requirements.
-   **When not to use:** Dense volumes, hierarchical traversal needed, multi-resolution requirements.

## Background
SparseGrid is described in the context of GPU-based volume rendering and iso-surface extraction, specifically within the `isoext` project [S1, unofficial]. It is designed to work with marching tetrahedra and dual mesh generation algorithms [S1, unofficial]. The structure inherits from a base `Grid` class and stores data on the device using Thrust [S1, unofficial].

## Sources
[S1] GuangyanCai/isoext, https://github.com/GuangyanCai/isoext, fetched 2026-10-27 (unofficial)
