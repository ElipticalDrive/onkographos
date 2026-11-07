# Chunk-Based World Storage

Also called: Chunk Class, Chunk Voxel Storage, Chunk-Based Voxel Storage, Chunk-Based Voxel Storage and Access, Chunked Voxel Storage, Voxel Chunking, Voxel Container, voxel chunk system

## Overview
Chunk-Based World Storage maps world voxel coordinates to fixed-size 3D blocks (chunks) to enable direct coordinate-based access to voxel data [S1, unofficial][S2, unofficial][S3, unofficial]. It solves the problem of managing infinite or large voxel worlds by dividing the space into manageable units, allowing efficient lookup and modification of individual voxels without scanning the entire world [S1, unofficial][S2, unofficial][S3, unofficial]. It differs from [Sparse Voxel Octree Representation](2340-sparse-voxel-octree-representation.md) by using fixed-size dense arrays per chunk rather than hierarchical tree structures [S2, unofficial].

## How it works
**Inputs:** World voxel coordinates $(x, y, z)$, chunk dimensions $CHUNK\_SIZE$, world bounds $WORLD\_W, WORLD\_H, WORLD\_D$ (or $WORLD\_SIZE$ for 2D layouts) [S1, unofficial][S2, unofficial][S4, unofficial].
**Outputs:** Voxel value (e.g., block ID) or status (solid/empty) [S2, unofficial][S3, unofficial].

**Data Structures:**
1.  **Chunk:** Contains an origin vector $O = (ox, oy, oz)$ and a fixed-size 3D voxel array $V$ of dimensions $Width \times Height \times Depth$ [S2, unofficial][S3, unofficial].
2.  **World Storage:** A collection of Chunks stored in a flat array indexed by chunk coordinates or a hash map [S1, unofficial][S2, unofficial].
    *   3D flat array index formula: $Index = cx + WORLD\_W \cdot cz + WORLD\_AREA \cdot cy$, where $WORLD\_AREA = WORLD\_W \cdot WORLD\_D$ [S1, unofficial].
    *   2D flat array index formula: $Index = chunkX + (chunkZ \cdot WORLD\_SIZE)$ [S4, unofficial].
3.  **Voxel Index in Flat Vector:** $index = z \cdot Height \cdot Width + y \cdot Width + x$ [S2, unofficial].

**Algorithm: Lookup Voxel at $(x, y, z)$**
1.  **Calculate Chunk Coordinates:** Compute chunk indices $(cx, cy, cz)$ using integer division [S1, unofficial]. Standard formula: $cx = \lfloor x / CHUNK\_SIZE \rfloor$, $cy = \lfloor y / CHUNK\_SIZE \rfloor$, $cz = \lfloor z / CHUNK\_SIZE \rfloor$ [S1, unofficial]. For negative coordinates, use $cx = \lfloor (x - CHUNK\_SIZE + 1) / CHUNK\_SIZE \rfloor$ to ensure correct floor behavior [S4, unofficial].
2.  **Check World Bounds:** Verify $0 \le cx < WORLD\_W$, $0 \le cy < WORLD\_H$, $0 \le cz < WORLD\_D$ (or equivalent bounds) [S1, unofficial]. If out of bounds, return void/empty [S1, unofficial].
3.  **Retrieve Chunk:** Calculate the chunk's flat index using the chosen layout formula [S1, unofficial][S4, unofficial]. Retrieve the Chunk object $C$ from the world storage [S2, unofficial]. If no chunk exists at this location, return false/empty [S3, unofficial].
4.  **Calculate Local Coordinates:** Compute local voxel indices $(lx, ly, lz)$ within the chunk [S1, unofficial]. Formula: $lx = x - (cx \cdot CHUNK\_SIZE)$, $ly = y - (cy \cdot CHUNK\_SIZE)$, $lz = z - (cz \cdot CHUNK\_SIZE)$ [S1, unofficial][S3, unofficial]. Alternatively using modulo: $lx = x \% CHUNK\_SIZE$ [S4, unofficial]. Handle negative modulo: if $lx < 0$, $lx += CHUNK\_SIZE$ [S4, unofficial].
5.  **Check Local Bounds:** Verify $0 \le lx < CHUNK\_SIZE$, $0 \le ly < CHUNK\_SIZE$, $0 \le lz < CHUNK\_SIZE$ [S3, unofficial].
6.  **Return Voxel Value:** Access the voxel array: $V[lx][ly][lz]$ [S2, unofficial]. Determine solidity: Return $V[lx][ly][lz] \neq 0$ [S3, unofficial].

**Algorithm: Modify Voxel at $(x, y, z)$ to Value $v$**
1.  Perform steps 1-4 of Lookup [S2, unofficial].
2.  Set $C.V[lx][ly][lz] = v$ [S2, unofficial].
3.  **Boundary Mesh Update:** If the modified voxel is on a chunk boundary ($lx=0$ or $lx=CHUNK\_SIZE-1$, $ly=0$ or $ly=CHUNK\_SIZE-1$, $lz=0$ or $lz=CHUNK\_SIZE-1$), trigger mesh update on the adjacent neighbor chunk to expose/hide shared faces [S4, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def get_voxel(world, x, y, z, chunk_size, world_w, world_h, world_d):
    # 1. Calculate chunk coordinates with correct floor for negatives
    cx = (x - chunk_size + 1) // chunk_size if x < 0 else x // chunk_size
    cy = (y - chunk_size + 1) // chunk_size if y < 0 else y // chunk_size
    cz = (z - chunk_size + 1) // chunk_size if z < 0 else z // chunk_size

    # 2. Check world bounds
    if not (0 <= cx < world_w and 0 <= cy < world_h and 0 <= cz < world_d):
        return 0  # Void

    # 3. Retrieve chunk
    # WORLD_AREA = world_w * world_d
    # chunk_index = cx + world_w * cz + WORLD_AREA * cy
    chunk = world.get_chunk(cx, cy, cz)
    
    if chunk is None:
        return 0  # Chunk does not exist

    # 4. Calculate local coordinates
    lx = x - cx * chunk_size
    ly = y - cy * chunk_size
    lz = z - cz * chunk_size
    
    # 5. Check local bounds
    if not (0 <= lx < chunk_size and 0 <= ly < chunk_size and 0 <= lz < chunk_size):
        return 0

    # 6. Return voxel value
    return chunk.voxels[lx][ly][lz]

def set_voxel(world, x, y, z, value, chunk_size, world_w, world_h, world_d):
    cx = (x - chunk_size + 1) // chunk_size if x < 0 else x // chunk_size
    cy = (y - chunk_size + 1) // chunk_size if y < 0 else y // chunk_size
    cz = (z - chunk_size + 1) // chunk_size if z < 0 else z // chunk_size

    if not (0 <= cx < world_w and 0 <= cy < world_h and 0 <= cz < world_d):
        return

    chunk = world.get_chunk(cx, cy, cz)
    
    if chunk is None:
        chunk = world.create_chunk(cx, cy, cz)

    lx = x - cx * chunk_size
    ly = y - cy * chunk_size
    lz = z - cz * chunk_size

    if 0 <= lx < chunk_size and 0 <= ly < chunk_size and 0 <= lz < chunk_size:
        chunk.voxels[lx][ly][lz] = value
        
        # Boundary check for neighbor updates
        if lx == 0 or lx == chunk_size - 1 or \
           ly == 0 or ly == chunk_size - 1 or \
           lz == 0 or lz == chunk_size - 1:
            world.mark_neighbors_dirty(cx, cy, cz, chunk_size)
```

## Subtopics

### [Voxel World Storage](3984-voxel-world-storage.md)
Voxel World Storage is a dense grid data structure that maintains block types for each chunk in world coordinates, enabling direct ray querying of the voxel space. It solves the problem of selecting or interacting with specific voxels in a scene that lacks traditional mesh colliders by providing a coordinate-mapped lookup of block types.

### [Chunk Loading and Unloading via Hashmap and Area of Interest](4766-chunk-loading-and-unloading-via-hashmap-and-area-of-interest.md)
This method manages the lifecycle of voxel chunks by maintaining a hash map of active chunks and culling those outside a defined Area of Interest (AoI) centered on the camera. It solves the problem of memory exhaustion and rendering overhead in large voxel worlds by ensuring only visible or nearby geometry is resident in memory. It differs from static world storage by dynamically allocating and deallocating chunk objects based on camera position.

### [Chunk Coordinate Mapping and World Bounds](7013-chunk-coordinate-mapping-and-world-bounds.md)
Converts between chunk grid indices and world coordinates, and calculates the total world width covered by the chunk system.

### [Chunk-based Voxel World Management](7997-chunk-based-voxel-world-management.md)
A system for managing a voxel world by dividing it into fixed-size chunks, handling visibility culling against a frustum, and incrementally rebuilding chunks that have changed.

### Lossless Sparse Undo/Redo via Chunk Snapshots
An undo/redo mechanism for sparse voxel volumes that snapshots entire chunks rather than individual cell deltas to ensure lossless restoration [S7, unofficial]. When an edit occurs, the entire state of affected chunks is snapshot [S7, unofficial]. During undo, if an edit created a new chunk, the chunk is removed entirely rather than leaving an empty allocation [S7, unofficial]. Only snapshots for chunks whose cells actually changed are kept, keeping memory usage proportional to the edit [S7, unofficial].

### Signed 64-bit World Coordinate System
A coordinate system using signed 64-bit integers for world positions, allowing correct handling of negative section borders and overflow detection during neighbor offset calculations [S5, unofficial]. It defines `LocalPos` (0-15), `SectionPos` (world space section coordinates), and `WorldPos` (absolute i64 coordinates) [S5, unofficial]. The constant `SECTION_EDGE` represents the edge length of a section (16) [S5, unofficial]. Conversion between local and world coordinates, and calculation of neighbor offsets, must detect integer overflow to prevent incorrect section addressing [S5, unofficial].

### Chunking
A technique for managing large voxel worlds by dividing them into chunks, trading faster meshing for more draw calls [S6, unofficial]. Minecraft uses chunk dimensions of 16x16x256, divided vertically [S6, unofficial]. Demo engines often use arbitrary dimensions like 32x32x32 [S6, unofficial].

## Operating characteristics
**Time Complexity:** Lookup and modification are $O(1)$ with respect to world size, dependent only on chunk size [S1, unofficial][S2, unofficial]. Hash map lookups are average $O(1)$, while flat array access is $O(1)$ with bounds checks [S1, unofficial][S3, unofficial].
**Memory Complexity:** $O(N)$ where $N$ is the number of active chunks. Each chunk consumes $CHUNK\_SIZE^3$ voxels [S2, unofficial]. Sparse implementations only store non-empty chunks, reducing memory usage [S4, unofficial].
**Performance:** Fixed-size chunks optimize memory locality and cache performance [S2, unofficial][S3, unofficial]. Meshing is faster per chunk due to fixed boundaries, but results in more draw calls compared to global meshing [S6, unofficial].
**Strengths:** Simple coordinate translation, efficient memory locality, easy implementation of sparse storage via hash maps [S1, unofficial][S2, unofficial].
**Limitations:** Boundary handling requires neighbor chunk access for mesh updates [S3, unofficial]. Fixed chunk size may lead to memory waste if chunks are mostly empty (unless sparse) [S4, unofficial]. Integer overflow risks in large worlds without 64-bit coordinates [S5, unofficial].

## Background
Chunk-based storage is widely used in voxel games like Minecraft, which popularized the 16x16x256 chunk structure [S6, unofficial]. It is widely used in voxel engines for efficient world management and rendering [S2, unofficial][S3, unofficial]. The method is fundamental to many open-world voxel simulations, enabling infinite or large-scale worlds [S1, unofficial][S3, unofficial].

## Sources
[S1] StanislavPetrovV/Minecraft, https://github.com/StanislavPetrovV/Minecraft, fetched 2026-10-26 (unofficial)
[S2] JamesRandall/SimpleVoxelEngine, https://github.com/JamesRandall/SimpleVoxelEngine, fetched 2026-10-26 (unofficial)
[S3] whudsontosh/CPU-Rasterizer, https://github.com/whudsontosh/CPU-Rasterizer, fetched 2026-10-27 (unofficial)
[S4] GitHub - ernerdgn/voxel-engine: A voxel engine written from scratch in ..., https://github.com/ernerdgn/voxel-engine, fetched 2026-10-29 (unofficial)
[S5] PRoGG1111/voxel-forge, https://github.com/PRoGG1111/voxel-forge, fetched 2026-10-26 (unofficial)
[S6] Presentation on the noa voxel game engine, made and presented in-engine (2015), http://andyhall.github.io/noa-lt/, fetched 2026-10-26 (unofficial)
[S7] feat(terrain): DDA voxel raycast + editing brushes with undo/redo, https://github.com/drsnuggles8/OloEngineBase/pull/973, fetched 2026-10-25 (unofficial)
