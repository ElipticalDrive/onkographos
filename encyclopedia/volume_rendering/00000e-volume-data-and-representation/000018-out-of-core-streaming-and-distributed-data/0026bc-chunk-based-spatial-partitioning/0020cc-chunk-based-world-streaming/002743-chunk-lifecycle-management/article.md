# Chunk Lifecycle Management

Also called: Chunk Management, Chunk Spawning and Despawn

## Overview
Chunk Lifecycle Management is the core synchronization mechanism in chunk-based volume streaming that ensures the set of loaded chunks matches the set of chunks required by the current camera position [G]. It solves the problem of memory leaks and visual pop-in by explicitly identifying which chunks must be spawned (loaded) and which must be despawned (unloaded) each frame [G]. It sets itself apart from [Synchronous Chunk Loading](3904-synchronous-chunk-loading.md) by focusing on the decision logic of what to load/unload rather than the blocking I/O operations, and from [Chunk-Based World Management & Asynchronous Loading](6440-chunk-based-world-management-asynchronous-loading.md) by providing the foundational state-transition logic that higher-level managers orchestrate [G].

## How it works
**Inputs:**
- Current camera/player position [S1, unofficial].
- Render distance (radius) defining the visible volume [S1, unofficial].
- Current world state: a collection of existing chunk objects and their positions [S1, unofficial].

**Outputs:**
- A list of chunks to spawn (newly created objects) [S1, unofficial].
- A list of chunks to despawn (objects to destroy) [S1, unofficial].
- Updated world state containing only chunks within the render distance [S1, unofficial].

**Data Structures:**
- `chunks`: A hash map mapping `ChunkPosition` (key) to `Chunk` object (value) [S1, unofficial].
- `_nearbyPositionsSet`: A hash set of `ChunkPosition` objects representing all chunk coordinates within the render distance [S1, unofficial].
- `_chunksAlreadyExist`: A hash set of `ChunkPosition` objects representing positions of chunks that currently exist in the world [S1, unofficial].
- `chunksToDelete`: A list of positions marked for removal [S1, unofficial].
- `newChunks`: A list of newly created chunk objects [S1, unofficial].

**Steps:**
1. Identify nearby chunk positions using `ChunkPositionsWithinRenderDistance` [S1, unofficial].
2. Populate `_nearbyPositionsSet` with all nearby positions and `_chunksAlreadyExist` with positions of chunks that currently exist in the world [S1, unofficial].
3. Iterate through existing chunks; if a chunk's position is not in `_nearbyPositionsSet`, delete the chunk object, remove it from frame draw calls, and add its position to `chunksToDelete` [S1, unofficial].
4. Iterate through `nearbyPositions`; if a chunk does not already exist, create a new `Chunk` object, insert it into the `chunks` hashmap, and add it to `newChunks` [S1, unofficial].
5. Generate terrain for all new chunks using `Chunk::MultithreadGenerateTerrain` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Chunk:
    def __init__(self, pos):
        self.pos = pos
    def generate_terrain_multithreaded(self):
        pass

def manage_chunk_lifecycle(player_pos, render_distance, chunks, render_list):
    # Step 1: Identify nearby positions
    nearby_positions = []
    r_int = int(render_distance)
    for x in range(-r_int, r_int + 1):
        for y in range(-r_int, r_int + 1):
            for z in range(-r_int, r_int + 1):
                if x*x + y*y + z*z <= render_distance*render_distance:
                    nearby_positions.append((x, y, z))

    _nearbyPositionsSet = set(nearby_positions)
    _chunksAlreadyExist = set(chunks.keys())

    chunks_to_delete = []
    new_chunks = []

    # Step 3: Despawn logic
    for pos in _chunksAlreadyExist:
        if pos not in _nearbyPositionsSet:
            chunk_obj = chunks.pop(pos)
            render_list.discard(chunk_obj)
            chunks_to_delete.append(pos)

    # Step 4: Spawn logic
    for pos in nearby_positions:
        if pos not in _chunksAlreadyExist:
            new_chunk = Chunk(pos)
            chunks[pos] = new_chunk
            new_chunks.append(new_chunk)

    # Step 5: Generation
    for chunk in new_chunks:
        chunk.generate_terrain_multithreaded()

    return chunks_to_delete, new_chunks
```

**Parameters:**
- `render_distance`: The radius in chunk units defining the visible volume [S1, unofficial].

**Edge Cases:**
- **Precision:** Floating-point player positions must be correctly converted to integer chunk coordinates to avoid jitter or missed chunks at boundaries [G].
- **Degenerate Input:** If `render_distance` is zero, all chunks outside the immediate player chunk should be despawned [G].

## Subtopics

### [Dynamic Chunk Generation with Radius](4002-dynamic-chunk-generation-with-radius.md)
Manages chunk instantiation and removal based on a spherical render distance around the player, using set difference operations to determine which chunks to load or unload. It builds on [Chunk Lifecycle Management](3987-chunk-lifecycle-management.md) by replacing static or scanner-based triggers with a continuous distance check against the player's current position [G]. Unlike [Chunk Loading/Unloading Scanner](4404-chunk-loading-unloading-scanner.md), which scans discrete regions, this method evaluates every chunk within a calculated bounding box relative to the player to maintain a consistent spherical visibility volume [G].

### [Chunk Loading/Unloading Scanner](4404-chunk-loading-unloading-scanner.md)
The Chunk Loading/Unloading Scanner is a spatial management technique that determines which chunks of a volume dataset need to be loaded into memory or unloaded based on the camera's current and previous positions. It solves the problem of efficiently streaming large voxel worlds by identifying a load area and an unload area using offset grids, then computing the symmetric difference between these areas to generate precise queue updates. It differs from [Chunk Visibility Culling](5031-chunk-visibility-culling.md) by operating on loaded chunks and using distance thresholds rather than frustum intersection tests.

### [Budget Plans](5497-budget-plans.md)
Budget Plans manage chunk loading policies for chunked volumes, replacing the single octree-focused policy with presets for exploration, whole-volume figures, and frame-rate holding. Unlike [Dynamic Chunk Generation with Radius](4002-dynamic-chunk-generation-with-radius.md) which focuses on geometric radius, or [Chunk Loading/Unloading Scanner](4404-chunk-loading-unloading-scanner.md) which manages eviction, Budget Plans define the target resident count and detail level based on user intent.

## Operating characteristics
Not given in the sources.

## Background
Not given in the sources.

## Sources
[S1] gabkhanfig/Cube, https://github.com/gabkhanfig/Cube, fetched 2026-10-27 (unofficial)
