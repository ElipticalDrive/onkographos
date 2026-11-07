# Chunk-Based World Management & Asynchronous Loading

Also called: Async Chunk Streaming, Chunk Culling and Unloading, chunk garbage collection, far chunk removal, Planet Chunk Management and Rendering

## Overview
This method manages a voxel world by dividing it into fixed-size chunks, loading them asynchronously based on player proximity, and removing distant chunks to manage memory and rendering load [S1, unofficial]. It sets itself apart from synchronous loading by processing a limited number of pending chunks per frame, preventing frame drops during high-demand generation phases [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Player position, configuration options (chunk size, generation function, async flag), and a list of pending chunk coordinates [S1, unofficial].
*   **Outputs:** Updated scene graph with visible chunk meshes, updated internal chunk storage, and emitted events for chunk removal and rendering [S1, unofficial].

**Data Structures:**
*   `pendingChunks`: An array of strings representing chunk coordinates, formatted as `"x|y|z"` [S1, unofficial].
*   `self.voxels.chunks`: A collection storing active chunk data and associated Three.js meshes [S1, unofficial].
*   `opts.chunkSize`: Integer, typically 32 [S1, unofficial].

**Parameters:**
*   `removeDistance`: Maximum distance from the player to keep chunks loaded [S1, unofficial].
*   `asyncChunkGeneration`: Boolean flag. If `false`, loads all pending chunks synchronously. If `true`, loads a fraction (default 10%) per frame [S1, unofficial].
*   `generateChunks`: Boolean flag. If `false`, chunk generation is disabled [S1, unofficial].
*   `isClient`: Boolean flag. Determines if rendering logic (`showChunk`) should be executed [S1, unofficial].

**Steps:**
1.  **Configuration:**
    *   Set `this.generate` to a sphere world function: $x^2 + y^2 + z^2 \le 15^2$ returns 1, else 0, if not provided in options [S1, unofficial].
    *   Set `this.generateVoxelChunk` handler [S1, unofficial].
    *   Listen for `missingChunk` events. On event, push chunk position to `pendingChunks` and call `loadPendingChunks` [S1, unofficial].
2.  **Load Pending Chunks (`loadPendingChunks`):**
    *   Determine `count` of chunks to process this frame:
        *   If `!asyncChunkGeneration`: `count = pendingChunks.length` [S1, unofficial].
        *   If `asyncChunkGeneration`:
            *   Calculate `count = pendingChunks.length * 0.1` (if `count` is not already set) [S1, unofficial].
            *   Clamp `count`: `count = Math.max(1, Math.min(count, pendingChunks.length))` [S1, unofficial].
    *   Iterate `i` from 0 to `count - 1`:
        *   Parse `chunkPos` from `pendingChunks[i]` by splitting on `'|'` and converting to integers [S1, unofficial].
        *   Call `this.voxels.generateChunk(x, y, z)` to generate the chunk data [S1, unofficial].
        *   If `this.isClient` is true, call `showChunk(chunk)` [S1, unofficial].
    *   Remove processed chunks from `pendingChunks`: `pendingChunks.splice(0, count)` [S1, unofficial].
3.  **Remove Far Chunks (`removeFarChunks`):**
    *   Get `nearbyChunks` using `voxels.nearbyChunks(playerPosition, removeDistance)` [S1, unofficial].
    *   Iterate over `self.voxels.chunks`:
        *   If a chunk is not in `nearbyChunks`:
            *   Remove from `pendingChunks` if present [S1, unofficial].
            *   Dispose of Three.js meshes and geometries [S1, unofficial].
            *   Delete chunk and mesh data from storage [S1, unofficial].
            *   Emit `removeChunk` event [S1, unofficial].
    *   Call `voxels.requestMissingChunks(playerPosition)` to trigger loading of newly adjacent chunks [S1, unofficial].
4.  **Show Chunk (`showChunk`):**
    *   Get bounds of the chunk [S1, unofficial].
    *   Create mesh via `voxelMesh(chunk, mesher, scale, THREE)` [S1, unofficial].
    *   Handle surface mesh and wire mesh creation [S1, unofficial].
    *   Set mesh position to the chunk's world coordinates [G].
    *   Add mesh to scene [S1, unofficial].
    *   Emit `renderChunk` event [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def load_pending_chunks(pending_chunks, voxels, is_client, async_chunk_generation):
    if not async_chunk_generation:
        count = len(pending_chunks)
    else:
        count = len(pending_chunks) * 0.1
        count = max(1, min(count, len(pending_chunks)))

    for i in range(count):
        chunk_pos_str = pending_chunks[i]
        chunk_pos = [int(coord) for coord in chunk_pos_str.split('|')]
        chunk = voxels.generate_chunk(chunk_pos[0], chunk_pos[1], chunk_pos[2])
        if is_client:
            show_chunk(chunk)

    if count > 0:
        del pending_chunks[:count]

def remove_far_chunks(voxels, player_position, remove_distance, pending_chunks):
    nearby_chunks = voxels.nearby_chunks(player_position, remove_distance)
    nearby_keys = set(nearby_chunks)
    
    chunk_keys = list(voxels.chunks.keys())
    for chunk_key in chunk_keys:
        if chunk_key not in nearby_keys:
            if chunk_key in pending_chunks:
                pending_chunks.remove(chunk_key)
            
            chunk_data = voxels.chunks[chunk_key]
            if chunk_data.mesh:
                for vao in chunk_data.mesh.vertex_array_objects:
                    vao.dispose()
                chunk_data.mesh.dispose()
            
            del voxels.chunks[chunk_key]
            voxels.meshes.pop(chunk_key, None)
            emit_event('removeChunk', chunk_key)
            
    voxels.request_missing_chunks(player_position)

def show_chunk(chunk):
    bounds = chunk.get_bounds()
    mesh = create_mesh_from_chunk(chunk) # [G] Standard voxel-to-mesh conversion
    mesh.position = (chunk.x * chunk_size, chunk.y * chunk_size, chunk.z * chunk_size)
    scene.add(mesh)
    emit_event('renderChunk', chunk)
```

**Edge Cases and Pitfalls:**
*   **Precision:** Chunk coordinates are parsed from strings. Ensure consistent formatting (e.g., no spaces) to avoid parsing errors [S1, unofficial].
*   **Boundaries:** The `removeDistance` parameter is critical. If set too low, chunks may flicker in and out. If set too high, memory usage increases [S1, unofficial].
*   **Degenerate Input:** If `pending_chunks` is empty, `count` becomes 0, and the loop does not execute. This is handled correctly by the `max(1, ...)` logic only if `asyncChunkGeneration` is true and `count` is not pre-set. If `asyncChunkGeneration` is false, `count` is 0, and the loop is skipped, which is correct [S1, unofficial].

## Subtopics

### Chunk Lifecycle Management
Handles the full lifecycle of a chunk from creation to disposal, including caching and reference counting. Choose this when you need fine-grained control over chunk persistence beyond simple distance-based removal. [Chunk Lifecycle Management](3987-chunk-lifecycle-management.md)

## Operating characteristics
*   **Time Complexity:** Loading is $O(K)$ per frame, where $K$ is the number of chunks processed (typically 10% of pending) [S1, unofficial]. Removal is $O(N)$, where $N$ is the number of loaded chunks, as it iterates through all chunks to check distance [S1, unofficial].
*   **Memory Complexity:** $O(M)$, where $M$ is the number of chunks within `removeDistance` of the player [S1, unofficial].
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Scales well with large worlds as only nearby chunks are loaded. Performance depends on the cost of `generateChunk` and `voxelMesh` [S1, unofficial].
*   **Strengths:** Prevents frame drops by spreading generation load over multiple frames. Reduces memory footprint by unloading distant chunks [S1, unofficial].
*   **Limitations:** Not given in the sources.
*   **When to use:** When rendering large voxel worlds in real-time applications (e.g., games) where memory and frame rate are critical [S1, unofficial].
*   **When not to use:** For small, static worlds where all data can be loaded at once [S1, unofficial].

## Background
This method originates from the `voxel-engine` library by max-mapper, an unofficial implementation of a voxel world engine for JavaScript [S1, unofficial]. It is designed to work with Three.js for rendering and gl-matrix for vector operations [S1, unofficial]. The approach is common in voxel-based games like Minecraft to handle vast, procedurally generated worlds [G].

## Sources
[S1] max-mapper/voxel-engine, https://github.com/max-mapper/voxel-engine, fetched 2026-10-26 (unofficial)
