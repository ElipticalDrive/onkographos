# OpenVDB Volume Grid Storage

Also called: OpenVDB Grid, VDB Grid, Tile-based Sparse Voxel Storage

## Overview
OpenVDB Volume Grid Storage is a data structure for efficiently storing dense and sparse volume grids as groups of voxels indexed by 3D integer coordinates [S1, unofficial]. It solves the memory inefficiency of storing individual values for every voxel in large grids by using tiling and active status flags [S1, unofficial]. It sets itself apart from sibling methods by explicitly combining hierarchical tiling (where uniform blocks are stored once) with per-voxel active status, allowing for complex topologies that can be merged or preserved during operations [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A set of voxel values, a grid transform, and a background value.
- Output: A sparse grid structure containing active tiles/voxels and their associated data.

**Data Structures:**
The grid consists of the following components:
1.  **Grid Transform:** A matrix (location, rotation, scale) mapping index-space to object-space [S1, unofficial].
2.  **Background Value:** A default scalar value for inactive regions [S1, unofficial].
3.  **Topology:** Defined by the combination of tiling and active status [S1, unofficial].
4.  **Tiles:** Blocks of $8 \times 8 \times 8$ voxels (512 voxels) [S1, unofficial]. Higher-level tiles (e.g., $16 \times 16 \times 16$ small tiles) are less common but supported [S1, unofficial].
5.  **Active Status:** A flag for each voxel and tile. Active voxels/tiles store data; inactive ones return the background value [S1, unofficial].

**Coordinate System:**
- Voxels are indexed by 3D integer coordinates $(x, y, z)$ [S1, unofficial].
- The coordinate $(0, 0, 0)$ is the center voxel [S1, unofficial].
- The coordinate $(1, 0, 0)$ is the voxel to the right of the center [S1, unofficial].

**Transform Mapping:**
The object-space position $P_{obj}$ is calculated from the index-space coordinate $I$ and the grid transform $T$:
$$ P_{obj} = T \cdot I $$
For example, if the grid scale is $s = 0.1$, the voxel $(5, 0, 0)$ corresponds to object space position $(0.5, 0, 0)$ [S1, unofficial].

**Algorithm Steps:**
1.  **Initialize Grid:** Set the grid transform $T$ and background value $B$ [S1, unofficial].
2.  **Assign Voxels:** For each voxel at index $I$, determine its tile membership. A tile of size $8 \times 8 \times 8$ covers indices where $\lfloor x/8 \rfloor, \lfloor y/8 \rfloor, \lfloor z/8 \rfloor$ are constant [S1, unofficial].
3.  **Check Uniformity:** If all 512 voxels in a tile have the same value $V$, mark the tile as uniform and store $V$ once for the entire tile [S1, unofficial].
4.  **Set Active Status:** Mark voxels/tiles as active if they hold non-background data or are explicitly set to be active [S1, unofficial]. Inactive voxels/tiles are skipped during most operations [S1, unofficial].
5.  **Merge Topologies:** When combining grids (e.g., addition), merge the topologies of input grids, preserving active status and tiling structure [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class OpenVDBGrid:
    def __init__(self, transform, background_value):
        self.transform = transform  # Matrix mapping index to object space
        self.background_value = background_value
        self.tiles = {}  # Dictionary mapping tile coordinates to data

    def set_voxel(self, x, y, z, value):
        # Calculate tile coordinates (8x8x8 block)
        tile_x = x // 8
        tile_y = y // 8
        tile_z = z // 8
        tile_key = (tile_x, tile_y, tile_z)
        
        # Initialize tile if not exists
        if tile_key not in self.tiles:
            self.tiles[tile_key] = {
                'uniform': False,
                'value': None,
                'voxels': {}  # Stores individual voxel values if not uniform
            }
        
        tile = self.tiles[tile_key]
        
        # If tile is already uniform, check if new value matches
        if tile['uniform']:
            if tile['value'] == value:
                return  # No change needed
            else:
                # Break uniformity: convert to individual voxel storage
                tile['uniform'] = False
                tile['value'] = None
                # Copy existing uniform value to all voxels in tile
                for dx in range(8):
                    for dy in range(8):
                        for dz in range(8):
                            tile['voxels'][(dx, dy, dz)] = tile['background_value'] # Placeholder for actual background
        
        # Set individual voxel value
        local_x = x % 8
        local_y = y % 8
        local_z = z % 8
        tile['voxels'][(local_x, local_y, local_z)] = value
        
        # Check if tile has become uniform again
        if len(tile['voxels']) == 512:
            first_value = next(iter(tile['voxels'].values()))
            if all(v == first_value for v in tile['voxels'].values()):
                tile['uniform'] = True
                tile['value'] = first_value
                tile['voxels'] = {}  # Clear individual storage

    def get_voxel(self, x, y, z):
        tile_x = x // 8
        tile_y = y // 8
        tile_z = z // 8
        tile_key = (tile_x, tile_y, tile_z)
        
        if tile_key not in self.tiles:
            return self.background_value
        
        tile = self.tiles[tile_key]
        
        if tile['uniform']:
            return tile['value']
        else:
            local_x = x % 8
            local_y = y % 8
            local_z = z % 8
            return tile['voxels'].get((local_x, local_y, local_z), self.background_value)

    def get_object_space_position(self, x, y, z):
        # Apply transform to index coordinates
        index_vector = [x, y, z, 1]  # Homogeneous coordinate
        # Assuming transform is a 4x4 matrix
        object_position = [
            sum(self.transform[i][j] * index_vector[j] for j in range(4))
            for i in range(4)
        ]
        return object_position[:3]  # Return x, y, z
```

**Parameters:**
- Tile size: $8 \times 8 \times 8$ voxels (512 voxels) [S1, unofficial].
- Higher-level tiles: $16 \times 16 \times 16$ small tiles (less common) [S1, unofficial].
- Background value: User-defined, default often 0 [S1, unofficial].

**Edge Cases and Pitfalls:**
- Memory efficiency: Storing individual values for every voxel is impractical for large grids [S1, unofficial].
- Topology preservation: Most operations preserve the topology of the input grid; merging grids requires careful handling of active status [S1, unofficial].
- SDF grids: Negative values indicate inside, positive outside, zero at surface [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Accessing a voxel is $O(1)$ after tile lookup. Tile lookup is $O(1)$ with hash map. Setting/getting uniform tiles is $O(1)$. Breaking uniformity is $O(512)$ to copy values.
- **Memory Complexity:** Proportional to the number of active tiles and non-uniform voxels. Uniform tiles store one value per 512 voxels. Inactive regions store nothing [S1, unofficial].
- **Performance:** Efficient for sparse volumes with large uniform regions. Less efficient for highly detailed, non-uniform volumes where tiling benefits are minimal [S1, unofficial].
- **Strengths:** Memory efficiency for sparse data, support for complex topologies, easy boolean operations for SDFs [S1, unofficial].
- **Limitations:** Not efficient for storing detailed shapes compared to meshes; overhead for managing tiles and active status [S1, unofficial].
- **When to use:** Sparse volumes, SDFs requiring boolean operations, volumes with large uniform regions [S1, unofficial].
- **When not to use:** Dense, highly detailed volumes without uniform regions, where mesh representation is more suitable [S1, unofficial].

## Background
OpenVDB is an industry-standard library for storing and processing volumetric data [S1, unofficial]. It was developed to address the memory challenges of storing large volume grids [S1, unofficial]. It is widely used in visual effects and animation for storing SDFs, fluid simulations, and other volumetric data [S1, unofficial]. The integration into tools like Blender's Geometry Nodes has made it accessible for general volume modeling [S1, unofficial].

## Sources
[S1] Volume Grids in Geometry Nodes — Blender Developers Blog, https://code.blender.org/2025/10/volume-grids-in-geometry-nodes/, fetched 2026-10-27 (unofficial)
