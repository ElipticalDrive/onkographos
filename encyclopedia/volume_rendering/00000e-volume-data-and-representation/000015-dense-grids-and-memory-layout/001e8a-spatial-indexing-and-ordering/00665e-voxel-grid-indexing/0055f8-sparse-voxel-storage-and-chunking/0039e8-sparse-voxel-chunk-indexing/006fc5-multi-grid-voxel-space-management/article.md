# Multi-Grid Voxel Space Management

Also called: Clunker Voxel Space

## Overview
Multi-Grid Voxel Space Management is a sparse voxel storage mechanism that partitions a large 3D voxel world into fixed-size sub-grids (VoxelGrids) indexed by 3D integer coordinates. It solves the problem of efficiently mapping global voxel coordinates to specific data blocks while supporting dynamic insertion and removal of these blocks. It differs from flat sparse arrays by using a hierarchical dictionary-based lookup for sub-grids, allowing for efficient spatial locality within chunks while maintaining sparse global distribution.

## How it works
**Inputs and Outputs:**
- **Input:** Global voxel coordinates $(x, y, z)$, voxel value, or grid coordinate $(gx, gy, gz)$.
- **Output:** Voxel value, grid reference, or void.

**Data Structures:**
The structure maintains two primary dictionaries and two configuration vectors:
1.  `_grids`: A dictionary mapping `Vector3i` (grid coordinates) to `VoxelGrid` objects.
2.  `_indices`: A dictionary mapping `VoxelGrid` objects back to their `Vector3i` coordinates.
3.  `GridSize`: A `Vector3i` defining the dimensions $(W, H, D)$ of each sub-grid in voxels.
4.  `VoxelSize`: A `float` defining the physical size of a single voxel.

**Numbered Steps:**

1.  **Global to Local Mapping (Get/Set):**
    To access a voxel at global index $I = (i_x, i_y, i_z)$:
    1.  Calculate the grid index $G = (g_x, g_y, g_z)$ using floor division:
        $$g_x = \lfloor i_x / W \rfloor, \quad g_y = \lfloor i_y / H \quad \rfloor, \quad g_z = \lfloor i_z / D \rfloor$$
        *Note: Use `MathF.Floor` on the float division result to handle negative coordinates correctly [S1, unofficial].*
    2.  Calculate the local voxel index $L = (l_x, l_y, l_z)$ within the sub-grid:
        $$l_x = i_x - g_x \cdot W, \quad l_y = i_y - g_y \cdot H, \quad l_z = i_z - g_z \cdot D$$
    3.  Retrieve the grid object $Grid = \text{lookup}(\_grids, G)$.
    4.  If $Grid$ is null, return void/null. Otherwise, return $Grid.\text{Data}[L]$.

2.  **Grid Insertion:**
    To add a new grid at global coordinate $G$:
    1.  Create a new `VoxelGrid` component.
    2.  Set the transform position of the associated GameObject to $G \cdot \text{GridSize} \cdot \text{VoxelSize}$.
    3.  Add the GameObject as a child to the parent scene hierarchy.
    4.  Insert the mapping into `_grids[G] = Grid` and `_indices[Grid] = G`.
    5.  Subscribe to the `Grid.VoxelsChanged` event if the grid is alive.
    6.  Invoke the `GridAdded` event.

3.  **Grid Removal:**
    To remove a grid at global coordinate $G$:
    1.  Check if $G$ exists in `_grids`.
    2.  Retrieve $Grid = \_grids[G]$.
    3.  Remove $G$ from `_grids` and $Grid$ from `_indices`.
    4.  Unsubscribe from `Grid.VoxelsChanged`.
    5.  Remove the grid's GameObject from the scene hierarchy.
    6.  Invoke the `GridRemoved` event.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class MultiGridVoxelSpace:
    def __init__(self, grid_size: Vector3i, voxel_size: float):
        self.grid_size = grid_size  # W, H, D
        self.voxel_size = voxel_size
        self._grids = {}  # Dict[Vector3i, VoxelGrid]
        self._indices = {}  # Dict[VoxelGrid, Vector3i]

    def get_voxel(self, index: Vector3i) -> Optional[float]:
        # Step 1: Calculate grid index using floor division for negative coords
        gx = math.floor(index.x / self.grid_size.x)
        gy = math.floor(index.y / self.grid_size.y)
        gz = math.floor(index.z / self.grid_size.z)
        grids_index = Vector3i(gx, gy, gz)

        # Step 2: Calculate local voxel index
        local_x = index.x - gx * self.grid_size.x
        local_y = index.y - gy * self.grid_size.y
        local_z = index.z - gz * self.grid_size.z
        local_index = Vector3i(local_x, local_y, local_z)

        # Step 3: Retrieve grid
        grid = self._grids.get(grids_index)
        if grid is None:
            return None

        # Step 4: Return voxel data
        return grid.data[local_index]

    def set_voxel(self, index: Vector3i, value: float):
        gx = math.floor(index.x / self.grid_size.x)
        gy = math.floor(index.y / self.grid_size.y)
        gz = math.floor(index.z / self.grid_size.z)
        grids_index = Vector3i(gx, gy, gz)

        local_x = index.x - gx * self.grid_size.x
        local_y = index.y - gy * self.grid_size.y
        local_z = index.z - gz * self.grid_size.z
        local_index = Vector3i(local_x, local_y, local_z)

        grid = self._grids.get(grids_index)
        if grid is not None:
            grid.set_voxel(local_index, value)

    def add_grid(self, index: Vector3i):
        # Create new grid component
        grid = VoxelGrid()
        
        # Set transform position
        # Position = index * GridSize * VoxelSize
        pos_x = index.x * self.grid_size.x * self.voxel_size
        pos_y = index.y * self.grid_size.y * self.voxel_size
        pos_z = index.z * self.grid_size.z * self.voxel_size
        grid.transform.position = Vector3(pos_x, pos_y, pos_z)

        # Add to scene hierarchy (child)
        scene.add_child(grid.game_object)

        # Update dictionaries
        self._grids[index] = grid
        self._indices[grid] = index

        # Subscribe to events
        grid.on_voxels_changed += self.handle_voxels_changed

        # Invoke event
        self.invoke_grid_added_event()

    def remove_grid(self, index: Vector3i):
        if index not in self._grids:
            return

        grid = self._grids[index]

        # Remove from dictionaries
        del self._grids[index]
        del self._indices[grid]

        # Unsubscribe
        grid.on_voxels_changed -= self.handle_voxels_changed

        # Remove from scene
        scene.remove_child(grid.game_object)

        # Invoke event
        self.invoke_grid_removed_event()
```

**Parameters:**
- `GridSize`: Fixed dimensions of each sub-grid. Not given in the sources.
- `VoxelSize`: Physical size of a voxel. Not given in the sources.

**Edge Cases and Pitfalls:**
- **Negative Coordinates:** Integer division truncates toward zero in many languages, but grid indices must floor toward negative infinity to correctly map negative global coordinates to their respective sub-grids. The implementation explicitly uses `MathF.Floor` on the float division result [S1, unofficial].
- **Null Grids:** Accessing a voxel in a non-existent grid returns null/void rather than throwing an exception.

## Subtopics

### [GPU Accelerated Sparse Grid Construction](9812-gpu-accelerated-sparse-grid-construction.md)
Optimizes the creation of sparse grids on the GPU. Choose when grid population is the bottleneck and data is generated procedurally on the GPU.

### [IndexGrid](9819-indexgrid.md)
An alternative indexing structure. Choose when a different trade-off between memory overhead and lookup speed is required.

## Operating characteristics
- **Time Complexity:** Lookup and insertion are $O(1)$ average case due to dictionary hashing, assuming constant-time hash operations for `Vector3i`.
- **Memory Complexity:** Proportional to the number of active sub-grids plus the overhead of two dictionary entries per grid.
- **Strengths:** Efficient sparse storage; supports dynamic addition/removal of chunks; simple coordinate mapping.
- **Limitations:** Dictionary overhead per grid; potential cache misses due to scattered grid objects in memory.
- **When to use:** Large voxel worlds with sparse occupancy where dynamic chunk management is needed.
- **When not to use:** Dense voxel grids where a flat array is more efficient; scenarios requiring strict memory locality across chunks.

## Background
Originated in the Clunker game engine project, specifically within the `Clunker` namespace. It is part of the engine's voxel rendering and world management system. The implementation is documented in the source code of the Wrecker project.

## Sources
[S1] generatives/Wrecker, https://github.com/generatives/Wrecker, fetched 2026-10-27 (unofficial)
