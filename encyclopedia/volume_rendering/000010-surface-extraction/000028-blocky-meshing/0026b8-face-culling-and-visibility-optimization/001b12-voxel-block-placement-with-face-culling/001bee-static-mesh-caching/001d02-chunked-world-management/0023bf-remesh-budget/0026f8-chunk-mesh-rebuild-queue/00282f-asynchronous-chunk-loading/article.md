# Asynchronous Chunk Loading

Also called: Async Chunk Streaming, Async Voxel Generation, Chunk Task Queue, Multi-threaded Async Chunk Setup

## Overview
Asynchronous Chunk Loading offloads the expensive geometry generation (face culling and buffer creation) of voxel chunks to a background thread pool, allowing the main game or rendering thread to continue processing without frame stutter [S1, unofficial][S2, unofficial]. It sets itself apart from synchronous loading by decoupling the computational cost of mesh creation from the frame loop, using futures and queues to synchronize results with the game thread [S1, unofficial]. This method builds on [Chunk Mesh Rebuild Queue](3911-chunk-mesh-rebuild-queue.md) by providing the underlying mechanism to execute those rebuild tasks in parallel [S1, unofficial].

## How it works
The method manages a queue of pending chunk mesh generation tasks, distributes them across a thread pool, and synchronizes the results back to the main thread for finalization.

### Inputs and Outputs
*   **Inputs**: Player position (to determine load radius), chunk coordinate system, voxel data generation callback (e.g., `PopulateBlock`), and configuration parameters (`ChunkRenderDistance`, `NUMBER_OF_CHUNKS_TO_CREATE_PER_FRAME`).
*   **Outputs**: Completed `MeshData` structures (vertices, indices, normals, UVs) uploaded to the rendering engine's mesh components.

### Data Structures
1.  **`chunkResults` (In-flight Map)**: A hash map keyed by chunk hash (integer) storing pointers to futures (`TFuture<...>*` or equivalent). Used to prevent duplicate generation for the same chunk while a task is in progress [S1, unofficial].
2.  **`m_chunks_to_setup_stack` (Pending Queue)**: A stack or queue of chunk indices waiting to be submitted to the thread pool [S2, unofficial].
3.  **`chunkLoaderQueue` (Result Queue)**: A thread-safe queue storing completed mesh data arrays, consumed by the game thread [S1, unofficial].
4.  **`ChunkMap` (Spatial Hash)**: A map of loaded chunk hashes to active chunk actors/objects [S1, unofficial].

### Numbered Steps
1.  **Detection**: On the game thread, detect player movement or chunk boundary crossing. Calculate the square loading radius based on `ChunkRenderDistance` [S1, unofficial].
2.  **Queueing**: For each chunk coordinate `(i, j)` in the radius:
    *   Check if the chunk hash exists in `ChunkMap` (already loaded) or `chunkResults` (currently generating) [S1, unofficial].
    *   If neither, push the chunk index to `m_chunks_to_setup_stack` [S2, unofficial].
    *   *Pitfall*: Ensure the player's current chunk `(0,0)` relative to player is explicitly handled, as some loop implementations skip it [S1, unofficial].
3.  **Submission**: In a per-frame update function (`create_chunks_from_setup_stack`):
    *   Pop up to `NUMBER_OF_CHUNKS_TO_CREATE_PER_FRAME` indices from the stack [S2, unofficial].
    *   For each index, submit a task to the thread pool using an asynchronous execution primitive (e.g., `Async` with `TFuture` return type) [S1, unofficial].
    *   The task executes `GetMeshDataForChunk(i, j, PopulateBlock)` [S1, unofficial].
    *   Store the returned future pointer in `chunkResults` keyed by the chunk hash [S1, unofficial].
4.  **Processing**: On the game thread, dequeue results from `chunkLoaderQueue` one per frame [S1, unofficial].
5.  **Finalization**: Upon receiving a future result:
    *   Call the chunk creation function (e.g., `AChunk::CreateChunk()`) [S1, unofficial].
    *   Upload the mesh data (vertices, indices, normals, UVs) to the rendering component (e.g., `UProceduralMeshComponent`) [S1, unofficial].
    *   Add the chunk to `ChunkMap` [S1, unofficial].
    *   Remove the entry from `chunkResults` [S1, unofficial].

### Formulas
*   **Chunk Hash**: $H = \text{GetHashFromChunkPosition}(i, j)$ [S1, unofficial].
*   **Loading Radius**: Chunks are processed where $|i - \text{PlayerChunkX}| \le \text{ChunkRenderDistance}$ and $|j - \text{PlayerChunkY}| \le \text{ChunkRenderDistance}$ [S1, unofficial].

### Parameters
*   `ChunkRenderDistance`: Integer radius for the square loading area [S1, unofficial].
*   `NUMBER_OF_CHUNKS_TO_CREATE_PER_FRAME`: Maximum number of chunks to start generating per frame [S2, unofficial].
*   `PopulateBlock`: Callback function signature for procedural voxel data generation [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Global state
chunk_results = {}  # Map<int32, Future*>
chunk_map = {}      # Map<int32, Chunk*>
setup_stack = []    # List<int32>
result_queue = Queue()

def on_player_move(new_chunk_x, new_chunk_y, render_distance, populate_block):
    """Called when player moves to a new chunk."""
    # 1. Iterate square radius
    for i in range(new_chunk_x - render_distance, new_chunk_x + render_distance + 1):
        for j in range(new_chunk_y - render_distance, new_chunk_y + render_distance + 1):
            # 2. Skip player's current chunk if not handled elsewhere (Pitfall)
            if i == new_chunk_x and j == new_chunk_y:
                continue
            
            chunk_hash = get_hash(i, j)
            
            # 3. Check if already loaded or in-flight
            if chunk_hash in chunk_map:
                continue
            if chunk_hash in chunk_results:
                continue
                
            # 4. Add to setup stack
            setup_stack.append(chunk_hash)

def process_setup_stack(max_chunks_per_frame):
    """Called every frame on game thread."""
    count = 0
    while setup_stack and count < max_chunks_per_frame:
        chunk_hash = setup_stack.pop()
        
        # Submit to thread pool
        future = thread_pool.submit_async(get_mesh_data_for_chunk, chunk_hash, populate_block)
        
        # Store in-flight reference
        chunk_results[chunk_hash] = future

def process_results():
    """Called every frame on game thread."""
    while not result_queue.is_empty():
        chunk_data = result_queue.dequeue()
        chunk_hash = chunk_data.hash
        create_chunk(chunk_hash, chunk_data)
        
        # Cleanup in-flight map
        if chunk_hash in chunk_results:
            del chunk_results[chunk_hash]

def get_mesh_data_for_chunk(chunk_hash, populate_block):
    """Executed on worker thread."""
    # Decode hash to coordinates (implementation specific)
    i, j = decode_hash(chunk_hash)
    
    # Generate voxel data
    voxels = populate_block(i, j)
    
    # Generate mesh (face culling, etc.)
    mesh_data = generate_mesh(voxels)
    
    return mesh_data

def create_chunk(chunk_hash, mesh_data):
    """Executed on game thread."""
    # Upload mesh to GPU/Engine
    actor = spawn_chunk_actor(chunk_hash)
    actor.upload_mesh(mesh_data)
    chunk_map[chunk_hash] = actor
```

### Edge Cases and Pitfalls
*   **Duplicate Generation**: Without checking `chunkResults`, concurrent player movement or rapid chunk boundary crossings can trigger multiple generation tasks for the same chunk [S1, unofficial].
*   **Current Chunk Skipping**: Loop implementations that iterate relative coordinates may skip `(0,0)` if not explicitly guarded, leaving the player's current chunk unqueued [S1, unofficial].
*   **Thread Safety**: The `chunkResults` map and `setup_stack` must be protected if accessed from multiple threads, though typically the game thread manages the queue and worker threads only write to the result queue [S1, unofficial][S2, unofficial].

## Subtopics

### [Dependency Queue Gating](11597-dependency-queue-gating.md)
Dependency Queue Gating delays the mesh generation of a voxel chunk until all its neighboring chunks are loaded, ensuring that complete visibility data is available before culling inner faces. This method sets itself apart from reactive remeshing by proactively managing load order to minimize redundant mesh calculations, trading increased pre-loading checks for reduced re-meshing overhead. It builds on [Asynchronous Chunk Loading](4216-asynchronous-chunk-loading.md) by adding a prerequisite check to the task queue.

### Chunk Upload Blocking
Chunk Upload Blocking delays GPU uploads for nearby changing chunks to prevent visual voids caused by asynchronous mesh generation [S3, unofficial]. It iterates over chunks, marking those within a distance of 512 units from the camera as "near" [S3, unofficial]. If a near chunk is dirty or changing, it blocks upload (`block_upload = 1`) and queues the update (max 16 uploads) [S3, unofficial]. The block is forced to release after 0.1 seconds or if the unblock time is exceeded, ensuring progress even if the chunk remains dirty [S3, unofficial].

### Concurrent Chunk Mesh Queue Processing
Concurrent Chunk Mesh Queue Processing uses a thread-safe concurrent hash map to track chunks and a lock-free queue to process incoming mesh data, allowing asynchronous mesh generation while rendering occurs on a single thread [S4, unofficial]. It processes a deletion queue by popping indices, finding them in the map, and deallocating buffers [S4, unofficial]. A key pitfall is that rendering must occur on a single thread because OpenGL context and state changes are not thread-safe across multiple draw calls [S4, unofficial].

### Asynchronous Chunk Generation Scheduling
Asynchronous Chunk Generation Scheduling processes a queue of pending chunk positions, generating voxel data and optionally rendering them, with support for async batching [S5, unofficial]. It uses a boolean flag `asyncChunkGeneration` to control processing volume: if false, all pending chunks are processed synchronously; if true, only 10% of the queue is processed per tick (clamped between 1 and total length) [S5, unofficial]. Chunk positions are stored as strings `'x|y|z'` and parsed using `split('|')` and bitwise OR `|0` to ensure integer indices [S5, unofficial]. If the count is 0, no chunks are processed, preventing UI freezes [S5, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(N)$ for generating $N$ chunks, but distributed across multiple frames and threads, reducing per-frame latency from $O(N)$ to $O(1)$ amortized [S1, unofficial][S2, unofficial].
*   **Memory Complexity**: $O(K)$ where $K$ is the number of in-flight chunks, due to storing futures and intermediate mesh data in `chunkResults` and queues [S1, unofficial].
*   **Measured Performance**: Prevents frame stutter by capping the number of chunks generated per frame (`NUMBER_OF_CHUNKS_TO_CREATE_PER_FRAME`) [S2, unofficial].
*   **Strengths**: Eliminates hitching during chunk loading; allows main thread to remain responsive for input and logic [S1, unofficial].
*   **Limitations**: Introduces latency between chunk request and availability; requires careful synchronization to avoid race conditions [S1, unofficial].
*   **When to Use**: Large voxel worlds with dynamic generation or editing where frame rate stability is critical [S1, unofficial][S2, unofficial].
*   **When Not to Use**: Small, static worlds where synchronous loading is negligible; or when strict real-time consistency of neighbor culling is required without gating [S1, unofficial].

## Background
Asynchronous Chunk Loading originated in modern game engines (e.g., UE5) and voxel engine implementations to address the performance bottlenecks of procedural geometry generation [S1, unofficial][S2, unofficial]. It evolved from simple synchronous loading schemes as voxel worlds grew in size and complexity, requiring offloading of face culling and buffer creation to background threads [S1, unofficial][S2, unofficial]. It is widely used in voxel-based games, terrain editors, and procedural generation tools [S1, unofficial][S2, unofficial].

## Sources
[S1] Manolo94/UE5MinecraftClone, https://github.com/Manolo94/UE5MinecraftClone, fetched 2026-10-26 (unofficial)
[S2] rtarun9/voxel-engine, https://github.com/rtarun9/voxel-engine, fetched 2026-10-27 (unofficial)
[S3] kosshi-net/voxplat, https://github.com/kosshi-net/voxplat, fetched 2026-10-27 (unofficial)
[S4] EmaMaker/voxel-engine, https://github.com/EmaMaker/voxel-engine, fetched 2026-10-26 (unofficial)
[S5] voxel/voxel-engine-stackgl, https://github.com/voxel/voxel-engine-stackgl, fetched 2026-10-27 (unofficial)
