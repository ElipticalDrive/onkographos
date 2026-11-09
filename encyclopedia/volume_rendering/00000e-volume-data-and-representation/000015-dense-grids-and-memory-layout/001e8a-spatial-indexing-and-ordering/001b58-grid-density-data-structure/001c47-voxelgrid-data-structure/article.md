# VoxelGrid Data Structure

Also called: VoxelGrid3D, VoxelGrid3D Data Structure

## Overview
The VoxelGrid Data Structure is a dense 3D grid representation that maps continuous world coordinates to discrete voxel indices within a defined bounding box [S2, unofficial]. It stores per-voxel data such as occupancy bits or label histograms, differing from sparse alternatives by guaranteeing $O(1)$ index-based access at the cost of memory proportional to the bounding box volume [S1, unofficial][S2, unofficial].

## How it works
### Inputs and Outputs
- **Input**: A 3D bounding box (defined by origin and extents), a voxel resolution (size in world units per axis), and optional initial data (e.g., a pre-allocated BitArray or label histogram vectors) [S1, unofficial][S2, unofficial].
- **Output**: A data structure providing $O(1)$ read/write access to voxel data via integer indices $(i, j, k)$ or world coordinates $(x, y, z)$ [S1, unofficial][S2, unofficial].

### Data Structures
The structure consists of global grid parameters and a contiguous storage array for voxel data [S1, unofficial][S2, unofficial].

**Global Parameters:**
- `resolution_` (float): The physical size of one voxel along each axis [S1, unofficial].
- `sizex_`, `sizey_`, `sizez_` (uint32_t): The number of voxels along the X, Y, and Z axes [S1, unofficial].
- `offset_` (Vector4f or Point3d): The world coordinate of the grid origin (min corner) [S1, unofficial][S2, unofficial].
- `BBox` (Box): The bounding box defining the spatial extent [S2, unofficial].

**Voxel Storage:**
Two distinct layouts are observed in the sources:
1.  **Occupancy BitArray ([S2, unofficial])**:
    -   `Grid`: A `BitArray` or `TCollection` storing a single boolean value per voxel.
    -   Size: `SizeUVW.SelfProduct()` (i.e., $sizex\_ \times sizey\_ \times sizez\_$) [S2, unofficial].
    -   Memory: 1 bit per voxel.
2.  **Label Histogram ([S1, unofficial])**:
    -   `voxels_`: A `std::vector<Voxel>` [S1, unofficial].
    -   `Voxel` struct:
        -   `labels`: `std::map<uint32_t, uint32_t>` mapping label IDs to counts.
        -   `count`: `uint32_t` total occupancy count.
    -   Auxiliary arrays (optional, for optimization):
        -   `occupied_`: `std::vector<uint32_t>` indices of occupied voxels.
        -   `occlusions_`, `invalid_`, `occludedBy_`: `std::vector<int32_t>` for visibility tracking.
        -   `occlusionsValid_`: `bool` flag.

### Numbered Steps
1.  **Initialization**: The constructor accepts a bounding box and voxel dimensions to initialize grid parameters [S2, unofficial]. If a BitArray is provided, it is injected directly without copying; otherwise, a new BitArray of size `SizeUVW.SelfProduct()` is created and initialized to false [S2, unofficial]. For label histograms, a vector of `Voxel` structs is allocated [S1, unofficial].
2.  **World-to-Index**: Given a world point $P = (pos\_x, pos\_y, pos\_z)$, compute the voxel index $(i, j, k)$ using $i = (pos\_x - offset\_[0]) / resolution\_$, $j = (pos\_y - offset\_[1]) / resolution\_$, $k = (pos\_z - offset\_[2]) / resolution\_$ [S1, unofficial]. [G] Apply truncation or floor and clamp indices to $[0, sizex\_ - 1]$, $[0, sizey\_ - 1]$, $[0, sizez\_ - 1]$ to handle boundary precision.
3.  **Index-to-World**: Given voxel indices $(i, j, k)$, compute the center of the voxel in world coordinates using $pos\_x = offset\_[0] + i \times resolution\_ + 0.5 \times resolution\_$, $pos\_y = offset\_[1] + j \times resolution\_ + 0.5 \times resolution\_$, $pos\_z = offset\_[2] + k \times resolution\_ + 0.5 \times resolution\_$ [S1, unofficial].
4.  **Linear Index**: Map 3D indices $(i, j, k)$ to a 1D array index using $index(i, j, k) = i + j \times sizex\_ + k \times sizex\_ \times sizey\_$ [S1, unofficial].
5.  **Bounds Check**: Verify if a sub-point $(i, j, k)$ is within grid bounds by checking $i \ge 0$, $j \ge 0$, $k \ge 0$ and $i < sizex\_$, $j < sizey\_$, $k < sizez\_$ [S2, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class VoxelGrid:
    def __init__(self, offset, resolution, sizex, sizey, sizez, bit_array=None):
        """
        Initialize VoxelGrid.
        offset: (x, y, z) world coordinates of the origin.
        resolution: float, size of one voxel.
        sizex, sizey, sizez: int, grid dimensions.
        bit_array: optional list of bools to inject without copying.
        """
        self.offset = offset
        self.resolution = resolution
        self.sizex = sizex
        self.sizey = sizey
        self.sizez = sizez
        self.total_voxels = sizex * sizey * sizez
        
        if bit_array is not None:
            self.grid = bit_array
        else:
            self.grid = [False] * self.total_voxels

    def world_to_index(self, x, y, z):
        """Convert world coordinates to 3D voxel index (i, j, k)."""
        i = int((x - self.offset[0]) / self.resolution)
        j = int((y - self.offset[1]) / self.resolution)
        k = int((z - self.offset[2]) / self.resolution)
        return i, j, k

    def index_to_world(self, i, j, k):
        """Convert 3D voxel index to world coordinates (center of voxel)."""
        x = self.offset[0] + i * self.resolution + 0.5 * self.resolution
        y = self.offset[1] + j * self.resolution + 0.5 * self.resolution
        z = self.offset[2] + k * self.resolution + 0.5 * self.resolution
        return x, y, z

    def linear_index(self, i, j, k):
        """Convert 3D index to 1D array index."""
        return i + j * self.sizex + k * self.sizex * self.sizey

    def point_in_grid(self, i, j, k):
        """Check if indices are within grid bounds."""
        return (i >= 0 and j >= 0 and k >= 0 and
                i < self.sizex and j < self.sizey and k < self.sizez)

    def clone(self):
        """Create a deep copy of the grid."""
        new_grid = VoxelGrid.__new__(VoxelGrid)
        new_grid.offset = self.offset
        new_grid.resolution = self.resolution
        new_grid.sizex = self.sizex
        new_grid.sizey = self.sizey
        new_grid.sizez = self.sizez
        new_grid.total_voxels = self.total_voxels
        new_grid.grid = [False] * self.total_voxels
        for idx in range(self.total_voxels):
            if self.grid[idx]:
                new_grid.grid[idx] = True
        return new_grid

    def clone_empty(self):
        """Create a new grid with the same dimensions but empty storage."""
        return VoxelGrid(self.offset, self.resolution, self.sizex, self.sizey, self.sizez)
```

### Parameters and Edge Cases
- **Resolution**: Must be a positive float [S1, unofficial].
- **Bounding Box**: Defines the spatial extent of the grid [S2, unofficial].
- **Precision**: Floating-point division in `world_to_index` can suffer from precision errors near boundaries [G]. It is recommended to clamp indices to `[0, size-1]`.
- **Memory**: Scales linearly with the total number of voxels $N = sizex \times sizey \times sizez$ [S2, unofficial].

## Operating characteristics
- **Time Complexity**:
    - Access/Update: $O(1)$ for index-based access [S1, unofficial][S2, unofficial].
    - World-to-Index: $O(1)$ (3 subtractions, 3 divisions) [S1, unofficial].
    - Clone: $O(N)$ where $N$ is the number of voxels [S2, unofficial].
- **Memory Complexity**: $O(N)$ where $N = sizex \times sizey \times sizez$ [S2, unofficial].
    - BitArray layout: $N$ bits [S2, unofficial].
    - Label Histogram layout: $N \times (\text{map overhead} + \text{counts})$ [S1, unofficial].
- **Strengths**: Simple dense layout, $O(1)$ random access, cache-friendly for dense data.
- **Limitations**: Memory usage grows cubically with resolution, inefficient for sparse scenes.
- **When to use**: Dense volumes, small to medium bounding boxes, or when frequent random access is required.
- **When not to use**: Large-scale outdoor scenes or highly sparse data.

## Background
Not given in the sources.

## Sources
[S1] jbehley/voxelizer, https://github.com/jbehley/voxelizer, fetched 2026-10-27 (unofficial)
[S2] voxeltool, https://github.com/arendvw/voxeltools, fetched 2026-10-29 (unofficial)
