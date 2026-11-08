# Chunk Mesh Rebuild Queue

Also called: ProcessRebuildQueue, QueueRebuild

## Overview
A queue-based system to batch and process chunk mesh rebuilds asynchronously or on tick [S1, unofficial]. It solves the problem of redundant mesh regeneration by ensuring each modified chunk is processed exactly once per frame or tick cycle [S1, unofficial]. Unlike immediate synchronous rebuilding, which can cause frame drops during rapid block placement, this method decouples modification detection from geometry generation [S1, unofficial].

## How it works
**Inputs:**
- `RebuildQueue`: A dynamic array or stack storing `ChunkIndex` values (integer pairs/tuples) pending mesh regeneration.
- `RebuildSet`: A hash set or boolean map storing `ChunkIndex` values currently in the queue to prevent duplicates.
- `ActiveChunks`: A set of chunk indices currently loaded and visible in the world.
- `ChunkCache`: A data structure mapping `ChunkIndex` to raw voxel block data.
- `WorldMesh`: The rendering component responsible for managing chunk meshes.

**Outputs:**
- Updated meshes for chunks in `RebuildQueue`.
- Collision state updates for chunks near the player.

**Data Structures:**
- `ChunkIndex`: Typically a 2D integer coordinate $(x, z)$ or a flattened integer ID.
- `RebuildQueue`: LIFO stack (pop from end).
- `RebuildSet`: $O(1)$ lookup structure.

**Steps:**
1. **Enqueue Modification:** When a chunk is modified, check if its `ChunkIndex` exists in `RebuildSet`. If not, add `ChunkIndex` to both `RebuildSet` and `RebuildQueue` [S1, unofficial].
2. **Process Loop:** Check if `RebuildQueue` is empty or `WorldMesh` is null. If so, terminate [S1, unofficial].
3. **Pop Task:** Retrieve the last element from `RebuildQueue`: `ChunkIndex = RebuildQueue.Last()`. Remove it from the queue: `RebuildQueue.Pop()`. Remove `ChunkIndex` from `RebuildSet` [S1, unofficial].
4. **Validate Activity:** Check if `ChunkIndex` is present in `ActiveChunks`. If not, discard and return to step 2 [S1, unofficial].
5. **Retrieve Data:** Fetch chunk data from `ChunkCache` using `ChunkIndex`. If data is missing, discard and return to step 2 [S1, unofficial].
6. **Prepare Mesh:** Call `AWorldMeshActor::PrepareChunkMesh(*Data)` to generate the geometry from the raw block data [S1, unofficial].
7. **Update Mesh:** Call `WorldMesh->UpdateChunk(ChunkIndex, Mesh)` to replace the old mesh with the new one [S1, unofficial].
8. **Collision Check:** Determine if the chunk is near the player (within 1 chunk in X and Y axes). If true, enable collision for the chunk [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ProcessRebuildQueue(RebuildQueue, RebuildSet, ActiveChunks, ChunkCache, WorldMesh):
    # Step 2: Check termination conditions
    if not RebuildQueue or WorldMesh is None:
        return

    # Step 3: Pop last element
    ChunkIndex = RebuildQueue[-1]
    RebuildQueue.pop()
    RebuildSet.remove(ChunkIndex)

    # Step 4: Validate active status
    if ChunkIndex not in ActiveChunks:
        return

    # Step 5: Retrieve data
    Data = ChunkCache.get(ChunkIndex)
    if Data is None:
        return

    # Step 6: Prepare mesh
    Mesh = PrepareChunkMesh(Data)

    # Step 7: Update mesh
    WorldMesh.UpdateChunk(ChunkIndex, Mesh)

    # Step 8: Collision check
    if IsNearPlayer(ChunkIndex, PlayerPosition):
        EnableCollision(ChunkIndex)

def IsNearPlayer(ChunkIndex, PlayerPosition):
    # Check if within 1 chunk in X and Y
    return abs(ChunkIndex.x - PlayerPosition.x) <= 1 and abs(ChunkIndex.z - PlayerPosition.z) <= 1
```

**Edge Cases:**
- **Duplicate Modifications:** The `RebuildSet` prevents adding the same chunk to the queue multiple times if multiple blocks change in the same frame [S1, unofficial].
- **Unloaded Chunks:** Chunks modified while outside the active view are skipped if they are not in `ActiveChunks` during processing [S1, unofficial].
- **Null Mesh:** If `WorldMesh` is not initialized, the queue processing halts immediately to prevent crashes [S1, unofficial].

## Subtopics

### [Asynchronous Chunk Loading](4216-asynchronous-chunk-loading.md)
Offloads mesh generation to background threads to prevent frame stutter. Choose when frame rate stability is critical.

### [Multithreaded Job Queue (Generation & Remeshing)](6848-multithreaded-job-queue-generation-remeshing.md)
Distributes tasks across CPU threads using LIFO priority. Choose for high-performance CPU utilization; builds on this method.

## Operating characteristics
- **Time Complexity:** $O(1)$ for enqueue and dequeue operations. Mesh generation is $O(N)$ where $N$ is the number of blocks in the chunk.
- **Memory Complexity:** $O(M)$ where $M$ is the number of modified chunks in the queue.
- **Performance:** Reduces per-frame overhead by batching updates. Prevents blocking the main thread if combined with asynchronous loading [S1, unofficial].
- **Strengths:** Simple implementation, prevents redundant work, easy to integrate with existing chunk systems.
- **Limitations:** Does not inherently handle parallelism (requires external threading). Collision updates are synchronous with mesh updates.

## Background
Originated in voxel world management systems to handle dynamic terrain changes efficiently [S1, unofficial]. It is commonly used in sandbox games where players frequently modify the environment [S1, unofficial].

## Sources
[S1] HawkDev15/VoxelWorld, https://github.com/HawkDev15/VoxelWorld, fetched 2026-10-28 (unofficial)
