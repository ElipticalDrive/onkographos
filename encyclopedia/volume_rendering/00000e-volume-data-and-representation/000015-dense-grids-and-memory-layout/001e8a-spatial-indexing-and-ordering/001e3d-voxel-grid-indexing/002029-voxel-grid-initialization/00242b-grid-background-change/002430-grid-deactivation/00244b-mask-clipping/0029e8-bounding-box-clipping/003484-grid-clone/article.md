# Grid Clone

Also called: PixelGrid2D Clone

## Overview
Grid Clone creates a deep copy of a source voxel grid, preserving its spatial metadata and all active voxel values. It solves the problem of needing an independent instance of a grid for modification without affecting the original data. Unlike [Bounding Box Clipping](4643-bounding-box-clipping.md), which filters voxels by spatial extent, Grid Clone retains the full logical structure of the source grid.

## How it works
**Inputs:** A source grid object (e.g., `PixelGrid2D` or `ScalarGrid3D`) containing a bounding box, pixel size, and an array of voxel values.
**Outputs:** A new grid instance with identical metadata and a deep copy of the voxel data.

**Data Structures:**
The source grid typically inherits from a base `Grid3D` class. Key fields include:
- `BoundingBox`: Defines the spatial extent (e.g., `Box` in Rhino.Geometry).
- `PixelSize` (or `PixelDimensions`): A 3D vector defining the physical size of a voxel in X, Y, and Z.
- `Grid`: A 1D array (e.g., `float[]`) storing voxel values. The size is determined by the product of the dimensions derived from the bounding box and pixel size.

**Steps:**
1. **Initialize Metadata:** Create a new grid instance. Copy the `BoundingBox` and `PixelSize` from the source grid to the new instance. This ensures the new grid occupies the same physical space and has the same resolution.
2. **Allocate Data:** Allocate a new array for the voxel values. The size of this array must match the source grid's data size. In implementations like `ScalarGrid3D`, this is calculated as `SizeUVW.SelfProduct()`, which is the total number of voxels ($N_x \times N_y \times N_z$).
3. **Copy Values:** Iterate through the source grid's data array from index $0$ to $Count - 1$. Assign each value to the corresponding index in the new grid's data array.
   $$ \text{newGrid.Grid}[i] = \text{sourceGrid.Grid}[i] \quad \forall i \in [0, \text{Count}) $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def grid_clone(source_grid):
    # Step 1: Initialize new grid with source metadata
    # Assuming a generic Grid class with constructor taking bbox and pixel_size
    new_grid = Grid(
        bounding_box=source_grid.bounding_box,
        pixel_size=source_grid.pixel_size
    )
    
    # Step 2: Allocate data array
    # Size is the total number of voxels
    count = source_grid.count
    new_grid.grid = [0.0] * count
    
    # Step 3: Copy values
    for i in range(count):
        new_grid.grid[i] = source_grid.grid[i]
        
    return new_grid
```

**Edge Cases:**
- **Empty Grids:** If the source grid has zero voxels, the new grid should also have an empty data array but retain valid metadata.
- **Precision:** Floating-point values are copied bit-for-bit in standard implementations, preserving precision.

## Subtopics

### PixelGrid2D CloneEmpty
Creates a new `PixelGrid2D` object initialized with the current bounding box and pixel size, but without copying any voxel data. This is useful for creating a blank canvas with specific spatial parameters. It differs from `Grid Clone` by skipping the data copy step, resulting in a grid with uninitialized or default values. Use this when you need a grid structure for future population rather than a copy of existing data.

## Operating characteristics
**Time Complexity:** $O(N)$, where $N$ is the total number of voxels in the grid. The operation requires a single pass to copy data.
**Memory Complexity:** $O(N)$, as a new array of the same size is allocated.
**Performance:** Linear scaling with grid resolution. For large grids, this can be memory-intensive and slow due to the allocation and copy overhead.
**Strengths:** Simple, reliable deep copy; preserves all metadata.
**Limitations:** High memory usage for large volumes; does not compress or optimize data.

## Background
This method is part of the VoxelTools library, an unofficial C# implementation for voxel manipulation. It provides basic grid operations for 3D scalar and pixel grids. The implementation is found in the `voxeltool` repository.

## Sources
[S1] voxeltool, https://github.com/arendvw/voxeltools, fetched 2026-10-29 (unofficial)
