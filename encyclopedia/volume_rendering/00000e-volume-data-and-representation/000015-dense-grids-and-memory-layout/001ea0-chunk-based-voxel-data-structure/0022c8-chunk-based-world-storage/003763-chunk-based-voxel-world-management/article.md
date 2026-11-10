# Chunk-based Voxel World Management

Also called: Chunk-based Voxel World Storage and Rendering, VoxBox Chunk System

## Overview
Chunk-based Voxel World Management is a system for managing a voxel world by dividing it into fixed-size chunks, handling visibility culling against a frustum, and incrementally rebuilding chunks that have changed [S1, S2, unofficial]. It enables dynamic loading/unloading, localized updates, and efficient rendering by isolating geometry generation to visible or modified regions [S2, unofficial]. It differs from monolithic storage by allowing per-chunk frustum culling and incremental rebuilds, and from simple chunk storage by actively managing the render pipeline's update budget [S1, S2, unofficial].

## How it works

### Inputs and Outputs
*   **Inputs:** World coordinates $(x, y, z)$, camera frustum, list of modified blocks, frame time budget.
*   **Outputs:** A set of chunks ready for rendering (`_chunkRenderSet`), a set of chunks pending geometry rebuild (`_chunkRebuildSet`), and updated block data structures.

### Data Structures

#### Chunk Manager Grid
The `ChunkManager` stores chunks in a linear `std::vector<Chunk*> _chunks` [S1, unofficial]. The grid is cubic with dimension `_size` [S1, unofficial]. Chunks are accessed via a linear index calculated from chunk coordinates $(cx, cy, cz)$:
$$
\text{index} = (cx \times \text{_size}) + (cy \times \text{_size} \times \text{_size}) + cz
$$
Negative coordinates are clamped to 0 before indexing [S1, unofficial].

#### Chunk Map (Alternative Implementation)
Some implementations use a hash map (e.g., `.NET Dictionary`) mapping chunk-space position to `Chunk` objects [S2, unofficial]. The key is derived from floor division of world coordinates by `ChunkSize` [S2, unofficial].

#### Block Data Structure
Each block stores metadata in a flat 1D array within the chunk [S2, unofficial]. Fields include:
*   `Id` (int): Block type (e.g., 0=Air, 1=Grass, 15=OakLeaves) [S2, unofficial].
*   `Solid` (bool): Whether the block occludes light/geometry [S2, unofficial].
*   `Transparent` (bool): Whether the block is transparent [S2, unofficial].
*   `BlockLight` (int): Light intensity [S2, unofficial].
*   `IsSmooth` (bool): Smoothing flag [S2, unofficial].
*   `Geometry` (enum/string): Shape definition (e.g., 'Box') [S2, unofficial].
*   `TopFace`, `BottomFace`, `LeftFace`, `RightFace` (string): Texture references [S2, unofficial].

#### Chunk Geometry
A chunk contains a mesh. Unity3D Mesh primitives use 16-bit index buffers, limiting a single mesh to 65,536 vertices [S2, unofficial].

### Parameters
*   `ChunkSize`: Dimension of a chunk in blocks. Typical value: $16$ [S2, unofficial].
*   `_rebuildsPerFrame`: Maximum number of chunks rebuilt per frame. Typical value: $5$ [S1, unofficial].
*   `_blocksPerChunk`: Number of blocks per chunk dimension. Equal to `ChunkSize` [S1, S2, unofficial].
*   `_blockSize`: Physical size of a block. Used in coordinate scaling [S1, unofficial].

### Algorithm Steps

#### 1. Coordinate Translation
To access a block at world coordinates $(x, y, z)$:
1.  Calculate chunk indices:
    $$
    cx = \lfloor x / \text{ChunkSize} \rfloor, \quad cy = \lfloor y / \text{ChunkSize} \rfloor, \quad cz = \lfloor z / \text{ChunkSize} \rfloor
    $$
    [S2, unofficial]
2.  Retrieve the chunk from the storage structure (vector or hashmap) using $(cx, cy, cz)$ [S2, unofficial].
3.  Calculate local block indices:
    $$
    lx = x \pmod{\text{ChunkSize}}, \quad ly = y \pmod{\text{ChunkSize}}, \quad lz = z \pmod{\text{ChunkSize}}
    $$
    [S2, unofficial]
4.  Map local 3D coordinates to the flat array index to retrieve the block ID [S2, unofficial].

*Note: One source uses `blockRenderSize = _blockSize * 2` for block index calculation and `chunkRenderSize = _blocksPerChunk * _blockSize * 2` for chunk index calculation, clamping negatives to 0 [S1, unofficial]. Another source uses direct integer division by `ChunkSize` (16) [S2, unofficial]. The integer division method is standard for voxel grids.*

#### 2. Visibility Culling (`updateVisibilityList`)
Iterate through all chunks to determine render status [S1, unofficial]:
1.  Loop through chunk grid indices $i, j, k$ from $0$ to `_size` [S1, unofficial].
2.  Get chunk bounds (e.g., bounding sphere) [S1, unofficial].
3.  Check bounds against the camera frustum:
    $$
    \text{side} = \text{frustum.checkSphere}(\text{bounds})
    $$
    [S1, unofficial]
4.  If `side` is `INSIDE` or `INTERSECT`:
    *   If the chunk is already set up (`isSetup()` is true), insert into `_chunkRenderSet` [S1, unofficial].
    *   Otherwise, insert into `_chunkRebuildSet` [S1, unofficial].

#### 3. Incremental Rebuild (`rebuildChunks`)
Process chunks pending geometry updates [S1, unofficial]:
1.  Initialize `rebuiltCount = 0` [S1, unofficial].
2.  Iterate through `_chunkRebuildSet` [S1, unofficial]:
    *   If `rebuiltCount >= _rebuildsPerFrame`, break loop [S1, unofficial].
    *   Call `chunk->build()` to generate geometry [S1, unofficial].
    *   Add chunk to a temporary `rebuiltList` [S1, unofficial].
    *   Insert chunk into `_chunkRenderSet` [S1, unofficial].
    *   Increment `rebuiltCount` [S1, unofficial].
3.  Erase all chunks in `rebuiltList` from `_chunkRebuildSet` [S1, unofficial].

#### 4. Grid Allocation
When initializing, `_size` is determined by the maximum dimension required to cover initial block coordinates [S1, unofficial]:
$$
xSize = \lfloor \text{blockX} / \text{chunkSize} \rfloor
$$
$$
\text{_size} = \max(xSize, ySize, zSize)
$$
This ensures the grid covers the world but may result in a cubic grid even if the world is not cubic [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
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
        cx = floor(world_x / chunk_size)
        cy = floor(world_y / chunk_size)
        cz = floor(world_z / chunk_size)
        
        # Retrieve chunk (simplified access)
        idx = self.get_chunk_index(cx, cy, cz)
        chunk = self._chunks[idx]
        if chunk is None:
            return None
            
        # Calculate local offset
        lx = world_x % chunk_size
        ly = world_y % chunk_size
        lz = world_z % chunk_size
        
        # Map to flat array index (example logic)
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
                    
                    if side in [FrustumSide.INSIDE, FrustumSide.INTERSECT]:
                        if chunk.is_setup():
                            self._chunk_render_set.add(chunk)
                        else:
                            self._chunk_rebuild_set.add(chunk)

    def rebuild_chunks(self):
        rebuilt_count = 0
        rebuilt_list = []
        
        # Iterate through rebuild set
        # Note: In Python, iterating over a set while modifying it requires care.
        # Here we iterate over a copy or list.
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
*   **Vertex Buffer Limit:** A $16^3$ chunk with a 3D checkerboard pattern (2,048 cubes) generates 49,152 vertices ($2,048 \times 4 \times 6$), which is under the 16-bit index buffer limit of 65,536 [S2, unofficial]. Complex shapes or transparent blocks may exceed this, requiring mesh splitting [S2, unofficial].
*   **Draw Calls:** Smaller chunks (e.g., $8^3$) increase draw calls by 8x, increasing CPU overhead [S2, unofficial].
*   **Grid Shape:** The cubic grid allocation may waste memory if the world is not cubic [S1, unofficial].
*   **Array Bounds:** Accessing multidimensional arrays in some runtimes (e.g., C#) performs multiple bounds checks; using a flat 1D array with custom mapping improves performance [S2, unofficial].

## Operating characteristics

### Time and Memory Complexity
*   **Memory:** A typical world has ~5,000 chunks ($20,480,000$ voxels) [S2, unofficial]. Storing explicit position data (3 `int32`s per voxel) would add ~234MB; implicit positioning saves this [S2, unofficial].
*   **Time:** Visibility culling is $O(N)$ where $N$ is the number of chunks in the grid [S1, unofficial]. Rebuilds are limited to `_rebuildsPerFrame` per frame, ensuring constant time per frame for updates [S1, unofficial].

### Measured Performance
*   **Hardware:** Tested on Unity3D with 16-bit index buffers [S2, unofficial].
*   **Resolution:** $16^3$ chunks [S2, unofficial].
*   **Scaling:** Larger chunks risk vertex buffer overflow; smaller chunks increase draw calls [S2, unofficial].

### Strengths and Limitations
*   **Strengths:** Dynamic loading/unloading, frustum culling, localized updates, memory efficiency via implicit positioning [S2, unofficial].
*   **Limitations:** Vertex buffer limits for complex geometry, potential memory waste from cubic grid allocation [S1, S2, unofficial].

### When to Use
*   Use for voxel worlds requiring dynamic modification and large-scale rendering [S2, unofficial].
*   Avoid if the world is static and small, or if geometry complexity consistently exceeds vertex buffer limits without splitting [S2, unofficial].

## Background
The method originates from voxel engine implementations such as the VoxelEngine by nnarain [S1, unofficial] and the VoxBox project described in "Procedural Generation of Voxel Worlds with Castles" [S2, unofficial]. The $16^3$ chunk size is a common choice, matching Minecraft's render chunks [S2, unofficial]. The approach separates block storage from display chunks in some engines, but this implementation keeps them together for simplicity, using flat arrays for performance [S2, unofficial].

## Sources

[S1] nnarain/VoxelEngine, https://github.com/nnarain/VoxelEngine, fetched 2026-10-27 (unofficial)
[S2] Procedural Generation of Voxel Worlds with Castles, https://researchcommons.waikato.ac.nz/entities/publication/70d98783-1629-43df-9d01-b973ddbb5d75, fetched 2026-10-28 (unofficial)
