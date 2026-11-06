# Chunk Border Neighbor Access

Also called: Read-only neighborhood view, VoxelNeighborhood trait

## Overview
Chunk Border Neighbor Access is a strategy for handling neighbor lookups at chunk boundaries by using a read-only neighborhood view trait, treating chunk borders as semantically identical to internal voxels [S1, unofficial]. It solves the problem of cross-chunk data access during operations like meshing or ray marching, where an algorithm expects a uniform local neighborhood but the data is partitioned into separate memory blocks. It sets itself apart from sibling methods like [Sparse Voxel Octree Representation](2340-sparse-voxel-octree-representation.md) by maintaining a dense grid layout within chunks while abstracting the boundary logic, and differs from [Compact Voxel Section Storage](4219-compact-voxel-section-storage.md) by explicitly providing a unified interface for both intra-chunk and inter-chunk queries.

## How it works
**Inputs and Outputs:**
- Input: A position in global voxel coordinates ($P \in \mathbb{Z}^3$).
- Output: The block state ($B$) at that position, or a null/missing state if the chunk is not available.

**Data Structures:**
The core abstraction is the `VoxelNeighborhood` trait (or interface). It does not store data itself but provides access to it.
- `VoxelNeighborhood`: An interface with a single method:
  - `get(position: IVec3) -> Option<BlockState>`
  - `IVec3`: A 3D integer vector representing global voxel coordinates.
  - `BlockState`: The data type representing a voxel's content (e.g., material ID, density).
  - `Option`: A type indicating either a valid `BlockState` or a "missing" state.

**Steps:**
1. The rendering or meshing algorithm requests the state of a neighbor at global coordinate $P$ using `neighborhood.get(P)`.
2. The implementation determines which chunk contains $P$ based on the chunk size and origin.
3. If the chunk is loaded and $P$ is within the chunk's bounds, the value is retrieved from the chunk's internal array.
4. If the chunk is not loaded or $P$ is out of bounds, the implementation returns a "missing" state.
5. The algorithm handles the "missing" state according to a predefined policy (see Pitfalls).

**Formulas:**
- Chunk Index for coordinate $P$:
  $$ C = \lfloor P / S \rfloor $$
  Where $S$ is the chunk size (assumed uniform in all dimensions).
- Local Offset within Chunk:
  $$ L = P \pmod S $$
  (Note: Modulo must handle negative coordinates correctly, typically by adding $S$ if the result is negative).

**Parameters:**
- Chunk Size ($S$): Not given in the sources. Typical values depend on the parent structure [Chunk-based Voxel Data Structure](1818-chunk-based-voxel-data-structure.md).

**Edge Cases and Pitfalls:**
- **Missing Neighbors:** If a neighbor chunk is not currently available (e.g., not yet streamed in), the engine must define a policy [S1, unofficial]. Common options include:
  1. Treating missing data as air and remeshing later.
  2. Delaying meshing until required neighbors are present.
  3. Using an explicit "unknown" state.
  The correct choice depends on the world-streaming model [S1, unofficial].
- **Transparency:** Geometry coverage and render opacity are related but not identical. Two adjacent glass blocks may geometrically cover each other, but whether their shared face should be removed depends on material/rendering semantics. A robust culling decision requires material-aware neighbor access [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Assuming a global VoxelWorld class that manages chunks
class VoxelWorld:
    def __init__(self, chunk_size: int):
        self.chunk_size = chunk_size
        self.chunks = {}  # Map from ChunkCoord to ChunkData

    def get_block_state(self, position: IVec3) -> Optional[BlockState]:
        """
        Implements the VoxelNeighborhood trait.
        Returns the block state at the global position, or None if unavailable.
        """
        # 1. Determine chunk coordinates
        # Handle negative modulo correctly for chunk indexing
        chunk_x = floor_div(position.x, self.chunk_size)
        chunk_y = floor_div(position.y, self.chunk_size)
        chunk_z = floor_div(position.z, self.chunk_size)
        chunk_coord = (chunk_x, chunk_y, chunk_z)

        # 2. Check if chunk is loaded
        if chunk_coord not in self.chunks:
            return None

        chunk = self.chunks[chunk_coord]

        # 3. Calculate local coordinates within the chunk
        # floor_div ensures correct wrapping for negative numbers
        local_x = position.x - chunk_x * self.chunk_size
        local_y = position.y - chunk_y * self.chunk_size
        local_z = position.z - chunk_z * self.chunk_size

        # 4. Retrieve from chunk's internal array
        # Assuming chunk has a method to get local block state
        return chunk.get_block_at(local_x, local_y, local_z)

def floor_div(a: int, b: int) -> int:
    """Integer division that floors towards negative infinity."""
    return a // b if a * b >= 0 else -((-a) // b)
```

## Subtopics

### [Chunk Neighbor Referencing for Boundary Meshing](4171-chunk-neighbor-referencing-for-boundary-meshing.md)
Chunk Neighbor Referencing is a strategy for handling meshing at chunk boundaries by precomputing pointers to adjacent chunks, thereby avoiding global lookups during the meshing process. It builds on Chunk Border Neighbor Access by providing the specific data structures and logic required to resolve neighbor identities for boundary voxels efficiently. It is chosen when meshing performance is critical and the set of boundary voxels is known in advance, allowing for optimized cache locality during the meshing pass.

### Padded Chunk Data Structure
A 3D array of voxels with 1-voxel boundary padding on all sides to allow neighbor lookups without bounds checks [S2, unofficial]. It sets itself apart by embedding the boundary logic into the data layout rather than the access logic. It is chosen when the overhead of boundary checks is significant and the memory cost of padding is acceptable.
- **Layout:** `PaddedChunk` is a flat array of type `[VoxelBlock; PADDED_CHUNK_VOLUME]` where `PADDED_CHUNK_SIZE = CHUNK_SIZE_U32 + 2` and `PADDED_CHUNK_VOLUME = PADDED_CHUNK_SIZE^3` [S2, unofficial].
- **Formula:** Linearization index for a 3D coordinate `[x, y, z]` in a padded chunk with X innermost: `index = x + PADDED_CHUNK_SIZE * (y + PADDED_CHUNK_SIZE * z)` [S2, unofficial].

### Chunk Neighbor Retrieval
Retrieves the 6 face-adjacent neighbor chunk coordinates for a given center chunk coordinate, or all 26 surrounding neighbor chunk coordinates (including diagonals) [S3, unofficial]. It sets itself apart by providing explicit coordinate generation for neighbor discovery, which can be used to populate the `VoxelNeighborhood` cache.
- **Steps for 6 neighbors:** Reset the output array to size 6. Add `ChunkCoord + (1,0,0)`, `(-1,0,0)`, `(0,1,0)`, `(0,-1,0)`, `(0,0,1)`, `(0,0,-1)` [S3, unofficial].
- **Steps for 26 neighbors:** Reset the output array to size 26. Loop X from -1 to 1, Y from -1 to 1, Z from -1 to 1. If X, Y, Z are not all zero, add `ChunkCoord + (X,Y,Z)` to the output [S3, unofficial].

## Operating characteristics
- **Time Complexity:** Lookup is $O(1)$ if the chunk is loaded and the position is valid. If the chunk is not loaded, it depends on the loading mechanism (e.g., disk I/O or streaming latency).
- **Memory Complexity:** $O(1)$ overhead per chunk for the neighbor lookup structure, plus the memory for the chunks themselves. Padded chunks add a fixed overhead of $( (S+2)^3 - S^3 )$ voxels per chunk.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales well with chunk count as long as the neighbor lookup mechanism (e.g., hash map or sparse array) remains efficient.
- **Strengths:** Decouples meshing/ray marching logic from chunk boundary logic; simplifies code by treating internal and boundary voxels uniformly.
- **Limitations:** Requires explicit handling of missing chunks; padding increases memory usage.
- **When to use:** When using dense chunk-based storage and needing uniform neighbor access for algorithms like Marching Cubes or ray marching.
- **When not to use:** When memory is extremely constrained and padding is prohibitive, or when using sparse structures like Octrees where neighbor access is handled by tree traversal.

## Background
The concept of treating chunk borders as semantically identical to internal voxels via a read-only neighborhood view was described in the context of face culling for voxel cubes and partial shapes [S1, unofficial]. The padded chunk data structure is used in the Infinigen project [S2, unofficial]. The neighbor retrieval algorithms are found in the VoxelWorlds project [S3, unofficial].

## Sources
[S1] Face Culling for Voxel Cubes and Partial Shapes, https://ogyrecdev.github.io/posts/face-culling/, fetched 2026-10-28 (unofficial)
[S2] jameshiew/infinigen, https://github.com/jameshiew/infinigen, fetched 2026-10-27 (unofficial)
[S3] draquel/VoxelWorlds, https://github.com/draquel/VoxelWorlds, fetched 2026-10-27 (unofficial)
