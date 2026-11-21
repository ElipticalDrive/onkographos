# Chunk-Based World Management & Asynchronous Loading

Also called: Async Chunk Streaming, Dynamic Chunk Loading, Chunk Culling and Unloading, Chunk Management (Distance-Based Loading/Unloading)

## Overview
Chunk-Based World Management & Asynchronous Loading is a system for managing volumetric or mesh-based voxel worlds by dividing them into fixed-size chunks and dynamically loading or unloading them based on camera proximity [S1, S3, S4, unofficial]. It solves memory and rendering load issues by removing distant chunks and requesting missing ones [S1, S2, unofficial]. It sets itself apart from [Synchronous Chunk Loading](3904-synchronous-chunk-loading.md) by throttling generation to prevent frame spikes, and from [Chunk Lifecycle Management](3987-chunk-lifecycle-management.md) by focusing on the spatial culling and async I/O/generation pipeline rather than just object lifetime [S1, S4, unofficial].

## How it works
**Inputs:** Camera/player position, render distance parameters, chunk size, asynchronous generation flag.
**Outputs:** Updated set of active chunks in the scene, with meshes rendered and distant meshes disposed.

**Data Structures:**
*   `chunks`: Map or vector storing chunk data keyed by position (string `'x|y|z'` or `glm::ivec3`) [S2, S3, unofficial].
*   `pendingChunks`: List of chunk positions awaiting generation [S1, unofficial].
*   `activeChunkKeys_`: List of chunk keys currently within render distance [S3, unofficial].
*   `meshes`: Map storing GPU resources (VAOs, geometries) associated with chunk keys [S2, unofficial].

**Steps:**

1.  **Calculate Proximity:**
    *   For flat worlds: Compute camera chunk coordinates using integer division:
        $$x_{pos} = \lfloor \text{CameraPos}_x / \text{CHUNK\_SIZE} \rfloor$$
        $$y_{pos} = \lfloor \text{CameraPos}_z / \text{CHUNK\_SIZE} \rfloor$$ [S4, unofficial]
    *   For spherical worlds: Calculate distance to planet center:
        $$d_{center} = \| \text{CamPos} - \text{PlanetCenter} \|$$
        $$d_{surface} = d_{center} - \text{Radius}$$ [S3, unofficial]
        Calculate relative camera chunk key:
        $$\text{CamChunkKey} = \lfloor (\text{CamPos} - \text{PlanetCenter}) / \text{ChunkSize} \rfloor$$ [S3, unofficial]

2.  **Identify Active Chunks:**
    *   Query nearby chunks within `removeDistance` (or `chunkRenderDistance`) from the camera [S1, S2, S4, unofficial].
    *   For spherical worlds, iterate a radius around the camera chunk key. Skip chunks if their distance from the camera exceeds the render limit [S3, unofficial].
    *   For spherical worlds, ensure chunks intersect the planet sphere by checking if the chunk center distance is within $\text{Radius} + \text{ChunkSize} \times \sqrt{3}$ [S3, unofficial].

3.  **Unload Distant Chunks:**
    *   Iterate over all currently loaded chunks.
    *   If a chunk is not in the active set:
        *   Remove from `pendingChunks` if present [S2, unofficial].
        *   Dispose GPU resources (e.g., delete VAOs, geometries) [S1, S2, unofficial].
        *   Delete chunk data and mesh references from storage [S2, unofficial].
        *   Emit `removeChunk` event [S1, S2, unofficial].

4.  **Load Missing Chunks (Asynchronous):**
    *   Identify chunks in the active set that are not yet loaded [S2, S4, unofficial].
    *   If `asyncChunkGeneration` is false, load all pending chunks immediately [S1, unofficial].
    *   If `asyncChunkGeneration` is true, throttle the load rate:
        *   Calculate count: $\max(1, \min(\text{pendingChunks.length} \times 0.1, \text{pendingChunks.length}))$ [S1, unofficial].
        *   Or limit to `maxNewPerFrame` (e.g., 16) for spherical worlds [S3, unofficial].
        *   Or load only one chunk per frame if a `chunkAdded` flag is set [S4, unofficial].
    *   For each chunk to load:
        *   Add to `pendingChunks` if not already present [S1, unofficial].
        *   Trigger asynchronous generation task with seed, position, and radius [S3, unofficial].
        *   If meshing is required, add async mesh building task [S3, unofficial].

5.  **Render Active Chunks:**
    *   For each key in `activeChunkKeys_`:
        *   If chunk data is ready and mesh exists:
            *   Initialize OpenGL context if needed [S3, unofficial].
            *   Render if distance $\le \text{ChunkSize} \times (\text{ChunkRenderDistance} + 1)$ [S3, unofficial].

**Parameters:**
*   `CHUNK_SIZE`: Spatial dimensions of a chunk in world units [S4, unofficial].
*   `CHUNK_RENDER_RADIUS`: Maximum distance in chunks from camera to load/unload [S4, unofficial].
*   `removeDistance`: Maximum distance from player to keep chunks loaded [S1, S2, unofficial].
*   `asyncChunkGeneration`: Boolean flag for throttled loading [S1, unofficial].
*   `CHUNK_LOAD_RATE`: Minimum time interval (seconds) between chunk loading operations [S4, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def manage_chunks(camera_pos, chunk_size, render_radius, remove_distance, async_gen):
    # 1. Identify active chunks
    # Flat world example
    cam_chunk_x = floor(camera_pos.x / chunk_size)
    cam_chunk_y = floor(camera_pos.z / chunk_size)
    
    active_keys = set()
    for dx in range(-render_radius, render_radius + 1):
        for dy in range(-render_radius, render_radius + 1):
            chunk_key = (cam_chunk_x + dx, cam_chunk_y + dy)
            # Check distance logic specific to world type
            if is_within_render_distance(chunk_key, camera_pos, chunk_size, render_radius):
                active_keys.add(chunk_key)

    # 2. Unload distant chunks
    loaded_keys = set(chunks.keys())
    for key in loaded_keys:
        if key not in active_keys:
            dispose_mesh(key)
            del chunks[key]
            del meshes[key]
            if key in pending_chunks:
                pending_chunks.remove(key)
            emit_event('removeChunk', key)

    # 3. Load missing chunks
    missing_keys = active_keys - loaded_keys
    pending_chunks.update(missing_keys)
    
    # Throttle loading
    if not async_gen:
        load_count = len(pending_chunks)
    else:
        # S1: 10% of pending, min 1
        load_count = max(1, min(len(pending_chunks) * 0.1, len(pending_chunks)))
        # S4: Or limit to 1 per frame
        # load_count = 1 if not chunk_added_flag else 0
    
    for i in range(load_count):
        if not pending_chunks:
            break
        key = pending_chunks.pop(0) # Or iterate and splice
        chunk_data = generate_chunk_async(key)
        if chunk_data:
            chunks[key] = chunk_data
            meshes[key] = create_mesh(chunk_data)
            emit_event('renderChunk', key)
```

## Subtopics

### Chunked Async Reading
Reads data in chunks to fill a buffer asynchronously. Reserves 8 MB in buffer, loops reading into buffer until capacity is reached or EOF, yielding after each read [S5, unofficial].

## Operating characteristics
*   **Time Complexity:** Loading is throttled to $O(1)$ or $O(k)$ per frame where $k$ is small (e.g., 1-16 chunks), preventing frame spikes [S1, S3, S4, unofficial]. Unloading is $O(N)$ where $N$ is the number of loaded chunks, but typically $N$ is small relative to the total world.
*   **Memory Complexity:** Memory usage is proportional to the render volume ($\text{RenderRadius}^2 \times \text{ChunkSize}^3$) [S2, S4, unofficial].
*   **Performance:** Prevents stuttering during camera movement or large jumps by spreading generation cost over multiple frames [S4, unofficial].
*   **Limitations:** Requires careful tuning of `removeDistance` and `renderRadius` to balance memory and latency. Spherical worlds add complexity with distance checks against planet radius [S3, unofficial].
*   **When to use:** Large open worlds where the player moves continuously. Not suitable for small, static scenes where synchronous loading is simpler.

## Background
Originating from voxel game engines like `voxel-engine` [S1, S2, unofficial] and extended to spherical planet rendering in projects like `AzureVoxel` [S3, unofficial] and `VoxelEngine` [S4, unofficial], this method addresses the scalability limits of in-memory voxel storage. It evolved from simple distance-based culling to include asynchronous generation and meshing to maintain frame rates [S1, S3, unofficial].

## Sources
[S1] max-mapper/voxel-engine, https://github.com/max-mapper/voxel-engine, fetched 2026-10-26 (unofficial)
[S2] voxel/voxel-engine-stackgl, https://github.com/voxel/voxel-engine-stackgl, fetched 2026-10-27 (unofficial)
[S3] AlexandruDodita/AzureVoxel, https://github.com/AlexandruDodita/AzureVoxel, fetched 2026-10-28 (unofficial)
[S4] GitHub Repo for VoxelEngine, https://github.com/TheSlabby/VoxelEngine, fetched 2026-10-29 (unofficial)
[S5] Brush – A new compatible Gaussian splatting engine, https://github.com/ArthurBrussee/brush, fetched 2026-10-26 (unofficial)
