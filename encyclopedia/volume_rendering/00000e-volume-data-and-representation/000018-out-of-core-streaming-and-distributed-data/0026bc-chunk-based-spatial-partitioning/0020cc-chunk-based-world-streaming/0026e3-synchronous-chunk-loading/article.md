# Synchronous Chunk Loading

Also called: ForceLoadChunksAt

## Overview
Synchronous Chunk Loading is a method that synchronously generates and applies a 3x3 grid of chunks around a given world position [S1, unofficial]. It ensures local geometry and collision are ready before returning, differing from asynchronous streaming variants by executing all generation and mesh preparation on the calling thread [G].

## How it works
**Inputs:**
- `WorldPos`: The 3D world coordinate to center the loading grid on [G].
- `WorldMesh`: A reference to the actor or manager responsible for rendering chunk meshes [G].
- `ChunkCache`: A data structure mapping chunk indices to generated chunk data [G].
- `ActiveChunks`: A set or list of chunk indices currently rendered and enabled [G].
- `ModifiedBlocks`: A collection of pending block modifications to apply to generated data [G].
- `GenParams`: Parameters passed to the chunk generator [G].

**Data Structures:**
- `ChunkIndex`: A 2D coordinate pair `(X, Y)` representing the chunk grid position [G].
- `ChunkData`: The raw voxel or geometry data generated for a chunk [G].
- `ChunkMesh`: The prepared renderable mesh for a single chunk [G].

**Steps:**
1. **Validation:** Check if `WorldMesh` exists. If not, return immediately [S1, unofficial].
2. **Center Calculation:** Compute the center chunk index for the given world position: `Center = WorldToChunkIndex(WorldPos)` [S1, unofficial].
3. **Iteration:** Loop through a 3x3 grid relative to the center. Let $DX$ range from $-1$ to $1$ and $DY$ range from $-1$ to $1$ [S1, unofficial].
4. **Index Calculation:** For each iteration, calculate the target chunk index: `Index = (Center.X + DX, Center.Y + DY)` [S1, unofficial].
5. **Skip Active:** If `Index` is present in `ActiveChunks`, continue to the next iteration [S1, unofficial].
6. **Generation:** If `Index` is not in `ChunkCache`, generate the chunk data: `Data = FWorldGenerator::GenerateChunk(GenParams, Index)` [S1, unofficial].
7. **Modification:** Apply any pending changes from `ModifiedBlocks` to the generated `Data` [S1, unofficial].
8. **Caching:** Store the processed `Data` in `ChunkCache` under `Index` [S1, unofficial].
9. **Mesh Preparation:** Retrieve `Data` from `ChunkCache` and prepare the mesh: `Mesh = AWorldMeshActor::PrepareChunkMesh(*Data)` [S1, unofficial].
10. **Application:** Apply the mesh to the world: `WorldMesh->ApplyChunk(Index, Mesh)` [S1, unofficial].
11. **Collision:** Enable collision for the chunk: `WorldMesh->EnableCollision(Index)` [S1, unofficial].
12. **Activation:** Add `Index` to `ActiveChunks` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def SynchronousChunkLoading(WorldPos, WorldMesh, ChunkCache, ActiveChunks, ModifiedBlocks, GenParams):
    # Step 1: Validation
    if WorldMesh is None:
        return

    # Step 2: Center Calculation
    # WorldToChunkIndex: Not given in the sources. [G]
    Center = WorldToChunkIndex(WorldPos)

    # Step 3-4: Iteration and Index Calculation
    for DX in range(-1, 2):
        for DY in range(-1, 2):
            Index = (Center[0] + DX, Center[1] + DY)

            # Step 5: Skip Active
            if Index in ActiveChunks:
                continue

            # Step 6-8: Generation, Modification, Caching
            if Index not in ChunkCache:
                # FWorldGenerator::GenerateChunk: Not given in the sources. [G]
                Data = GenerateChunk(GenParams, Index)
                # ApplyModifications: Not given in the sources. [G]
                Data = ApplyModifications(Data, ModifiedBlocks)
                ChunkCache[Index] = Data
            else:
                Data = ChunkCache[Index]

            # Step 9: Mesh Preparation
            # AWorldMeshActor::PrepareChunkMesh: Not given in the sources. [G]
            Mesh = PrepareChunkMesh(Data)

            # Step 10: Application
            WorldMesh.ApplyChunk(Index, Mesh)

            # Step 11: Collision
            WorldMesh.EnableCollision(Index)

            # Step 12: Activation
            ActiveChunks.add(Index)
```

**Parameters:**
- `WorldPos`: The player or camera position.
- `GenParams`: Generator configuration. Typical values: Not given in the sources. [G]

**Edge Cases:**
- **Missing WorldMesh:** The function returns immediately if the mesh actor is not initialized [S1, unofficial].
- **Degenerate Input:** If `WorldPos` is invalid, `WorldToChunkIndex` behavior is undefined [G].

## Subtopics

### [Progressive Splat Loading (Direct to Uncompressed Array)](7122-progressive-splat-loading-direct-to-uncompressed-array.md)
Progressive Splat Loading is a method for ingesting chunk-based splat data into an uncompressed array during synchronous chunk loading, enabling immediate rendering of available data without waiting for full chunk assembly. It differs from standard chunk loading by parsing splats directly from a streaming buffer into a global array as bytes arrive, rather than holding them in a temporary structure until the entire chunk is received. This reduces latency for initial visibility but requires careful management of partial data and memory layout. [G]

## Operating characteristics
- **Time Complexity:** Not given in the sources. [G]
- **Memory Complexity:** Not given in the sources. [G]
- **Performance:** Not given in the sources. [G]
- **Strengths:** Guarantees that the local area is fully rendered and collision-enabled before returning [G].
- **Limitations:** Blocks the calling thread until all chunks are generated and meshed [G].
- **When to use:** Not given in the sources. [G]
- **When not to use:** Not given in the sources. [G]

## Background
This method is derived from the `VoxelWorld` project by HawkDev15, which implements a voxel-based world generation system in Unreal Engine [S1, unofficial]. The specific function `ForceLoadChunksAt` is used to ensure chunks around the player are loaded synchronously [S1, unofficial].

## Sources
[S1] HawkDev15/VoxelWorld, https://github.com/HawkDev15/VoxelWorld, fetched 2026-10-28 (unofficial)
