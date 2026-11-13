# Multi-type Voxel Storage

Also called: HashMap-based Voxel Storage, VoxelKind Enum Storage

## Overview
Multi-type Voxel Storage is a sparse voxel data structure that maps 3D integer coordinates to discrete material identifiers using a hash map, enabling the representation of heterogeneous voxel worlds where different blocks have distinct properties [S1, unofficial]. It extends the parent structure's chunk-based spatial partitioning by replacing binary existence flags with explicit material enumeration [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A world state containing a collection of chunks, a 3D integer position (`IVec3`) in world space, and an optional `VoxelKind` (or `none` for air) [S1, unofficial].
*   **Output:** Updated world state with the voxel kind set or removed at the specified position, and the associated chunk marked for mesh regeneration [S1, unofficial].

**Data Structures:**
1.  **Chunk:**
    *   `position`: `IVec3` representing the chunk's coordinate in chunk-space (each chunk is $32 \times 32 \times 32$ voxels) [S1, unofficial].
    *   `voxels`: `HashMap<IVec3, VoxelKind>`. Maps local 3D integer coordinates (relative to the chunk origin) to the material type [S1, unofficial].
    *   `mesh`: Optional mesh object. `none` indicates the mesh is invalid and needs regeneration [S1, unofficial].
2.  **VoxelKind:** An enumeration defining material types (e.g., `Dirt`, `Stone`) [S1, unofficial].
3.  **World:** A collection mapping chunk positions to `Chunk` objects [S1, unofficial].

**Algorithm Steps:**
1.  **Chunk Localization:** Given a world position $P$, calculate the chunk position $C$ and the local position $L$ within that chunk:
    $$ C = \text{div\_euclid}(P, 32) $$
    $$ L = \text{rem\_euclid}(P, 32) $$
    Where $\text{div\_euclid}$ and $\text{rem\_euclid}$ are Euclidean division and remainder operations, ensuring correct handling of negative coordinates [S1, unofficial].
2.  **Chunk Retrieval/Creation:** Retrieve the `Chunk` at $C$ from the world. If it does not exist, create it [S1, unofficial].
    *   Initialize `Chunk.voxels` as an empty hash map [S1, unofficial].
    *   **Initial Generation:** If the chunk is being created for the first time, populate it based on world geometry rules [S1, unofficial]:
        *   Iterate $x, y, z$ from $0$ to $31$ [S1, unofficial].
        *   Calculate world position $W = C \times 32 + (x, y, z)$ [S1, unofficial].
        *   If $W.y == 0$, insert $(x, y, z)$ with `VoxelKind::Dirt` [S1, unofficial].
        *   If $W.y < 0$, insert $(x, y, z)$ with `VoxelKind::Stone` [S1, unofficial].
3.  **Voxel Update:**
    *   If the requested `kind` is `none` (air), remove the key $L$ from `Chunk.voxels` [S1, unofficial].
    *   Otherwise, insert the key $L$ with value `kind` into `Chunk.voxels` [S1, unofficial].
4.  **Invalidation:** Set `Chunk.mesh` to `none` to signal that the visual representation is out of date [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class IVec3:
    def __init__(self, x, y, z): self.x, self.y, self.z = x, y, z
    def __mul__(self, s): return IVec3(self.x * s, self.y * s, self.z * s)
    def __add__(self, o): return IVec3(self.x + o.x, self.y + o.y, self.z + o.z)
    def div_euclid(self, d): return IVec3(self.x // d, self.y // d, self.z // d)
    def rem_euclid(self, d): return IVec3(self.x % d, self.y % d, self.z % d)

class VoxelKind:
    NONE = 0
    DIRT = 1
    STONE = 2

class Chunk:
    def __init__(self, position):
        self.position = position
        self.voxels = {}
        self.mesh = None

class World:
    def __init__(self):
        self.chunks = {}

def set_voxel(world: World, position: IVec3, kind: int):
    chunk_position = IVec3(position.x // 32, position.y // 32, position.z // 32)
    local_position = IVec3(position.x % 32, position.y % 32, position.z % 32)
    chunk = get_or_create_chunk(world, chunk_position)
    chunk.mesh = None
    if kind == VoxelKind.NONE:
        if local_position in chunk.voxels:
            del chunk.voxels[local_position]
    else:
        chunk.voxels[local_position] = kind

def get_or_create_chunk(world: World, chunk_position: IVec3) -> Chunk:
    if chunk_position not in world.chunks:
        chunk = Chunk(chunk_position)
        for z in range(32):
            for y in range(32):
                for x in range(32):
                    offset = IVec3(x, y, z)
                    world_voxel_position = chunk_position * 32 + offset
                    if world_voxel_position.y == 0:
                        chunk.voxels[offset] = VoxelKind.DIRT
                    elif world_voxel_position.y < 0:
                        chunk.voxels[offset] = VoxelKind.STONE
        world.chunks[chunk_position] = chunk
    return world.chunks[chunk_position]
```

**Parameters and Edge Cases:**
*   **Chunk Size:** Fixed at $32^3$ voxels [S1, unofficial].
*   **Coordinate System:** Uses Euclidean division/remainder to handle negative world coordinates correctly, ensuring consistent chunk mapping for negative indices [S1, unofficial].
*   **Air Representation:** Air is implicit. Only non-air voxels are stored in the hash map [S1, unofficial].
*   **Mesh Invalidation:** Every modification to a voxel invalidates the chunk's mesh [S1, unofficial].

## Subtopics

### [Typed Voxels](1399-typed-voxels.md)
Extends this structure by enforcing a strict enumeration-based type system for voxel content, allowing distinct material properties beyond simple existence. Choose it when heterogeneous material simulation is required.

### [Voxel Hashing](2702-voxel-hashing.md)
Uses a flat spatial hash table for $O(1)$ block access, prioritizing GPU parallelism and constant-time lookup over chunk-based indexing. Choose it for real-time 3D scene reconstruction with dynamic allocation.

### [Adaptive Bit-Packed Voxel Array](4986-adaptive-bit-packed-voxel-array.md)
Implements adaptive bit-packing within a single section to optimize memory for homogeneous regions, differing from this method's direct hash map storage. Choose it when memory efficiency for fixed-size sections is prioritized.

## Operating characteristics
*   **Time Complexity:** Lookup and insertion are $O(1)$ average case due to hash map operations [G]. Chunk localization is $O(1)$ [S1, unofficial]. Mesh regeneration is $O(N)$ where $N$ is the number of voxels in the chunk [G].
*   **Memory Complexity:** Proportional to the number of non-air voxels. Each entry stores a 3D integer key and an enum value, plus hash map overhead. Empty space is not stored [S1, unofficial].
*   **Strengths:** Simple implementation; efficient storage for sparse worlds; easy to add new voxel types via enum extension [S1, unofficial].
*   **Limitations:** Hash map overhead can be significant for dense voxels [G]. Frequent voxel updates cause frequent mesh invalidations [S1, unofficial]. Not optimized for GPU direct access without additional data restructuring [G].
*   **When to Use:** Ideal for voxel engines where world sparsity is high, and dynamic modification of voxel types is frequent [S1, unofficial]. Not suitable for highly dense, static volumes where contiguous arrays or octrees might be more efficient [G].

## Background
This method originates from practical implementations of voxel engines, specifically described in "Make yourself a Voxel Engine THIS weekend" [S1, unofficial]. It builds upon the parent structure's chunk-based spatial partitioning by replacing binary existence flags with explicit material enumeration, allowing for heterogeneous voxel worlds [S1, unofficial].

## Sources
[S1] Make yourself a Voxel Engine THIS weekend, https://daymare.net/blogs/voxel-engine-in-a-weekend/, fetched 2026-10-26 (unofficial)
[S2] Voxel Hashing, https://example.com/voxel-hashing, fetched 2026-10-26 (unofficial)
