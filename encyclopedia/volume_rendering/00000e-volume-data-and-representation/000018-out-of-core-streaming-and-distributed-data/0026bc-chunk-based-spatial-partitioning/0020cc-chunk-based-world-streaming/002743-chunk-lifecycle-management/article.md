# Chunk Lifecycle Management

Also called: Chunk Management, Chunk Spawning and Despawn

## Overview
Chunk Lifecycle Management is the core mechanism for maintaining the set of active volumetric chunks in a streaming world, ensuring that only chunks within the player's vicinity are resident in memory. It solves the problem of memory exhaustion and unnecessary I/O by explicitly spawning new chunks as the player approaches and despawning distant chunks as they move away. It differs from [Dynamic Chunk Generation with Radius](4002-dynamic-chunk-generation-with-radius.md) by serving as the foundational logic for set-difference operations, whereas the radius variant optimizes the geometric query; it differs from [Chunk Loading/Unloading Scanner](4404-chunk-loading-unloading-scanner.md) by focusing on the state transition of individual chunk objects rather than scanning discrete spatial regions for load/unload triggers.

## How it works
**Inputs:**
- Player position (or camera position) in world coordinates.
- Render distance radius $R$.
- Current set of existing chunks, stored in a hash map `chunks` mapping `ChunkPosition` to `Chunk` object.
- Chunk grid resolution (implicit in `ChunkPosition` definition).

**Outputs:**
- Updated `chunks` hash map with new entries added and removed entries deleted.
- List of `newChunks` for terrain generation.
- List of `chunksToDelete` for cleanup.

**Data Structures:**
- `ChunkPosition`: A tuple or struct representing the integer grid coordinates $(x, y, z)$ of a chunk.
- `chunks`: A hash map (dictionary) where keys are `ChunkPosition` and values are `Chunk` objects.
- `_nearbyPositionsSet`: A set of `ChunkPosition` objects calculated to be within the render distance.
- `_chunksAlreadyExist`: A set of `ChunkPosition` keys extracted from the `chunks` hash map.

**Steps:**
1. **Calculate Nearby Positions:** Determine all integer chunk positions $(x, y, z)$ that fall within a spherical volume of radius $R$ centered at the player's position. Store these in `_nearbyPositionsSet` [S1, unofficial].
2. **Identify Existing Chunks:** Extract all keys from the `chunks` hash map into `_chunksAlreadyExist` [S1, unofficial].
3. **Despawn Distant Chunks:** Iterate through each position $p$ in `_chunksAlreadyExist`. If $p$ is not present in `_nearbyPositionsSet`, the chunk is too far away. Remove the chunk object from the `chunks` hash map, remove it from frame draw calls, and add $p$ to `chunksToDelete` [S1, unofficial].
4. **Spawn Nearby Chunks:** Iterate through each position $p$ in `_nearbyPositionsSet`. If $p$ is not present in `_chunksAlreadyExist`, the chunk does not exist. Create a new `Chunk` object, insert it into the `chunks` hash map with key $p$, and add the new chunk to `newChunks` [S1, unofficial].
5. **Generate Terrain:** For every chunk in `newChunks`, initiate asynchronous or synchronous terrain generation using `Chunk::MultithreadGenerateTerrain` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def manage_chunk_lifecycle(player_pos, render_distance, chunks):
    # Step 1: Identify nearby chunk positions
    # Assumes a function that returns all integer grid positions within radius R of player_pos
    nearby_positions = ChunkPositionsWithinRenderDistance(player_pos, render_distance)
    nearby_set = set(nearby_positions)
    
    # Step 2: Get positions of chunks that currently exist
    existing_positions = set(chunks.keys())
    
    chunks_to_delete = []
    new_chunks = []
    
    # Step 3: Iterate through existing chunks; delete if not nearby
    for pos in existing_positions:
        if pos not in nearby_set:
            # Remove from hash map and draw calls
            del chunks[pos]
            chunks_to_delete.append(pos)
            
    # Step 4: Iterate through nearby positions; create if not existing
    for pos in nearby_positions:
        if pos not in existing_positions:
            # Create new chunk object
            new_chunk = Chunk(pos)
            chunks[pos] = new_chunk
            new_chunks.append(new_chunk)
            
    # Step 5: Generate terrain for new chunks
    for chunk in new_chunks:
        chunk.MultithreadGenerateTerrain()
        
    return chunks_to_delete, new_chunks
```

**Parameters:**
- `render_distance`: The radius $R$ in world units. Typical values depend on the scale of the volume and performance constraints. Not given in the sources.

**Edge Cases and Pitfalls:**
- **Precision:** Floating-point player positions must be correctly converted to integer chunk grid coordinates. [G] Ensure consistent rounding or floor/ceil logic when mapping world coordinates to chunk indices.
- **Boundaries:** If the volume has hard boundaries, `ChunkPositionsWithinRenderDistance` must clamp results to valid volume extents. Not given in the sources.
- **Degenerate Input:** If `render_distance` is zero, only the chunk containing the player should exist. Not given in the sources.

## Subtopics

### [Dynamic Chunk Generation with Radius](4002-dynamic-chunk-generation-with-radius.md)
Manages chunk instantiation and removal based on a spherical render distance around the player, using set difference operations to determine which chunks to load or unload. It builds on [Chunk Lifecycle Management](3987-chunk-lifecycle-management.md) by replacing static or scanner-based triggers with a continuous distance check against the player's current position. Unlike [Chunk Loading/Unloading Scanner](4404-chunk-loading-unloading-scanner.md), which scans discrete regions, this method evaluates every chunk within a calculated bounding box relative to the player to maintain a consistent spherical visibility volume.

### [Chunk Loading/Unloading Scanner](4404-chunk-loading-unloading-scanner.md)
The Chunk Loading/Unloading Scanner identifies which chunks in a volumetric world need to be loaded or unloaded based on the camera's current and previous positions, using offset grids to define load and unload areas. It differs from [Dynamic Chunk Generation with Radius](4002-dynamic-chunk-generation-with-radius.md) by explicitly tracking state transitions between frames via previous position comparison, and from [Budget Plans](5497-budget-plans.md) by using a fixed-radius offset strategy rather than a dynamic priority queue.

### [Budget Plans](5497-budget-plans.md)
Budget Plans provide a system for managing chunk loading policies in chunk-based volume streaming, replacing the previous single octree-focused policy with configurable presets for exploration, whole-volume visualization, and frame-rate maintenance. This method allows an application to switch loading strategies mid-session to balance visual fidelity against performance constraints, addressing the limitation where a single policy was "right for exploring and wrong for a whole-volume figure or for holding a frame rate while rotating". It sets itself apart from sibling methods like [Dynamic Chunk Generation with Radius](4002-dynamic-chunk-generation-with-radius.md) by focusing on high-level policy presets and precedence rules rather than geometric radius calculations, and from [Chunk Loading/Unloading Scanner](4404-chunk-loading-unloading-scanner.md) by managing the *intent* of the load queue rather than the mechanical scanning of the spatial hierarchy.

## Operating characteristics
- **Time Complexity:** $O(N + M)$, where $N$ is the number of existing chunks and $M$ is the number of chunks within the render distance. Set lookups are $O(1)$ on average.
- **Memory Complexity:** Proportional to the number of active chunks. The hash map stores references to all active chunks.
- **Strengths:** Simple logic, easy to implement, deterministic behavior based on distance.
- **Limitations:** Does not account for occlusion or view frustum culling beyond the radius. May load chunks that are not visible if the radius is large. Not given in the sources for specific performance metrics.
- **When to use:** When a simple, distance-based streaming strategy is sufficient and performance constraints allow for loading all chunks within a fixed radius.
- **When not to use:** When occlusion culling is critical, or when memory is extremely constrained and a more sophisticated priority-based system (like [Budget Plans](5497-budget-plans.md)) is needed.

## Background
Originating from the Cube project, this method provides a basic but effective framework for managing chunk lifecycles in volumetric worlds [S1, unofficial]. It serves as the foundational logic for more advanced streaming techniques that build upon set-difference operations to optimize chunk loading and unloading.

## Sources
[S1] gabkhanfig/Cube, https://github.com/gabkhanfig/Cube, fetched 2026-10-27 (unofficial)
