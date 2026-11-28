# Asynchronous Chunk Loading

Also called: Async Chunk Streaming, Async Voxel Generation, Chunk Task Queue, Multi-threaded Async Chunk Setup

## Overview
Asynchronous Chunk Loading offloads the computationally expensive geometry generation (face culling and buffer creation) of voxel chunks to a background thread pool, allowing the main thread to continue processing gameplay and rendering while chunks are prepared [S2, unofficial]. It solves the problem of frame stutter caused by synchronous mesh generation by decoupling heavy computation from the render loop [S1, unofficial]. It differs from its parent [Chunk Mesh Rebuild Queue](3911-chunk-mesh-rebuild-queue.md), which manages the batching and scheduling of rebuilds, by specifically handling the execution of these tasks on a separate thread pool and synchronizing results via futures and queues [S1, unofficial]. It differs from siblings [Multithreaded Job Queue (Generation & Remeshing)](6848-multithreaded-job-queue-generation-remeshing.md) by focusing on the streaming aspect of chunk setup and the synchronization of generated mesh data with the game thread [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Player position (to determine chunk coordinates), a procedural voxel data generation callback (e.g., `PopulateBlock`), and configuration parameters (`ChunkRenderDistance`, `NUMBER_OF_CHUNKS_TO_CREATE_PER_FRAME`).
*   **Output:** A queue of completed `MeshData` structures ready for upload to the GPU, and the finalization of chunk actors/components in the game world.

**Data Structures:**
1.  **`chunkResults` (In-flight Map):** A hash map keyed by chunk hash (integer) storing pointers to asynchronous future objects (`TFuture<TArray<MeshData*>>*` or equivalent). This prevents duplicate generation requests for the same chunk while it is being processed [S1, unofficial].
2.  **`m_chunks_to_setup_stack` (Pending Queue):** A stack or queue of chunk indices waiting to be submitted to the thread pool [S2, unofficial].
3.  **`chunkLoaderQueue` (Result Queue):** A thread-safe queue (`TQueue` or similar) storing completed `MeshData` arrays from worker threads, consumed by the main thread [S1, unofficial].
4.  **`ChunkMap` (Spatial Hash):** A map from chunk coordinates to active chunk objects, used to check if a chunk is already loaded or being generated [S1, unofficial].

**Steps:**
1.  **Detection:** On the main thread (e.g., during player movement tick), detect when the player enters a new chunk or is within the render distance of unloaded chunks [S1, unofficial].
2.  **Queueing:** For each chunk coordinate `(i, j)` within the render distance:
    *   Calculate the chunk hash.
    *   Check if the chunk exists in `ChunkMap` (already loaded) or `chunkResults` (currently being generated).
    *   If neither, add the chunk index to `m_chunks_to_setup_stack` [S2, unofficial].
3.  **Submission:** In a dedicated setup function (e.g., `create_chunks_from_setup_stack`):
    *   Pop up to `NUMBER_OF_CHUNKS_TO_CREATE_PER_FRAME` indices from the stack [S2, unofficial].
    *   For each index, submit a task to the thread pool. The task signature typically involves calling a function like `GetMeshDataForChunk(i, j, PopulateBlock)` [S1, unofficial].
    *   Store the returned future pointer in `chunkResults` keyed by the chunk hash [S1, unofficial].
4.  **Processing:** On the main thread, dequeue results from `chunkLoaderQueue` one per frame [S1, unofficial].
5.  **Finalization:** For each dequeued result:
    *   Call the chunk creation function (e.g., `AChunk::CreateChunk`) to upload the mesh data to the GPU and instantiate the chunk actor [S1, unofficial].
    *   Remove the entry from `chunkResults` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Data Structures
chunk_results: Map<int32, Future<Array<MeshData*>>>  # Tracks in-flight generation
chunk_loader_queue: Queue<Array<MeshData*>>          # Stores completed mesh data
chunks_to_setup_stack: Stack<int32>                  # Chunks waiting to be submitted
chunk_map: Map<int32, Chunk>                         # Loaded chunks

# Parameters
CHUNK_RENDER_DISTANCE: int32
NUMBER_OF_CHUNKS_TO_CREATE_PER_FRAME: int32

# Callback
PopulateBlock: Function(int32 i, int32 j, int32 k) -> BlockType

def on_player_moved(new_chunk_x, new_chunk_y):
    """Called when player moves to a new chunk."""
    # 1. Identify chunks to load
    for i from (new_chunk_x - CHUNK_RENDER_DISTANCE) to (new_chunk_x + CHUNK_RENDER_DISTANCE):
        for j from (new_chunk_y - CHUNK_RENDER_DISTANCE) to (new_chunk_y + CHUNK_RENDER_DISTANCE):
            # Skip player's current chunk if handled separately or ensure it is queued
            if i == new_chunk_x and j == new_chunk_y:
                continue # Pitfall: Ensure current chunk is handled elsewhere if needed
            
            chunk_hash = get_hash(i, j)
            
            # 2. Check if already loaded or in-flight
            if chunk_hash in chunk_map:
                continue
            if chunk_hash in chunk_results:
                continue
                
            # 3. Add to setup stack
            chunks_to_setup_stack.push(chunk_hash)

def create_chunks_from_setup_stack():
    """Called every frame to submit tasks to the thread pool."""
    count = 0
    while not chunks_to_setup_stack.empty() and count < NUMBER_OF_CHUNKS_TO_CREATE_PER_FRAME:
        chunk_hash = chunks_to_setup_stack.pop()
        
        # 4. Submit to thread pool
        # The task returns a Future containing the mesh data
        future = async_execute_thread_pool(get_mesh_data_for_chunk, chunk_hash, PopulateBlock)
        
        # 5. Store in-flight request
        chunk_results[chunk_hash] = future

def process_chunk_results():
    """Called every frame on the main thread to finalize loaded chunks."""
    while not chunk_loader_queue.empty():
        mesh_data_array = chunk_loader_queue.dequeue()
        
        # 6. Finalize chunk
        # Extract chunk coordinates from mesh_data_array or associated metadata
        # upload_mesh_to_gpu(mesh_data_array)
        # instantiate_chunk_actor(mesh_data_array)
        
        # Cleanup in-flight tracking
        # Note: In a real implementation, the future callback or a separate step 
        # removes the entry from chunk_results. Here we assume the queueing 
        # mechanism handles the removal or it is done in the callback.
        pass

def get_mesh_data_for_chunk(chunk_hash, populate_block_func):
    """Worker thread function."""
    # Decode hash to coordinates (i, j)
    i, j = decode_hash(chunk_hash)
    
    # Generate voxel data and mesh
    # This involves face culling and buffer creation
    mesh_data = generate_chunk_mesh(i, j, populate_block_func)
    
    return mesh_data

def chunk_callback(future_result, chunk_hash):
    """Callback invoked when a future completes."""
    mesh_data = future_result.get()
    chunk_loader_queue.enqueue(mesh_data)
    chunk_results.remove(chunk_hash)
```

**Parameters:**
*   `ChunkRenderDistance`: Integer radius defining the square area of chunks to consider for loading [S1, unofficial].
*   `NUMBER_OF_CHUNKS_TO_CREATE_PER_FRAME`: Maximum number of chunks to submit for generation per frame, throttling the load [S2, unofficial].
*   `PopulateBlock`: A callback function signature used to generate procedural voxel data for each chunk [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Current Chunk Skipping:** A common loop implementation iterates `i` and `j` relative to the player but may skip the `(0,0)` offset (player's current chunk) due to conditional checks like `if (i != 0 || j != 0)`, leaving the player's chunk unqueued if not handled separately [S1, unofficial].
*   **Duplicate Generation:** Without tracking in-flight requests in `chunkResults`, rapid player movement or camera rotation can trigger multiple generation tasks for the same chunk, wasting CPU cycles [S1, unofficial].
*   **Thread Safety:** The `chunkLoaderQueue` and `chunk_results` map must be thread-safe or accessed only from their respective threads (worker for submission, main for consumption) [S1, unofficial].

## Subtopics

### [Asynchronous Chunk Generation Scheduling](6853-asynchronous-chunk-generation-scheduling.md)
Asynchronous Chunk Generation Scheduling controls the rate at which pending voxel chunks are processed from a queue to prevent frame-rate drops during world generation. It solves the problem of synchronous mesh generation blocking the main thread by limiting the number of chunks generated per frame. It differs from its parent [Asynchronous Chunk Loading](4216-asynchronous-chunk-loading.md), which offloads work to a thread pool, by performing the generation logic directly on the calling thread (typically the main thread) but throttling the iteration count. It differs from siblings [Dependency Queue Gating](11597-dependency-queue-gating.md) and [Web Worker Pool for Meshing](14649-web-worker-pool-for-meshing.md) by using a simple percentage-based cap rather than dependency resolution or separate worker threads.

### [Dependency Queue Gating](11597-dependency-queue-gating.md)
Dependency Queue Gating delays mesh generation for a voxel chunk until all neighboring chunks are loaded, ensuring complete visibility data is available before culling inner faces. This method proactively manages chunk load order to minimize redundant mesh calculations, trading increased pre-loading checks for reduced re-meshing overhead.

### [Web Worker Pool for Meshing](14649-web-worker-pool-for-meshing.md)
This method offloads the computationally expensive task of chunk mesh generation (specifically greedy meshing) from the main thread to a pool of Web Workers, ensuring the main thread remains responsive for gameplay and rendering. It distinguishes itself by combining a background worker pool for bulk remeshing with a main-thread mesher for immediate edits, sharing a single WebAssembly voxel core to avoid memory duplication.

### [Threaded Chunk Generation](19413-threaded-chunk-generation.md)
Threaded Chunk Generation is a technique that offloads the generation of chunk data (density fields, material IDs, and mesh arrays) to worker threads, ensuring that runtime generation remains smooth while chunks stream around the player or camera. It differs from single-threaded generation by decoupling the heavy computation of mesh creation from the main render loop, and from pure asynchronous loading by focusing specifically on the computational cost of meshing rather than just I/O.

### Chunk Upload Blocking
Chunk Upload Blocking delays GPU uploads for nearby changing chunks to prevent visual voids caused by asynchronous mesh generation [S3, unofficial]. It iterates over all chunks, calculating the distance from the chunk center to the camera. If the distance is less than 512 units and the chunk is dirty/changing, it marks the chunk for upload blocking. It queues SVL/mesh updates in separate arrays (max 16). If a chunk is blocked for more than 0.1 seconds or until a specific time, it forces an unblock. Otherwise, it accumulates the blocked time. When unblocked, it processes the queued updates [S3, unofficial]. The condition for near chunks is `distance < 512`. Parameters include `MAX_UPLOADS: 16`, near distance threshold `512`, and max block duration `0.1` seconds [S3, unofficial].

### Concurrent Chunk Mesh Queue Processing
Concurrent Chunk Mesh Queue Processing uses a thread-safe concurrent hash map to track chunks and a lock-free queue to process incoming mesh data, allowing asynchronous mesh generation while rendering occurs on a single thread [S4, unofficial]. It processes a deletion queue by popping indices, finding chunks in the map, and calling `deallocateBuffers()` and `delete render_info` before erasing from the map [S4, unofficial]. A key pitfall is that rendering must occur on a single thread because OpenGL context and state changes are not thread-safe across multiple draw calls in this setup [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** Mesh generation is $O(N)$ where $N$ is the number of voxels in the chunk, but this work is distributed across frames and threads. The main thread overhead is $O(1)$ per frame for queue management and result processing, bounded by `NUMBER_OF_CHUNKS_TO_CREATE_PER_FRAME` [S1, unofficial].
*   **Memory Complexity:** $O(C)$ where $C$ is the number of chunks in the render distance, plus overhead for in-flight futures and queued mesh data [S1, unofficial].
*   **Performance:** Prevents frame stutter by smoothing out mesh generation spikes. Performance depends on the number of worker threads and the efficiency of the meshing algorithm (e.g., greedy meshing vs. simple face culling) [S2, unofficial].
*   **Strengths:** Maintains high frame rates during world generation or chunk streaming. Decouples heavy computation from the render loop.
*   **Limitations:** Introduces latency between chunk request and visibility. Requires careful synchronization to avoid race conditions or duplicate work. Not suitable for real-time, single-chunk edits that require immediate visual feedback without a separate fast-path mechanism [S1, unofficial].
*   **When to use:** Large open worlds with streaming chunks, procedural generation.
*   **When not to use:** Small, static worlds where all chunks are loaded at startup, or when immediate visual feedback for every voxel change is critical without a separate edit queue.

## Background
Asynchronous Chunk Loading originated in the context of voxel engine development, particularly in engines like Unreal Engine 5, where thread pools and futures are standard mechanisms for offloading work [S1, unofficial]. It is a common pattern in games like Minecraft clones to handle the dynamic generation of terrain. The technique has been adapted in various C++ and Web-based voxel engines to balance CPU load and maintain responsiveness [S2, unofficial][S4, unofficial].

## Sources
[S1] Manolo94/UE5MinecraftClone, https://github.com/Manolo94/UE5MinecraftClone, fetched 2026-10-26 (unofficial)
[S2] rtarun9/voxel-engine, https://github.com/rtarun9/voxel-engine, fetched 2026-10-27 (unofficial)
[S3] kosshi-net/voxplat, https://github.com/kosshi-net/voxplat, fetched 2026-10-27 (unofficial)
[S4] EmaMaker/voxel-engine, https://github.com/EmaMaker/voxel-engine, fetched 2026-10-26 (unofficial)
