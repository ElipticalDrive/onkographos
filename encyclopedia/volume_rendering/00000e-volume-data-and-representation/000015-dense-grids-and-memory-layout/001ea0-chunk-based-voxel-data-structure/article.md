# Chunk-based Voxel Data Structure

Also called: Voxel Chunking, Voxel Chunk Data Structure, Chunked Voxel World

## Overview
The Chunk-based Voxel Data Structure organizes a 3D voxel grid into fixed-size 3D blocks (chunks) to enable spatial locality, flat array indexing, and incremental updates [S1, S4, S5, unofficial]. It solves the problem of managing infinite or large-scale voxel worlds by dividing space into manageable units that can be independently meshed, serialized, and loaded/unloaded [S5, unofficial]. Unlike dense grids that allocate memory for the entire world volume, or sparse octrees that use complex tree traversal, chunks provide a flat, predictable memory layout with constant-time access, trading some memory overhead for simplified meshing and I/O operations [S4, S5, unofficial].

## How it works
**Inputs:** Global voxel coordinates $(x, y, z)$, chunk dimensions ($C_x, C_y, C_z$), and maximum world bounds or chunk counts.
**Outputs:** A pointer or index to the specific voxel data within a chunk, or the chunk object itself.

**Data Structures:**
1.  **Chunk:** A container holding voxel data and metadata.
    *   `data`: A flat 1D array (e.g., `Uint8Array` or `byte[]`) of size $C_x \times C_y \times C_z$. Stores voxel IDs (0 for air, 1+ for solid/material) [S2, S4, unofficial].
    *   `cx, cz`: Chunk coordinates in the world grid (integer indices) [S4, unofficial].
    *   `mesh`: Reference to the generated mesh geometry for this chunk [S3, S4, unofficial].
    *   `present`: Optional 1D boolean/integer array indicating if a block exists at a position (used for sparse-like behavior within a chunk) [S5, unofficial].
2.  **World/Map:** A collection of chunks.
    *   Can be implemented as a hash map (keyed by chunk coordinates) [S4, unofficial] or a pre-allocated flat array of chunk pointers/structs [S1, unofficial].
    *   If flat array: `chunks` array of size $MaxX \times MaxY \times MaxZ$ [S1, unofficial].

**Indexing Formulas:**
1.  **Global to Chunk Index:**
    Given global voxel $(x, y, z)$ and chunk size $S$ (assuming cubic chunks $C_x=C_y=C_z=S$ for simplicity, or using bit shifts for powers of 2):
    $$
    f = \lfloor x / C_x \rfloor, \quad g = \lfloor y / C_y \rfloor, \quad h = \lfloor z / C_z \rfloor
    $$
    If using bit shifts (where $S = 2^k$):
    $$
    f = x \gg k, \quad g = y \gg k, \quad h = z \gg k
    $$
    [S1, S4, unofficial]

2.  **Chunk Array Index (Flat Array Layout):**
    If the world is a flat array of chunks with dimensions $MaxX, MaxY, MaxZ$:
    $$
    index_{chunk} = f + g \cdot MaxX + h \cdot MaxX \cdot MaxY
    $$
    [S1, unofficial]
    *Note: Some implementations use a hash map to support infinite worlds, avoiding fixed $MaxX$ bounds [S4, unofficial].*

3.  **Local Voxel Index within Chunk:**
    $$
    x_{local} = x \pmod{C_x}, \quad y_{local} = y \pmod{C_y}, \quad z_{local} = z \pmod{C_z}
    $$
    Or using bitwise AND (for power-of-2 sizes):
    $$
    x_{local} = x \ \& \ (C_x - 1)
    $$
    [S1, unofficial]

4.  **Flat Voxel Data Index:**
    Row-major layout (Y is the major stride in some implementations, Z in others; standard is often X fastest, then Y, then Z, or Z fastest):
    *   **Layout A (from S4, S7):** $Y$ is major stride.
        $$
        index_{voxel} = y_{local} \cdot (C_x \cdot C_z) + z_{local} \cdot C_x + x_{local}
        $$
    *   **Layout B (from S2):** $X$ is major stride.
        $$
        index_{voxel} = x_{local} + y_{local} \cdot s + z_{local} \cdot s \cdot s
        $$
        Where $s$ is the chunk size [S2, unofficial].
    *   *Implementation Note:* Consistency is key. The pseudocode below uses Layout A as it is common in Minecraft-like engines [S4, S7, unofficial].

**Parameters:**
*   **Chunk Size:** Typically $16^3$ or $32^3$ voxels [S5, unofficial].
    *   $16 \times 256 \times 16$ is used in some engines to keep height data small and re-meshing fast while allowing tall structures [S4, unofficial].
    *   Too large: Slow remeshing when a single block changes [S5, unofficial].
    *   Too small: Excessive triangle counts and overhead [S5, unofficial].
*   **Boundary Policy:** "Air" (return 0 if out of bounds) or "Solid" [S2, unofficial].

**Algorithm Steps:**
1.  **Get Voxel:**
    *   Calculate chunk indices $(f, g, h)$ from global $(x, y, z)$.
    *   Retrieve the chunk object from the World container.
    *   Calculate local indices $(x_{local}, y_{local}, z_{local})$.
    *   Compute flat index and return `chunk.data[index]`.
    *   If chunk is null or coordinates are out of world bounds, return Air (0) [S2, S4, unofficial].
2.  **Set Voxel:**
    *   Retrieve or initialize the chunk at $(f, g, h)$.
    *   Compute local index.
    *   Set `chunk.data[index] = material_id`.
    *   Mark chunk as "dirty" or "modified" to trigger mesh regeneration [S7, unofficial].
3.  **Meshing Trigger:**
    *   When a voxel is set, check if it is on the chunk boundary.
    *   If on boundary, add adjacent chunks to a modification queue [S7, unofficial].
    *   Regenerate mesh for the modified chunk and its neighbors [S5, S7, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
CHUNK_SIZE_X = 16
CHUNK_SIZE_Y = 256
CHUNK_SIZE_Z = 16
AIR_ID = 0

class Chunk:
    def __init__(self, cx, cz):
        self.cx = cx
        self.cz = cz
        # Flat array for voxel data
        self.data = [AIR_ID] * (CHUNK_SIZE_X * CHUNK_SIZE_Y * CHUNK_SIZE_Z)
        self.mesh = None
        self.dirty = True

    def get_voxel(self, x, y, z):
        # Boundary check
        if x < 0 or x >= CHUNK_SIZE_X or \
           y < 0 or y >= CHUNK_SIZE_Y or \
           z < 0 or z >= CHUNK_SIZE_Z:
            return AIR_ID
        # Row-major index: y is major stride
        idx = y * (CHUNK_SIZE_X * CHUNK_SIZE_Z) + z * CHUNK_SIZE_X + x
        return self.data[idx]

    def set_voxel(self, x, y, z, material_id):
        if x < 0 or x >= CHUNK_SIZE_X or \
           y < 0 or y >= CHUNK_SIZE_Y or \
           z < 0 or z >= CHUNK_SIZE_Z:
            return
        idx = y * (CHUNK_SIZE_X * CHUNK_SIZE_Z) + z * CHUNK_SIZE_X + x
        self.data[idx] = material_id
        self.dirty = True

class VoxelWorld:
    def __init__(self):
        # Using a dictionary for infinite world support
        self.chunks = {}

    def _get_chunk_coords(self, x, y, z):
        cx = x // CHUNK_SIZE_X
        cy = y // CHUNK_SIZE_Y
        cz = z // CHUNK_SIZE_Z
        return (cx, cy, cz)

    def _get_local_coords(self, x, y, z):
        lx = x % CHUNK_SIZE_X
        ly = y % CHUNK_SIZE_Y
        lz = z % CHUNK_SIZE_Z
        return (lx, ly, lz)

    def get_voxel(self, x, y, z):
        cx, cy, cz = self._get_chunk_coords(x, y, z)
        key = (cx, cz) # Assuming 2D chunk grid for infinite world (common in Minecraft)
        # Note: Some implementations use 3D chunk keys if world is bounded
        chunk = self.chunks.get(key)
        if chunk is None:
            return AIR_ID
        lx, ly, lz = self._get_local_coords(x, y, z)
        return chunk.get_voxel(lx, ly, lz)

    def set_voxel(self, x, y, z, material_id):
        cx, cy, cz = self._get_chunk_coords(x, y, z)
        key = (cx, cz)
        
        if key not in self.chunks:
            self.chunks[key] = Chunk(cx, cz)
        
        chunk = self.chunks[key]
        lx, ly, lz = self._get_local_coords(x, y, z)
        chunk.set_voxel(lx, ly, lz, material_id)
        
        # Queue mesh regeneration for this chunk and neighbors if on boundary
        if self._is_on_boundary(lx, ly, lz):
            self._queue_chunk_for_meshing(key)
            self._queue_neighbors_for_meshing(key)

    def _is_on_boundary(self, lx, ly, lz):
        return (lx == 0 or lx == CHUNK_SIZE_X - 1 or
                ly == 0 or ly == CHUNK_SIZE_Y - 1 or
                lz == 0 or lz == CHUNK_SIZE_Z - 1)

    def _queue_chunk_for_meshing(self, key):
        # Implementation detail: add to a list of chunks to regenerate
        pass

    def _queue_neighbors_for_meshing(self, key):
        # Implementation detail: add adjacent chunk keys to the list
        pass
```

## Subtopics

### [Sparse Voxel Octree Representation](2340-sparse-voxel-octree-representation.md)
A Sparse Voxel Octree (SVO) is a hierarchical data structure for representing 3D space that eliminates sparse memory by not storing empty volumes. It enables efficient storage and traversal of voxel data, particularly for indirect illumination calculations at geometry-independent costs. Unlike dense grids or chunk-based structures, it adapts resolution to content density, trading traversal performance for memory efficiency.

### [Chunk-Based World Storage](2858-chunk-based-world-storage.md)
Divides the voxel world into fixed-size chunks to manage memory and rendering distance. Handles coordinate translation between world space and local chunk space.

### [Chunk Border Neighbor Access](2860-chunk-border-neighbor-access.md)
A strategy for handling neighbor lookups at chunk boundaries by using a read-only neighborhood view trait, treating chunk borders as semantically identical to internal voxels.

### [Compact Voxel Section Storage](4219-compact-voxel-section-storage.md)
A data structure representing a 16x16x16 voxel section that uses three storage modes to minimize memory: Empty (no allocation for all-air), Uniform (no allocation for single material), and Palette (packed indices for mixed materials).

### Terrain Chunk Data Structure
A value type struct representing a terrain chunk with its index, world position, size, and LOD. A reference type class representing a terrain chunk includes a computed max corner. A collection class manages an array of terrain chunks with global resolution and dimensions [S6, unofficial].
*   **TerrainChunk_Val:** Fields are `ChunkIndex` (int3), `MinCornerWS` (float3), `ChunkSize` (float3), `ChunkLOD` (TerrainLod).
*   **TerrainChunk_Ref:** Fields are `ChunkIndex` (int3), `MinCornerWS` (float3), `ChunkSize` (float3), `MaxCornerWS` (computed float3), `ChunkLOD` (TerrainLod).
*   **TerrainChunkCollection:** Fields are `TerrainChunks` (TerrainChunk_Val array), `Resolution` (int3), `ChunkDimension` (int3), `Origin` (float3) [S6, unofficial].

### Chunk Data Modification & Queue Management
Processes chunk modifications, expands voxel arrays if sparse, updates block types, tracks adjacent chunks for mesh regeneration, and queues affected chunks for mesh loading [S7, unofficial].
*   **Steps:**
    1.  Retrieve mutable chunk data. If `None`, skip [S7, unofficial].
    2.  If voxel array length is 1, expand to full chunk size (`CHUNK_SIZE_I32^3`) by duplicating the single block data [S7, unofficial].
    3.  Update voxel block type at calculated index: `i = vec3_to_index(local_pos, 32)` [S7, unofficial].
    4.  If modification is at chunk edge, add adjacent chunk to `adj_chunk_set` [S7, unofficial].
    5.  Queue adjacent chunks and current chunk for mesh loading [S7, unofficial].

## Operating characteristics
*   **Time Complexity:** Access is $O(1)$ for both get and set operations [S1, S4, unofficial]. Meshing is $O(N)$ where $N$ is the number of voxels in the chunk, but only performed on modified chunks [S5, unofficial].
*   **Memory Complexity:** $O(C_x \cdot C_y \cdot C_z)$ per chunk. For a $32^3$ chunk, this is 32,768 bytes (if 1 byte per voxel) [S2, S3, unofficial]. Total memory scales with the number of active chunks, not the total world volume [S5, unofficial].
*   **Measured Performance:** Using $16 \times 256 \times 16$ chunks keeps data small and re-meshing fast [S4, unofficial]. Greedy meshing can reduce triangle count by 10× [S4, unofficial].
*   **Strengths:** Efficient meshing (only changed chunks are regenerated), easy serialization (save/load individual chunks), spatial locality for cache performance [S5, unofficial].
*   **Limitations:** Fixed resolution within a chunk (cannot have high-res caves and low-res sky in the same chunk without sub-chunking), boundary handling complexity for meshing [S5, S7, unofficial].
*   **When to use:** Voxel-based games, terrain generation, real-time editing [S3, S5, unofficial].
*   **When not to use:** Highly sparse data with large empty volumes (use SVO), or when variable resolution is critical without sub-chunking overhead [S4, unofficial].

## Background
The chunk-based approach is foundational in voxel engine development, popularized by games like Minecraft [S4, S5, unofficial]. It allows for efficient meshing and world serialization by grouping blocks into manageable units [S5, unofficial]. Early implementations used fixed-size chunks (e.g., $16 \times 16 \times 16$) to balance memory usage and meshing speed [S5, unofficial]. Modern implementations often use larger or non-cubic chunks (e.g., $16 \times 256 \times 16$) to optimize for height and rendering performance [S4, unofficial]. The technique is widely used in both CPU-based (Unity Jobs) and GPU-based (Compute Shaders) voxel engines [S3, unofficial].

## Sources
[S1] vercidium-patreon/meshing, https://github.com/vercidium-patreon/meshing, fetched 2026-10-26 (unofficial)
[S2] m2-md/voxel-greedy-meshing-typescript, https://github.com/m2-md/voxel-greedy-meshing-typescript, fetched 2026-10-26 (unofficial)
[S3] Add foundational voxel engine: generic, terrain, and smooth voxel support, https://github.com/Five-Squared-Interactive/StraightFour/pull/40, fetched 2026-10-26 (unofficial)
[S4] Build a 3D Voxel Engine in WebGL | Tutorials, https://www.mysimulator.uk/content/tutorials/voxel-engine.html, fetched 2026-10-27 (unofficial)
[S5] kotychok/mutiny, https://github.com/kotychok/mutiny, fetched 2026-10-27 (unofficial)
[S6] 2JacT/Marching-Cubes-Terrain, https://github.com/2JacT/Marching-Cubes-Terrain, fetched 2026-10-27 (unofficial)
[S7] Binary Greedy Voxel Meshing Algorithm, https://github.com/TanTanDev/binary_greedy_mesher_demo, fetched 2026-10-26 (unofficial)
