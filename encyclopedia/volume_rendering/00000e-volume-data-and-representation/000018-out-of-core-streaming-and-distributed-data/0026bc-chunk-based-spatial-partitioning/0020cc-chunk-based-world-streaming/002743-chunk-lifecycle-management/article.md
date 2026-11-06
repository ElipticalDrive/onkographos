# Chunk Lifecycle Management

Also called: Chunk Management, Chunk Spawning and Despawn

## Overview
Chunk Lifecycle Management is the core synchronization mechanism in chunk-based volume streaming that ensures the set of loaded chunks matches the set of chunks required by the current camera position. It solves the problem of memory leaks and visual pop-in by explicitly identifying which chunks must be spawned (loaded) and which must be despawned (unloaded) each frame. It sets itself apart from [Synchronous Chunk Loading](3904-synchronous-chunk-loading.md) by focusing on the *decision* logic of what to load/unload rather than the blocking I/O operations, and from [Chunk-Based World Management & Asynchronous Loading](6440-chunk-based-world-management-asynchronous-loading.md) by providing the foundational state-transition logic that higher-level managers orchestrate.

## How it works
**Inputs:**
- Current camera/player position.
- Render distance (radius) defining the visible volume.
- Current world state: a collection of existing chunk objects and their positions.

**Outputs:**
- A list of chunks to spawn (newly created objects).
- A list of chunks to despawn (objects to destroy).
- Updated world state containing only chunks within the render distance.

**Data Structures:**
- `chunks`: A hash map (dictionary) mapping `ChunkPosition` (key) to `Chunk` object (value).
- `nearbyPositionsSet`: A hash set of `ChunkPosition` objects representing all chunk coordinates within the render distance.
- `chunksAlreadyExist`: A hash set of `ChunkPosition` objects representing keys present in `chunks`.

**Steps:**
1. **Identify Nearby Positions:** Calculate all integer chunk coordinates within the spherical render distance centered on the player's current position. Store these in `nearbyPositionsSet` [S1, unofficial].
2. **Identify Existing Chunks:** Populate `chunksAlreadyExist` with all keys currently present in the `chunks` hash map [S1, unofficial].
3. **Despawn Logic:** Iterate through `chunksAlreadyExist`. For each position:
   - If the position is *not* in `nearbyPositionsSet`, the chunk is outside the render distance.
   - Remove the chunk object from the `chunks` map.
   - Remove the chunk from frame draw calls (e.g., render list).
   - Add the position to a `chunksToDelete` list for cleanup [S1, unofficial].
4. **Spawn Logic:** Iterate through `nearbyPositionsSet`. For each position:
   - If the position is *not* in `chunksAlreadyExist`, the chunk does not exist.
   - Create a new `Chunk` object.
   - Insert the object into the `chunks` hash map with the position as the key.
   - Add the new chunk to a `newChunks` list [S1, unofficial].
5. **Generation:** For all chunks in `newChunks`, trigger terrain or data generation (e.g., `Chunk::MultithreadGenerateTerrain`) [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def manage_chunk_lifecycle(player_pos, render_distance, chunks):
    # Step 1: Identify nearby positions
    nearby_positions = get_chunk_positions_within_radius(player_pos, render_distance)
    nearby_positions_set = set(nearby_positions)
    
    # Step 2: Identify existing chunks
    existing_positions = set(chunks.keys())
    
    chunks_to_delete = []
    new_chunks = []
    
    # Step 3: Despawn logic
    for pos in existing_positions:
        if pos not in nearby_positions_set:
            # Remove from world
            chunk_obj = chunks.pop(pos)
            remove_from_draw_list(chunk_obj)
            chunks_to_delete.append(pos)
            
    # Step 4: Spawn logic
    for pos in nearby_positions:
        if pos not in existing_positions:
            # Create new chunk
            new_chunk = create_chunk(pos)
            chunks[pos] = new_chunk
            new_chunks.append(new_chunk)
            
    # Step 5: Generation
    for chunk in new_chunks:
        chunk.generate_terrain_multithreaded()
        
    return chunks_to_delete, new_chunks
```

**Parameters:**
- `render_distance`: The radius in chunk units. Typical values depend on the game's scale and hardware capabilities [S1, unofficial].

**Edge Cases:**
- **Precision:** Floating-point player positions must be correctly converted to integer chunk coordinates to avoid jitter or missed chunks at boundaries [G].
- **Degenerate Input:** If `render_distance` is zero, all chunks outside the immediate player chunk should be despawned [G].

## Subtopics

### [Dynamic Chunk Generation with Radius](4002-dynamic-chunk-generation-with-radius.md)
Manages chunk instantiation and removal based on a spherical render distance around the player, using set difference operations to determine which chunks to load or unload. It builds on [Chunk Lifecycle Management](3987-chunk-lifecycle-management.md) by replacing static or scanner-based triggers with a continuous distance check against the player's current position. Unlike [Chunk Loading/Unloading Scanner](4404-chunk-loading-unloading-scanner.md), which scans discrete regions, this method evaluates every chunk within a calculated bounding box relative to the player to maintain a consistent spherical visibility volume.

### [Chunk Loading/Unloading Scanner](4404-chunk-loading-unloading-scanner.md)
The Chunk Loading/Unloading Scanner is a spatial management technique that determines which chunks of a volume dataset need to be loaded into memory or unloaded based on the camera's current and previous positions. It solves the problem of efficiently streaming large voxel worlds by identifying a "load area" and an "unload area" using offset grids, then computing the symmetric difference between these areas to generate precise queue updates. Unlike [Dynamic Chunk Generation with Radius](4002-dynamic-chunk-generation-with-radius.md), which focuses on procedural generation triggers, or [Budget Plans](5497-budget-plans.md), which manage resource limits, the Scanner focuses on the geometric transition logic between camera states. It differs from [Chunk Visibility Culling](5031-chunk-visibility-culling.md) by operating on a flat list of chunks and performing binary load/unload decisions based on distance thresholds rather than frustum intersection tests.

### [Budget Plans](5497-budget-plans.md)
Budget Plans provide a system for managing chunk loading policies in chunk-based volume streaming, replacing the previous single octree-focused policy with configurable presets for exploration, whole-volume visualization, and frame-rate maintenance. This method allows an application to switch loading strategies mid-session to balance visual fidelity against performance constraints, addressing the limitation where a single policy was "right for exploring and wrong for a whole-volume figure or for holding a frame rate while rotating". It sets itself apart from sibling methods like [Dynamic Chunk Generation with Radius](4002-dynamic-chunk-generation-with-radius.md) by focusing on high-level policy presets and precedence rules rather than geometric radius calculations, and from [Chunk Loading/Unloading Scanner](4404-chunk-loading-unloading-scanner.md) by managing the *intent* of the load queue rather than the mechanical scanning of the spatial hierarchy.

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of chunks in the render distance, as it iterates through existing and nearby positions [S1, unofficial].
- **Memory Complexity:** $O(M)$ where $M$ is the number of loaded chunks, storing chunk objects and position sets [S1, unofficial].
- **Strengths:** Simple to implement; guarantees no chunks are loaded outside the render distance; prevents memory leaks by explicit despawning [S1, unofficial].
- **Limitations:** Can cause "pop-in" if generation is slow; does not account for occlusion or frustum culling beyond distance; may load chunks that are not immediately visible [G].
- **When to use:** Ideal for simple voxel worlds where render distance is the primary culling factor [S1, unofficial].
- **When not to use:** When complex visibility culling or budget constraints are required [S1, unofficial].

## Background
Originated in open-source voxel engine implementations, specifically cited in the `Cube` repository by gabkhanfig [S1, unofficial]. It represents a fundamental approach to chunk management before more advanced asynchronous or budget-based systems were introduced.

## Sources
[S1] gabkhanfig/Cube, https://github.com/gabkhanfig/Cube, fetched 2026-10-27 (unofficial)
