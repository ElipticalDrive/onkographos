# Chunk-based Voxel World Management

Also called: Chunk-based Voxel World Storage and Rendering, VoxBox Chunk System, Voxel Chunk Storage and Management

## Overview
Chunk-based Voxel World Management is a system for managing a voxel world by dividing it into fixed-size chunks, handling visibility culling against a frustum, and incrementally rebuilding chunks that have changed [S2, unofficial]. It differs from monolithic storage by enabling per-chunk frustum culling and incremental rebuilds, and from simple chunk storage by actively managing the render pipeline's update budget [S2, unofficial].

## How it works

### Inputs and Outputs
*   **Inputs:** World coordinates $(x, y, z)$, camera frustum, list of modified blocks, frame time budget.
*   **Outputs:** A set of chunks ready for rendering (`_chunkRenderSet`), a set of chunks pending geometry rebuild (`_chunkRebuildSet`), and updated block data structures.

### Data Structures

#### Chunk Manager Grid
The `ChunkManager` stores chunks in a linear `std::vector<Chunk*> _chunks` [S2, unofficial]. The grid is cubic with dimension `_size` [S2, unofficial]. Chunks are accessed via a linear index calculated from chunk coordinates $(cx, cy, cz)$:
$$
\text{index} = (cx \times \text{_size}) + (cy \times \text{_size} \times \text{_size}) + cz
$$
Negative coordinates are clamped to $0$ before indexing [S2, unofficial].

#### Chunk Map (Alternative Implementation)
Some implementations use a hash map (e.g., `.NET Dictionary`) mapping chunk-space position to `Chunk` objects [S3, unofficial]. The key is derived from floor division of world coordinates by `ChunkSize` [S3, unofficial].

#### Block Data Structure
Each block stores metadata in a flat 1D array within the chunk [S3, unofficial]. Fields include:
*   `Id` (int): Block type (e.g., $0$=Air, $1$=Grass, $15$=OakLeaves) [S3, unofficial].
*   `Solid` (bool): Whether the block occludes light/geometry [S3, unofficial].
*   `Transparent` (bool): Whether the block is transparent [S3, unofficial].
*   `BlockLight` (int): Light intensity [S3, unofficial].
*   `IsSmooth` (bool): Smoothing flag [S3, unofficial].
*   `Geometry` (enum/string): Shape definition (e.g., `'Box'`) [S3, unofficial].
*   `TopFace`, `BottomFace`, `LeftFace`, `RightFace` (string): Texture references [S3, unofficial].

#### Chunk Geometry
A chunk contains a mesh. Unity3D Mesh primitives use 16-bit index buffers, limiting a single mesh to 65,536 vertices [S3, unofficial].

### Parameters
*   `ChunkSize`: Dimension of a chunk in blocks. Typical value: $16$ [S3, unofficial].
*   `_rebuildsPerFrame`: Maximum number of chunks rebuilt per frame. Typical value: $5$ [S2, unofficial].
*   `_blocksPerChunk`: Number of blocks per chunk dimension. Equal to `ChunkSize` [S3, unofficial].
*   `_blockSize`: Physical size of a block. Used in coordinate scaling [S2, unofficial].

### Algorithm Steps

#### 1. Coordinate Translation
To access a block at world coordinates $(x, y, z)$:
1.  Calculate chunk indices:
    $$
    cx = \lfloor x / \text{ChunkSize} \rfloor, \quad cy = \lfloor y / \text{ChunkSize} \rfloor, \quad cz = \lfloor z / \text{ChunkSize} \rfloor
    $$
    [S3, unofficial]
2.  Retrieve the chunk from the storage structure (vector or hashmap) using $(cx, cy, cz)$ [S3, unofficial].
3.  Calculate local block indices:
    $$
    lx = x \pmod{\text{ChunkSize}}, \quad ly = y \pmod{\text{ChunkSize}}, \quad lz = z \pmod{\text{ChunkSize}}
    $$
    [S3, unofficial]
4.  Map local 3D coordinates to the flat array index to retrieve the block ID [S3, unofficial].
*Note: An alternative implementation calculates chunk indices using `chunkRenderSize = _blocksPerChunk * _blockSize * 2` and clamps negative coordinates to $0$ before indexing [S2, unofficial].*

#### 2. Visibility Culling (`updateVisibilityList`)
Iterate through all chunks to determine render status [S2, unofficial]:
1.  Loop through chunk grid indices $i, j, k$ from $0$ to `_size` [S2, unofficial].
2.  Get chunk bounds (e.g., bounding sphere) [S2, unofficial].
3.  Check bounds against the camera frustum:
    $$
    \text{side} = \text{frustum.checkSphere}(\text{bounds})
    $$
    [S2, unofficial]
4.  If `side` is `INSIDE` or `INTERSECT`:
    *   If the chunk is already set up (`isSetup()` is true), insert into `_chunkRenderSet` [S2, unofficial].
    *   Otherwise, insert into `_chunkRebuildSet` [S2, unofficial].

#### 3. Incremental Rebuild (`rebuildChunks`)
Process chunks pending geometry updates [S2, unofficial]:
1.  Initialize `rebuiltCount = 0` [S2, unofficial].
2.  Iterate through `_chunkRebuildSet` [S2, unofficial]:
    *   If `rebuiltCount >= _rebuildsPerFrame`, break loop [S2, unofficial].
    *   Call `chunk->build()` to generate geometry [S2, unofficial].
    *   Add chunk to a temporary `rebuiltList` [S2, unofficial].
    *   Insert chunk into `_chunkRenderSet` [S2, unofficial].
    *   Increment `rebuiltCount` [S2, unofficial].
3.  Erase all chunks in `rebuiltList` from `_chunkRebuildSet` [S2, unofficial].

#### 4. Grid Allocation
When initializing, `_size` is determined by the maximum dimension required to cover initial block coordinates [S2, unofficial]:
$$
xSize = \lfloor \text{blockX} / \text{chunkSize} \rfloor
$$
$$
\text{_size} = \max(xSize, ySize, zSize)
$$
This ensures the grid covers the world but may result in a cubic grid even if the world is not cubic [S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

class FrustumSide:
    INSIDE = "INSIDE"
    INTERSECT = "INTERSECT"
    OUTSIDE = "OUTSIDE"

class ChunkManager:
    def __init__(self, size, rebuilds_per_frame=5):
        self._size = size
        self._rebuilds_per_frame = rebuilds_per_frame
        self._chunks = [None] * (size * size * size)
        self._chunk_render_set = set()
        self._chunk_rebuild_set = set()

    def get_chunk_index(self, cx, cy, cz):
        # Clamp negative indices to 0
        cx = max(0, cx)
        cy = max(0, cy)
        cz = max(0, cz)
        return (cx * self._size) + (cy * self._size * self._size) + cz

    def get_block(self, world_x, world_y, world_z):
        chunk_size = 16
        # Calculate chunk position
        cx = math.floor(world_x / chunk_size)
        cy = math.floor(world_y / chunk_size)
        cz = math.floor(world_z / chunk_size)
        
        # Retrieve chunk (simplified access)
        idx = self.get_chunk_index(cx, cy, cz)
        chunk = self._chunks[idx]
        if chunk is None:
            return None
            
        # Calculate local offset
        lx = world_x % chunk_size
        ly = world_y % chunk_size
        lz = world_z % chunk_size
        
        # Map to flat array index
        local_idx = (lx * chunk_size * chunk_size) + (ly * chunk_size) + lz
        return chunk.blocks[local_idx]

    def update_visibility_list(self, frustum):
        self._chunk_render_set.clear()
        self._chunk_rebuild_set.clear()
        
        for i in range(self._size):
            for j in range(self._size):
                for k in range(self._size):
                    idx = self.get_chunk_index(i, j, k)
                    chunk = self._chunks[idx]
                    if chunk is None:
                        continue
                        
                    bounds = chunk.get_bounds()
                    side = frustum.check_sphere(bounds)
                    
                    if side in (FrustumSide.INSIDE, FrustumSide.INTERSECT):
                        if chunk.is_setup():
                            self._chunk_render_set.add(chunk)
                        else:
                            self._chunk_rebuild_set.add(chunk)

    def rebuild_chunks(self):
        rebuilt_count = 0
        rebuilt_list = []
        
        # Iterate through rebuild set
        chunks_to_process = list(self._chunk_rebuild_set)
        
        for chunk in chunks_to_process:
            if rebuilt_count >= self._rebuilds_per_frame:
                break
                
            chunk.build()
            rebuilt_list.append(chunk)
            self._chunk_render_set.add(chunk)
            rebuilt_count += 1
            
        for chunk in rebuilt_list:
            self._chunk_rebuild_set.discard(chunk)
```

### Edge Cases and Pitfalls
*   **Vertex Buffer Limit:** A $16^3$ chunk with a 3D checkerboard pattern ($2,048$ cubes) generates $49,152$ vertices ($2,048 \times 4 \times 6$), which is under the 16-bit index buffer limit of $65,536$ [S3, unofficial]. Complex shapes or transparent blocks may exceed this, requiring mesh splitting [S3, unofficial].
*   **Draw Calls:** Smaller chunks (e.g., $8^3$) increase draw calls by $8\times$, increasing CPU overhead [S3, unofficial].
*   **Grid Shape:** The cubic grid allocation may waste memory if the world is not cubic [S2, unofficial].
*   **Array Bounds:** Accessing multidimensional arrays in some runtimes (e.g., C#) performs multiple bounds checks; using a flat 1D array with custom mapping improves performance [S3, unofficial].

## Operating characteristics

### Time and Memory Complexity
*   **Memory:** A typical world has $\sim5,000$ chunks ($20,480,000$ voxels) [S3, unofficial]. Storing explicit position data ($3$ `int32`s per voxel) would add $\sim234$MB; implicit positioning saves this [S3, unofficial].
*   **Time:** Visibility culling is $O(N)$ where $N$ is the number of chunks in the grid [S2, unofficial]. Rebuilds are limited to `_rebuildsPerFrame` per frame, ensuring constant time per frame for updates [S2, unofficial].

### Measured Performance
*   **Hardware:** Tested on Unity3D with 16-bit index buffers [S3, unofficial].
*   **Resolution:** $16^3$ chunks [S3, unofficial].
*   **Scaling:** Larger chunks risk vertex buffer overflow; smaller chunks increase draw calls [S3, unofficial].

### Strengths and Limitations
*   **Strengths:** Dynamic loading/unloading, frustum culling, localized updates, memory efficiency via implicit positioning [S3, unofficial].
*   **Limitations:** Vertex buffer limits for complex geometry, potential memory waste from cubic grid allocation [S2, S3, unofficial].

### When to Use
*   Use for voxel worlds requiring dynamic modification and large-scale rendering [S3, unofficial].
*   Avoid if the world is static and small, or if geometry complexity consistently exceeds vertex buffer limits without splitting [S3, unofficial].

## Subtopics

## Background
The method originates from voxel engine implementations such as the VoxelEngine by nnarain [S2, unofficial] and the VoxBox project described in "Procedural Generation of Voxel Worlds with Castles" [S3, unofficial]. The $16^3$ chunk size is a common choice, matching Minecraft's render chunks [S3, unofficial]. The approach separates block storage from display chunks in some engines, but this implementation keeps them together for simplicity, using flat arrays for performance [S3, unofficial].

## Sources

[S1] Phase 2: voxel engine com chunks, meshing e edição, https://github.com/EduardoLavall/blockbound/issues/4, fetched 2026-10-26 (unofficial)
[S2] nnarain/VoxelEngine, https://github.com/nnarain/VoxelEngine, fetched 2026-10-27 (unofficial)
[S3] Procedural Generation of Voxel Worlds with Castles, https://researchcommons.waikato.ac.nz/entities/publication/70d98783-1629-43df-9d01-b973ddbb5d75, fetched 2026-10-28 (unofficial)
