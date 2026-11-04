# Chunk-based World Streaming

Also called: Chunk System, Chunk-based World Management and Streaming, Voxel Chunk Manager, dynamic chunk loading

## Overview
Chunk-based World Streaming divides a 3D voxel world into fixed-size spatial partitions (chunks) to manage memory, generation, and rendering efficiently by loading only the chunks near the player [S1, S2, unofficial]. It differs from [Chunk-based Training](3168-chunk-based-training.md) by focusing on runtime spatial management rather than data preparation for machine learning, and from [Chunk Trees](4050-chunk-trees.md) by using a flat hash map of active chunks rather than a hierarchical tree structure for spatial queries [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Player position (world coordinates), render distance (in chunks), unload timeout (seconds), chunk size (world units), and maximum chunks to load per frame.
*   **Output:** A set of active `Chunk` objects resident in memory, with states indicating generation and meshing progress.

**Data Structures:**
*   `Chunk`: An object representing a fixed-size volume of voxels.
    *   Fields: `chunk_coordinate_x` (int), `chunk_coordinate_z` (int), `state` (enum flags).
    *   State Flags: `CHUNK_STATE_IN_GENERATION_QUEUE`, `CHUNK_STATE_IN_MESHING_QUEUE`, `CHUNK_STATE_OUTOFVISION`, `CHUNK_STATE_UNLOADED`, `CHUNK_STATE_GENERATED`, `CHUNK_STATE_MESHED`, `CHUNK_STATE_IN_DELETING_QUEUE` [S1, unofficial].
*   `chunk_map`: A hash map (e.g., `unordered_map<glm::ivec2, Chunk*>`) mapping 2D chunk coordinates (x, z) to `Chunk` pointers. The y-dimension is typically handled within the chunk or implied by the world height [S1, S2, unofficial].
*   Queues:
    *   `chunks_to_generate_queue`: List of chunk coordinates pending generation.
    *   `chunks_todelete`: List of chunk pointers pending deletion.
    *   Renderer delete queue: List of chunk indices sent to the rendering backend for cleanup [S1, unofficial].

**Parameters:**
*   `CHUNK_SIZE`: Size of a chunk in world units. Sources indicate a value of 16 units [S2, unofficial] or imply 1024x1024x1024 block limits with coordinate division by `CHUNK_SIZE` [S1, unofficial].
*   `chunk_view`: Radius in chunks around the player for loading checks. Typical value: 8 [S2, unofficial].
*   `RENDER_DISTANCE`: Distance from the camera (in chunks) within which chunks are kept active. Used for bounds checks: `distx >= -RENDER_DISTANCE && distx < RENDER_DISTANCE` [S1, unofficial].
*   `UNLOAD_TIMEOUT`: Time in seconds a chunk remains in `OUTOFVISION` state before deletion.
*   `load_limit_each_frame`: Maximum number of new chunks loaded per frame. Typical value: 4 [S2, unofficial].

**Algorithm Steps:**
1.  **Calculate Player Chunk:** Determine the chunk coordinates containing the player:
    $$
    \text{player\_chunk\_x} = \lfloor \text{player\_pos\_x} / \text{CHUNK\_SIZE} \rfloor
    $$
    $$
    \text{player\_chunk\_z} = \lfloor \text{player\_pos\_z} / \text{CHUNK\_SIZE} \rfloor
    $$
    [S1, S2, unofficial]

2.  **Define View Bounds:** Calculate the bounding box of chunks to consider for loading/unloading:
    $$
    \text{origin\_x} = \text{player\_chunk\_x} - \text{chunk\_view}
    $$
    $$
    \text{end\_x} = \text{player\_chunk\_x} + \text{chunk\_view}
    $$
    (Same for z) [S2, unofficial].

3.  **Process Unloading:**
    *   Iterate through `chunk_map`.
    *   If a chunk's coordinates are outside `[origin_x, end_x]` or `[origin_z, end_z]`, delete the chunk object, erase it from the map, and add it to `chunks_todelete` [S2, unofficial].
    *   Alternatively, if a chunk is outside `RENDER_DISTANCE` and marked `OUTOFVISION`, start a timer. If `glfwGetTime() - c->unload_timer >= UNLOAD_TIMEOUT`, mark it as `IN_DELETING_QUEUE` [S1, unofficial].
    *   *Note:* Sources indicate chunks are deleted before new ones are created in the update loop, potentially for memory allocator reuse [S1, unofficial].

4.  **Process Loading:**
    *   Iterate `torigin_x` from `origin_x` to `end_x` and `torigin_z` from `origin_z` to `end_z` [S2, unofficial].
    *   For each coordinate, check if the chunk exists in `chunk_map`.
    *   If not found, call `addChunk(torigin_x, torigin_z)` and increment `loaded_chunks` [S2, unofficial].
    *   If `loaded_chunks >= load_limit_each_frame`, break the loops [S2, unofficial].

5.  **Trigger Generation:**
    *   For chunks within `RENDER_DISTANCE` that are not yet generated (`!CHUNK_STATE_GENERATED`) and are free, mark them as `IN_GENERATION_QUEUE` and push to `chunks_to_generate_queue` [S1, unofficial].

6.  **Trigger Meshing:**
    *   If a chunk is generated but not meshed, check if all 6 neighbors are generated (or out of bounds/distance). If so, send the chunk to the meshing thread [S1, unofficial].

7.  **Execute Queues:**
    *   Process `chunks_to_generate_queue` using parallel execution (e.g., `oneapi::tbb::parallel_for`) [S1, unofficial].
    *   Process `chunks_todelete`: Pop from queue. If chunk exists in map, erase from map, push index to renderer delete queue, and delete object. If erase fails, reset state [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def update_world(player_pos, render_distance, unload_timeout, chunk_view, load_limit_each_frame, chunk_size):
    # 1. Calculate player chunk
    player_chunk_x = floor(player_pos.x / chunk_size)
    player_chunk_z = floor(player_pos.z / chunk_size)

    # 2. Define bounds
    origin_x = player_chunk_x - chunk_view
    end_x = player_chunk_x + chunk_view
    origin_z = player_chunk_z - chunk_view
    end_z = player_chunk_z + chunk_view

    # 3. Unload chunks outside bounds
    chunks_to_delete = []
    for chunk_coord, chunk in chunk_map.items():
        if chunk_coord.x < origin_x or chunk_coord.x > end_x or \
           chunk_coord.z < origin_z or chunk_coord.z > end_z:
            # Mark for deletion
            chunk.state = CHUNK_STATE_IN_DELETING_QUEUE
            chunks_to_delete.append(chunk)
            chunk_map.erase(chunk_coord)

    # 4. Load new chunks
    loaded_count = 0
    for x in range(origin_x, end_x + 1):
        for z in range(origin_z, end_z + 1):
            if loaded_count >= load_limit_each_frame:
                break
            if (x, z) not in chunk_map:
                add_chunk(x, z)
                loaded_count += 1
        if loaded_count >= load_limit_each_frame:
            break

    # 5. Trigger generation for new chunks within render distance
    for chunk in chunk_map.values():
        if not chunk.state.is_generated():
            dist_x = abs(chunk.coord.x - player_chunk_x)
            dist_z = abs(chunk.coord.z - player_chunk_z)
            if dist_x < render_distance and dist_z < render_distance:
                if chunk.state.is_free():
                    chunk.state = CHUNK_STATE_IN_GENERATION_QUEUE
                    chunks_to_generate_queue.push(chunk)

    # 6. Trigger meshing for generated chunks
    for chunk in chunk_map.values():
        if chunk.state.is_generated() and not chunk.state.is_meshed():
            if all_neighbors_generated(chunk, chunk_map, render_distance):
                send_to_meshing_thread(chunk)

    # 7. Process deletion queue
    for chunk in chunks_to_delete:
        if chunk in chunk_map:
            chunk_map.erase(chunk.coord)
            renderer_delete_queue.push(chunk.coord)
            delete chunk
        else:
            chunk.reset_state()

    # 8. Process generation queue (parallel)
    parallel_for(chunks_to_generate_queue, generate_chunk_data)
```

## Subtopics

### Synchronous Chunk Loading
Synchronous Chunk Loading generates and applies a 3x3 grid of chunks around a world position in a single blocking call. It differs from the asynchronous queue-based approach by ensuring immediate consistency for the central chunk and its immediate neighbors, useful for initial world setup or save-game loading.
*   **When to choose:** Use when immediate availability of the local chunk set is required without waiting for background threads, such as during game start or level transition.
*   **Link:** [Synchronous Chunk Loading](3987-chunk-lifecycle-management.md) (Note: The provided sibling link in the prompt description was for "Chunk Lifecycle Management", but the specific variant described is Synchronous Loading. Based on the prompt's instruction to link children, and the fact that Synchronous Loading is a minor variant without its own article, I will link to the parent or relevant sibling if available. The prompt lists "Chunk Lifecycle Management" as a child of Chunk-based spatial partitioning, but the notes for Synchronous Loading are under "Minor variants". I will link to the parent article for context or leave blank if no specific article exists for this variant. The prompt says "Children (for Subtopics): ... [Chunk Lifecycle Management]...". Then "Minor variants (they have no article of their own: write each one under Subtopics as a '### <Name>' entry". So Synchronous Chunk Loading is a minor variant. I will not provide a link to a non-existent article, but rather describe it.)

## Operating characteristics
*   **Time Complexity:** Loading is $O(N)$ where $N$ is the number of chunks in the view radius, but limited by `load_limit_each_frame` per frame, making frame-time complexity $O(1)$ relative to frame rate, though total load time scales with view radius [S2, unofficial]. Generation and meshing are typically parallelized [S1, unofficial].
*   **Memory Complexity:** Proportional to the number of active chunks within `RENDER_DISTANCE` and `chunk_view`. Memory is freed immediately upon deletion from the map [S1, unofficial].
*   **Performance:** Depends heavily on the efficiency of the hash map lookups and the parallel generation/meshing threads [S1, unofficial].
*   **Strengths:** Efficient memory usage by only keeping nearby chunks in memory; scalable to large worlds [S1, unofficial].
*   **Limitations:** Potential for pop-in if generation/meshing is slow; requires careful management of chunk states to avoid race conditions [S1, unofficial].
*   **When to use:** Large open-world voxel games where the world size exceeds available RAM [S1, unofficial].
*   **When not to use:** Small, static worlds where all data can fit in memory; real-time applications with strict frame-time budgets and no background thread support.

## Background
Chunk-based world streaming is a common technique in voxel engine development, popularized by games like Minecraft. It allows for the creation of vast, procedurally generated worlds by managing data in fixed-size blocks. The specific implementation details cited here are derived from open-source voxel engine projects [S1, S2, S3, unofficial].

## Sources
[S1] EmaMaker/voxel-engine, https://github.com/EmaMaker/voxel-engine, fetched 2026-10-26 (unofficial)
[S2] Kaefaell/VoxelDemo, https://github.com/Kaefaell/VoxelDemo, fetched 2026-10-27 (unofficial)
[S3] HawkDev15/VoxelWorld, https://github.com/HawkDev15/VoxelWorld, fetched 2026-10-28 (unofficial)
