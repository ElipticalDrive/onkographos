# Asynchronous Chunk Loading

Also called: Async Chunk Streaming, Async Voxel Generation, Chunk Task Queue, Multi-threaded Async Chunk Setup

## Overview
Asynchronous Chunk Loading offloads the computationally expensive process of generating chunk meshes from the main execution thread to a background thread pool, preventing frame stutter and maintaining frame rate in large voxel worlds [S1, unofficial]. It solves the problem of blocking the game loop during mesh generation by using futures and task queues to synchronize data transfer between the generation thread and the rendering or game thread [S1, unofficial][S2, unofficial]. Unlike synchronous loading, which halts execution until the mesh is ready, this method allows the player to continue interacting with the world while chunks are being generated in the background [S1, unofficial].

## How it works
The method relies on a producer-consumer pattern where the game thread submits generation tasks and consumes results, while a thread pool executes the heavy lifting.

**Inputs and Outputs**
*   **Input:** Player position (to determine which chunks to load), `ChunkRenderDistance` (integer radius for the loading square), and a `PopulateBlock` callback function (`TPopulateBlockFunc`) for procedural voxel data generation [S1, unofficial].
*   **Output:** Uploaded mesh data (vertices, indices, normals, UVs) attached to chunk actors in the scene graph [S1, unofficial].

**Data Structures**
1.  **Chunk Map (`ChunkMap`):** A spatial hash map (`TMap<int32, AChunk*>` or equivalent) storing references to currently loaded chunk actors, keyed by a hash of their chunk coordinates [S1, unofficial].
2.  **In-Flight Results Map (`chunkResults`):** A map (`TMap<int32, TFuture<...>*>` or equivalent) storing pointers to futures for chunks currently being generated. This prevents duplicate generation requests for the same chunk [S1, unofficial].
3.  **Result Queue (`chunkLoaderQueue`):** A thread-safe queue (`TQueue<TArray<MeshData*>>` or equivalent) used to pass generated mesh data from the thread pool back to the game thread [S1, unofficial].
4.  **Setup Stack (`m_chunks_to_setup_stack`):** A stack storing chunk indices that need setup, used to batch processing up to `NUMBER_OF_CHUNKS_TO_CREATE_PER_FRAME` per frame [S2, unofficial].

**Numbered Steps**
1.  **Detection:** The game thread detects that the player has moved into a new chunk or that the render distance has changed [S1, unofficial].
2.  **Iteration:** The game thread iterates through all chunk coordinates within a square radius defined by `ChunkRenderDistance` relative to the player's current chunk [S1, unofficial].
3.  **Queueing:** For each chunk coordinate $(i, j)$, the system checks if the chunk hash exists in `ChunkMap` or `chunkResults` and skips it if present [S1, unofficial]. If not, the index is added to `m_chunks_to_setup_stack` provided it is not already being set up or loaded [S2, unofficial].
4.  **Submission:** In `create_chunks_from_setup_stack`, the system pops up to `NUMBER_OF_CHUNKS_TO_CREATE_PER_FRAME` indices from the stack and submits a task to the thread pool for each index [S2, unofficial]. The task returns a `SetupChunkData` struct [S2, unofficial].
5.  **Task Execution (Background Thread):** The thread pool executes the task, calling `GetMeshDataForChunk(i, j, PopulateBlock)` [S1, unofficial]. This generates the voxel block data via the `PopulateBlock` callback and performs mesh extraction (face culling, UV mapping) [S1, unofficial]. The result is wrapped in a `TFuture` and stored in `chunkResults` keyed by the chunk hash [S1, unofficial].
6.  **Synchronization:** The background task completes and places the result into the `chunkLoaderQueue` [S1, unofficial].
7.  **Consumption (Game Thread):** On the game thread, the system dequeues results from `chunkLoaderQueue` and processes them one per frame [S1, unofficial]. Upon receiving a future result, it calls `AChunk::CreateChunk()` to instantiate the actor and upload the mesh data to the GPU [S1, unofficial]. The corresponding entry is removed from `chunkResults` [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import threading
import queue

# Data structures
ChunkMap: dict[int, 'AChunk'] = {}
chunkResults: dict[int, 'threading.Thread'] = {}
chunkLoaderQueue: queue.Queue = queue.Queue()
m_chunks_to_setup_stack: list[int] = []

# Parameters
ChunkRenderDistance: int = 16
NUMBER_OF_CHUNKS_TO_CREATE_PER_FRAME: int = 4
NUMBER_OF_CHUNKS_TO_LOAD_PER_FRAME: int = 1

def GetHash(i: int, j: int) -> int:
    return (i << 16) | (j & 0xFFFF)

def OnPlayerMove(newChunkX: int, newChunkY: int):
    # Iterate square radius around player
    for i in range(newChunkX - ChunkRenderDistance, newChunkX + ChunkRenderDistance + 1):
        for j in range(newChunkY - ChunkRenderDistance, newChunkY + ChunkRenderDistance + 1):
            chunkHash = GetHash(i, j)
            # Skip if already loaded or currently generating
            if chunkHash in ChunkMap or chunkHash in chunkResults:
                continue
            # Add to setup stack if not already queued
            if chunkHash not in m_chunks_to_setup_stack:
                m_chunks_to_setup_stack.append(chunkHash)

    # Process setup stack in batches
    for _ in range(NUMBER_OF_CHUNKS_TO_CREATE_PER_FRAME):
        if not m_chunks_to_setup_stack:
            break
        chunkHash = m_chunks_to_setup_stack.pop()
        # Extract coordinates from hash for task submission
        i, j = (chunkHash >> 16), chunkHash & 0xFFFF
        # Submit async task to thread pool
        future = threading.Thread(target=GenerateChunkMesh, args=(i, j, PopulateBlock))
        future.start()
        chunkResults[chunkHash] = future

    # Consume results one per frame
    for _ in range(NUMBER_OF_CHUNKS_TO_LOAD_PER_FRAME):
        if not chunkLoaderQueue.empty():
            chunkData = chunkLoaderQueue.get()
            CreateChunk(chunkData)

def GenerateChunkMesh(i: int, j: int, PopulateBlock):
    # 1. Generate voxel data via callback
    blocks = GetMeshDataForChunk(i, j, PopulateBlock)
    # 2. Extract mesh (face culling, UV mapping)
    meshData = ExtractMesh(blocks)
    # 3. Enqueue result for game thread
    chunkLoaderQueue.put(meshData)

def CreateChunk(chunkData):
    # Instantiate actor, upload vertices/indices/normals/UVs
    # Add to ChunkMap
    pass
```

**Parameters**
*   `ChunkRenderDistance`: Integer radius defining the square area of chunks to load [S1, unofficial].
*   `PopulateBlock`: A callback function signature (`TPopulateBlockFunc`) that provides procedural voxel data [S1, unofficial].
*   `NUMBER_OF_CHUNKS_TO_CREATE_PER_FRAME`: Maximum number of chunks to start setting up per frame [S2, unofficial].
*   `NUMBER_OF_CHUNKS_TO_LOAD_PER_FRAME`: Maximum number of chunks to finalize (move to loaded state) per frame [S2, unofficial].

**Edge Cases and Pitfalls**
*   **Player Chunk Skipping:** A common pitfall is that the iteration loop may skip the player's current chunk if the condition `if (i != 0 || j != 0)` is used incorrectly relative to the player's offset, leaving the player's immediate chunk unqueued if not handled explicitly elsewhere [S1, unofficial].
*   **Duplicate Generation:** Without checking `chunkResults`, rapid player movement or jitter could cause the same chunk to be generated multiple times, wasting CPU cycles [S1, unofficial].
*   **Thread Safety:** The `chunkLoaderQueue` and `chunkResults` map must be thread-safe or accessed only from their respective threads (game thread for queue consumption and map updates, worker thread for submission) [S1, unofficial].

## Subtopics

### Chunk Upload Blocking
A variant that delays GPU uploads for nearby changing chunks to prevent visual voids caused by asynchronous mesh generation [S3, unofficial]. It iterates over all chunks and calculates the distance from the chunk center to the camera [S3, unofficial]. If the distance is less than 512 units and the chunk is dirty or changing, it marks the chunk for blocking by setting `block_upload = 1` [S3, unofficial]. It queues SVL and mesh updates in separate arrays (`qs` and `qm`) with a maximum of 16 queued uploads (`MAX_UPLOADS`) [S3, unofficial]. If a chunk is blocked for more than 0.1 seconds or until a specific time threshold, it forces an unblock to prevent indefinite stalling [S3, unofficial]. When unblocked, it processes the queued updates [S3, unofficial].

### Concurrent Chunk Mesh Queue Processing
A variant that uses a thread-safe concurrent hash map to track chunks and a lock-free queue to process incoming mesh data, allowing asynchronous mesh generation while rendering occurs on a single thread [S4, unofficial]. It defines specific chunk states (e.g., `CHUNK_STATE_GENERATED`, `CHUNK_STATE_MESH_LOADED`) to manage the lifecycle [S4, unofficial]. It includes a deletion queue to safely deallocate buffers and remove chunks from the map when they are unloaded [S4, unofficial]. A key pitfall is that rendering must occur on a single thread because OpenGL context and state changes are not thread-safe across multiple draw calls in this setup, preventing the use of `parallel_for` for rendering [S4, unofficial].

### Asynchronous Chunk Generation Scheduling
A variant that processes a queue of pending chunk positions, generating voxel data and optionally rendering them, with support for async batching [S5, unofficial]. It retrieves `pendingChunks` and calculates a processing count based on a boolean flag `this.asyncChunkGeneration` [S5, unofficial]. If the flag is false, it processes all pending chunks synchronously; if true, it processes 10% of the queue per tick, clamped between 1 and the total length [S5, unofficial]. It parses chunk positions stored as `'x|y|z'` strings using `split('|')` and bitwise OR `|0` to ensure integer indices [S5, unofficial]. If the calculated count is 0, no chunks are processed, preventing UI freezes during heavy generation [S5, unofficial].

## Operating characteristics
*   **Time Complexity:** Mesh generation is offloaded, so the game thread's time per frame is dominated by queue consumption and actor creation, which is $O(N)$ where $N$ is the number of chunks completed in that frame [S1, unofficial]. The total time for world generation is distributed across multiple frames, smoothing out latency spikes [S1, unofficial].
*   **Memory Complexity:** Requires memory for the voxel block data during generation, the mesh data structures (vertices, indices, etc.), and the queue buffers [S1, unofficial]. The `ChunkMap` and `chunkResults` map add overhead proportional to the render distance [S1, unofficial].
*   **Strengths:** Eliminates frame stutter associated with synchronous mesh generation and allows for responsive gameplay even in large worlds [S1, unofficial].
*   **Limitations:** No inter-chunk face culling is performed in the base implementation, leading to redundant geometry at chunk boundaries [S1, unofficial].
*   **When to use:** Essential for real-time voxel worlds with dynamic generation or editing where frame rate stability is critical [S1, unofficial].
*   **When not to use:** For static, pre-generated worlds where all chunks can be loaded during a loading screen, or for very small worlds where synchronous generation is negligible [S1, unofficial].

## Background
Asynchronous Chunk Loading originated in the context of Unreal Engine 5 voxel world implementations, specifically in projects like the UE5 Minecraft Clone, to address the performance bottlenecks of generating chunk meshes on the main thread [S1, unofficial]. It builds upon the concept of chunked world management and mesh rebuild queues, extending them with thread pool integration and future-based synchronization [S1, unofficial].

## Sources
[S1] Manolo94/UE5MinecraftClone, https://github.com/Manolo94/UE5MinecraftClone, fetched 2026-10-26 (unofficial)
[S2] rtarun9/voxel-engine, https://github.com/rtarun9/voxel-engine, fetched 2026-10-27 (unofficial)
[S3] kosshi-net/voxplat, https://github.com/kosshi-net/voxplat, fetched 2026-10-27 (unofficial)
[S4] EmaMaker/voxel-engine, https://github.com/EmaMaker/voxel-engine, fetched 2026-10-26 (unofficial)
[S5] voxel/voxel-engine-stackgl, https://github.com/voxel/voxel-engine-stackgl, fetched 2026-10-27 (unofficial)
