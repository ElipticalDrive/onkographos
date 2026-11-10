# Voxel World Data Structure
Also called: Hash Grid for Voxel World, HashSet of IVec3
## Overview
The Voxel World Data Structure is a sparse representation that stores voxel existence using a hash set of integer-based 3D vectors (`IVec3`) [S1, unofficial]. It solves the problem of efficiently tracking occupied grid positions in a voxel world while ensuring strict grid alignment [S1, unofficial]. It builds on [Hash Grids](936-hash-grids.md) by applying hash-based lookup to discrete integer coordinates rather than continuous floating-point positions [S1, unofficial].
## How it works
**Inputs and Outputs:**
- Input: A set of 3D integer coordinates representing occupied voxels.
- Output: A data structure allowing insertion, deletion, and existence checks for voxel positions [S1, unofficial].

**Data Structures:**
The core structure is a `World` object containing a single field:
- `voxels`: A hash set of `IVec3` (integer 3D vectors) [S1, unofficial]. The `IVec3` type ensures that coordinates are aligned to the integer grid [S1, unofficial].

**Steps:**
1. **Initialization**: Create an empty `World` instance with an empty hash set of `IVec3` [S1, unofficial].
2. **Population**: For each voxel to be added, convert its position to an `IVec3` and insert it into the set [S1, unofficial].
3. **Rendering**: Iterate over all elements in the `voxels` set and draw a cube at each stored `IVec3` position [S1, unofficial].
4. **Runtime Modification**: To toggle a voxel at the camera's position:
   - Convert the camera's float position to an `IVec3` [S1, unofficial].
   - Check if this `IVec3` exists in the set [S1, unofficial].
   - If it exists, remove it; otherwise, insert it [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class IVec3:
    def __init__(self, x: int, y: int, z: int):
        self.x = x
        self.y = y
        self.z = z
    def __hash__(self):
        return hash((self.x, self.y, self.z))
    def __eq__(self, other):
        return self.x == other.x and self.y == other.y and self.z == other.z

class World:
    def __init__(self):
        self.voxels = set()

    def add_voxel(self, x: int, y: int, z: int):
        self.voxels.add(IVec3(x, y, z))

    def remove_voxel(self, x: int, y: int, z: int):
        self.voxels.discard(IVec3(x, y, z))

    def toggle_voxel_at_camera(self, camera_position: tuple):
        # Convert float position to integer grid-aligned position [S1, unofficial]
        voxel_pos = IVec3(int(camera_position[0]), int(camera_position[1]), int(camera_position[2]))
        
        if voxel_pos in self.voxels:
            self.voxels.discard(voxel_pos)
        else:
            self.voxels.add(voxel_pos)

    def render(self):
        # Iterate over all stored voxel positions [S1, unofficial]
        for voxel_pos in self.voxels:
            draw_cube(voxel_pos)

def draw_cube(pos: IVec3):
    # Source does not specify mesh generation or GPU upload logic [G]
    pass

# Example Initialization
world = World()
for z in range(32):
    for x in range(32):
        world.add_voxel(x, 0, z)
```

**Parameters:**
- Grid Alignment: Coordinates must be integers (`IVec3`) to ensure voxels are aligned to the grid [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Performance**: Naively drawing a cube for every voxel without instancing leads to significant performance degradation as voxel count increases, due to excessive vertex uploads to the GPU [S1, unofficial].
- **Grid Alignment**: Using floating-point vectors instead of `IVec3` prevents strict grid alignment [S1, unofficial].

## Subtopics

### [Multi-type Voxel Storage](1104-multi-type-voxel-storage.md)
Multi-type Voxel Storage is a sparse voxel data structure that maps 3D integer coordinates to discrete material identifiers (kinds) using a hash map, enabling the representation of heterogeneous voxel worlds where different blocks have distinct properties. It sets itself apart from its sibling [Hash-Map Voxel Storage](5782-hash-map-voxel-storage.md) by explicitly associating a `VoxelKind` enum with each occupied entry rather than storing a single generic type or binary existence flag, and from [Brick-based Sparse Voxel Lookup with Open-Addressing Hash](2587-brick-based-sparse-voxel-lookup-with-open-addressing-hash.md) by using a flat hash map per chunk rather than a brick-based open-addressing scheme.

### [Brick-based Sparse Voxel Lookup with Open-Addressing Hash](2587-brick-based-sparse-voxel-lookup-with-open-addressing-hash.md)
A GPU-friendly sparse voxel storage and lookup scheme that groups active voxels into 8x8x8 bricks, stores brick occupancy as 512-bit bitsets, and maps brick coordinates to compact voxel/QEF arrays via an open-addressing hash table.

### [Hash-Map Voxel Storage](5782-hash-map-voxel-storage.md)
Hash-Map Voxel Storage is a sparse voxel representation that maps 3D integer coordinates to voxel attributes (such as color or density) using a hash table, building on the grid-alignment guarantees of [Voxel World Data Structure](1101-voxel-world-data-structure.md). It differs from sibling structures like [Multi-type Voxel Storage](1104-multi-type-voxel-storage.md) by focusing on a single unified attribute map per coordinate, and from [Brick-based Sparse Voxel Lookup with Open-Addressing Hash](2587-brick-based-sparse-voxel-lookup-with-open-addressing-hash.md) by storing individual voxels rather than compressed brick tiles. This method solves the problem of efficiently storing large, sparse volumetric datasets where most of the space is empty, avoiding the memory overhead of dense 3D arrays.

## Operating characteristics
- **Memory Complexity**: Memory usage is proportional to the number of occupied voxels, as only non-empty positions are stored [S1, unofficial].
- **Performance**: Rendering performance degrades significantly if each voxel is drawn as a separate mesh without instancing, due to high vertex upload costs [S1, unofficial].
- **Strengths**: Simple implementation; efficient sparse storage; guaranteed grid alignment [S1, unofficial].
- **Limitations**: Tracks voxel existence only; rendering efficiency depends on external optimization (instancing) [S1, unofficial].

## Background
The Voxel World Data Structure is described in the tutorial "Make yourself a Voxel Engine THIS weekend" [S1, unofficial]. It is presented as a foundational step in building a voxel engine, preceding more complex structures like meshes and chunks [S1, unofficial].

## Sources
[S1] Make yourself a Voxel Engine THIS weekend, https://daymare.net/blogs/voxel-engine-in-a-weekend/, fetched 2026-10-26 (unofficial)
