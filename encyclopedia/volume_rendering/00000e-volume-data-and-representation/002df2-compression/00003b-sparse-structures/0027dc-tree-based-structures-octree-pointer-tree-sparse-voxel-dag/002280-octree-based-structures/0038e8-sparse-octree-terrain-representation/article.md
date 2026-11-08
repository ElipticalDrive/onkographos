# Sparse Octree Terrain Representation

Also called: Sparse Octree Reconstruction Model, Sparse Octree Terrain

## Overview
Sparse Octree Terrain Representation replaces dense heightmap diffusion with a sparse octree structure that stores only occupied voxels, aligning with the Voxy schema to avoid wasting compute on empty cells [S1, unofficial]. It differs from its siblings by operating on a fixed $8 \times 8$ local patch with quantized Y-axis levels rather than the full $16 \times 16$ diffuse grid used in previous pipelines, ensuring that traversal and rendering work only over the actual terrain surface [S1, unofficial].

## How it works
**Inputs and Outputs**
- **Input:** A chunk coordinate $(chunkX, chunkZ)$, an $8 \times 8$ heightmap (`heightmap8x8`), biome data (`biomes64`), and a Level of Detail (LOD) integer.
- **Output:** A `TerrainPatch` object containing the sparse voxel set and the octree structure.

**Data Structures**
1.  **Voxel**: An immutable value class representing a single occupied voxel at $(x, y, z)$ in octree-local space. It implements `equals` and `hashCode` to serve as a key in a Set [S1, unofficial].
2.  **TerrainPatch**: Contains two fields computed from the heightmap:
    -   `Set<Voxel> occupiedVoxels`: An unmodifiable set of terrain positions [S1, unofficial].
    -   `int[][][] octreeStructure`: An $8 \times 8 \times 8$ binary occupancy grid indexed by `[x][y][z]`. The Y-axis quantizes the height range into `OCTREE_SIZE` levels [S1, unofficial].

**Parameters**
-   `OCTREE_SIZE`: A constant integer with value $8$. It defines the dimension of the binary occupancy grid and the number of Y-axis quantization levels [S1, unofficial].

**Algorithm Steps**
1.  **Validation**: Verify that the input `heightmap8x8` has dimensions exactly $8 \times 8$. Reject input if dimensions differ [S1, unofficial].
2.  **Smoothing**: Apply smoothing passes to the heightmap. The number of passes is calculated as:
    $$ \text{passes} = \text{OCTREE\_SIZE} - \text{lod} $$
    -   LOD $0$ results in the most detail (7 passes).
    -   LOD $\ge 8$ results in no smoothing (0 or negative passes, treated as no-op) [S1, unofficial].
3.  **Quantization and Occupancy**:
    -   Map the smoothed height values to the Y-axis quantization levels ($0$ to $7$).
    -   Populate the `octreeStructure` grid. For each $(x, z)$ in the $8 \times 8$ grid, determine the occupied Y-levels based on the quantized height.
    -   **Invariant**: The lowest Y-level ($0$) in the octree structure must always be occupied, even in the edge case of a flat heightmap [S1, unofficial].
4.  **Voxel Set Construction**: Iterate over the `octreeStructure`. For every cell where `octreeStructure[x][y][z] == 1`, create a `Voxel(x, y, z)` and add it to `occupiedVoxels`.
5.  **Return**: Construct and return the `TerrainPatch` with the populated `occupiedVoxels` and `octreeStructure` [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_sparse_octree(chunk_x, chunk_z, heightmap_8x8, biomes_64, lod):
    # 1. Validation
    if len(heightmap_8x8) != 8 or any(len(row) != 8 for row in heightmap_8x8):
        raise ValueError("Heightmap must be 8x8")

    octree_size = 8
    
    # 2. Smoothing
    # Apply smoothing passes. If lod >= octree_size, passes <= 0, so no smoothing.
    passes = max(0, octree_size - lod)
    smoothed_heightmap = apply_smoothing(heightmap_8x8, passes)

    # 3. Quantization and Grid Population
    # Initialize 8x8x8 grid with 0
    octree_structure = [[[0 for _ in range(octree_size)] for _ in range(octree_size)] for _ in range(octree_size)]
    occupied_voxels = set()

    for x in range(octree_size):
        for z in range(octree_size):
            height_val = smoothed_heightmap[x][z]
            # Quantize height to Y-level [0, 7]
            # Note: Exact quantization formula depends on heightmap range normalization.
            # Assuming heightmap values are normalized or mapped to 0-7 range.
            y_level = quantize_height(height_val, octree_size)
            
            # Invariant: Lowest Y-level (0) must always be occupied
            octree_structure[x][0][z] = 1
            occupied_voxels.add(Voxel(x, 0, z))
            
            # Mark the quantized level as occupied
            if 0 <= y_level < octree_size:
                octree_structure[x][y_level][z] = 1
                occupied_voxels.add(Voxel(x, y_level, z))

    # 4. Return TerrainPatch
    return TerrainPatch(occupied_voxels, octree_structure)

class Voxel:
    def __init__(self, x, y, z):
        self.x = x
        self.y = y
        self.z = z
    
    def __eq__(self, other):
        return isinstance(other, Voxel) and self.x == other.x and self.y == other.y and self.z == other.z
    
    def __hash__(self):
        return hash((self.x, self.y, self.z))

class TerrainPatch:
    def __init__(self, occupied_voxels, octree_structure):
        self.occupied_voxels = frozenset(occupied_voxels) # Unmodifiable
        self.octree_structure = octree_structure
    
    def get_occupied_voxels(self):
        return self.occupied_voxels
    
    def get_octree_structure(self):
        # Defensive copy
        return [[[cell for cell in row] for row in plane] for plane in self.octree_structure]
```

**Edge Cases and Pitfalls**
-   **Flat Heightmap**: The lowest Y-level ($0$) must always be occupied. If the heightmap is flat, the algorithm must still mark $y=0$ as occupied to maintain structural integrity [S1, unofficial].
-   **LOD Bounds**: LOD values $\ge 8$ result in zero smoothing passes. The implementation should handle this gracefully as a no-op [S1, unofficial].
-   **Input Dimensions**: The method strictly requires an $8 \times 8$ heightmap. Using the previous $16 \times 16$ input will cause validation failure [S1, unofficial].

## Operating characteristics
-   **Time Complexity**: $O(1)$ per chunk, as the grid size is fixed at $8 \times 8 \times 8$ ($512$ cells). Smoothing complexity depends on the kernel size and number of passes, but remains constant relative to chunk count.
-   **Memory Complexity**: $O(1)$ per chunk. The `octree_structure` uses $8 \times 8 \times 8$ integers (or bits if packed), and `occupiedVoxels` stores only non-empty voxels.
-   **Performance**: Significantly reduces compute waste compared to dense $16 \times 16$ diffusion, which processes $256$ values regardless of terrain presence. This method works only over the occupied $8 \times 8$ patch surface [S1, unofficial].
-   **Strengths**: Aligns with Voxy schema; avoids empty cell processing; backward-compatible with existing constructors [S1, unofficial].
-   **Limitations**: Fixed resolution per patch ($8 \times 8$); requires careful quantization of height values to Y-levels [S1, unofficial].

## Background
This method was introduced in the `LODiffusion` project to replace the existing LOD pipeline that forced dense $16 \times 16$ heightmap diffusion for every chunk. The change was part of a pull request (PR #17) by `rhythmatician` to align the data contract with Voxy's octree storage schema [S1, unofficial]. It adds a `SparseOctree` representation to `TerrainPatch` and a matching generation path in `DiffusionChunkGenerator` [S1, unofficial].

## Sources
[S1] feat: sparse octree terrain representation aligned with Voxy schema, https://github.com/rhythmatician/LODiffusion/pull/17, fetched 2026-10-26 (unofficial)
