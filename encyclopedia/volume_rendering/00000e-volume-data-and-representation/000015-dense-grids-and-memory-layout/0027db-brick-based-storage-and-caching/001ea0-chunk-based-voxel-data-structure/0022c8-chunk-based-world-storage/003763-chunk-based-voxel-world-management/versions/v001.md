# Chunk-based Voxel World Management

Also called: Chunk Visibility Culling and Incremental Rebuild

## Overview
This method manages a voxel world by dividing it into fixed-size chunks, handling visibility culling against a camera frustum, and incrementally rebuilding chunks that have changed [S1, unofficial]. It sets itself apart from sibling methods by combining frustum-based visibility determination with a rate-limited, incremental rebuild queue, ensuring that expensive mesh generation does not block the frame loop [S1, unofficial]. It builds on [Chunk-Based World Storage](2858-chunk-based-world-storage.md) by adding the logic for determining which chunks are active and need processing.

## How it works
**Inputs and Outputs:**
*   **Input:** World coordinates $(x, y, z)$, camera frustum, chunk grid dimensions, and a set of modified blocks.
*   **Output:** A set of chunks ready for rendering (`_chunkRenderSet`) and a set of chunks pending mesh regeneration (`_chunkRebuildSet`).

**Data Structures:**
*   `_chunks`: A `std::vector<Chunk*>` storing pointers to chunk objects. Chunks are accessed linearly using a 3D index mapped to 1D: `index = (x * _size) + (y * _size * _size) + z` [S1, unofficial].
*   `_chunkRenderSet`: A container (e.g., `std::set` or `std::unordered_set`) of chunks currently visible and ready for rendering [S1, unofficial].
*   `_chunkRebuildSet`: A container of chunks that have been modified and require mesh regeneration [S1, unofficial].
*   `_size`: The dimension of the cubic chunk grid. Determined by the maximum extent of the world in chunk coordinates [S1, unofficial].
*   `_rebuildsPerFrame`: Integer limiting the number of chunks rebuilt per frame. Typical value is 5 [S1, unofficial].

**Coordinate Mapping:**
To retrieve a chunk from world coordinates $(x, y, z)$:
1.  Calculate chunk indices:
    $$ \text{chunkX} = \lfloor x / \text{chunkRenderSize} \rfloor $$
    $$ \text{chunkY} = \lfloor y / \text{chunkRenderSize} \rfloor $$
    $$ \text{chunkZ} = \lfloor z / \text{chunkRenderSize} \rfloor $$
    Where $\text{chunkRenderSize} = \text{\_blocksPerChunk} \times \text{\_blockSize} \times 2$ [S1, unofficial].
2.  Clamp negative indices to 0:
    ```python
    if chunkX < 0: chunkX = 0
    if chunkY < 0: chunkY = 0
    if chunkZ < 0: chunkZ = 0
    ```
3.  Access the chunk via `_chunks[(chunkX * _size) + (chunkY * _size * _size) + chunkZ]` [S1, unofficial].

To retrieve a specific block from world coordinates:
1.  Calculate block indices:
    $$ \text{blockX} = \lfloor x / \text{blockRenderSize} \rfloor $$
    $$ \text{blockY} = \lfloor y / \text{blockRenderSize} \rfloor $$
    $$ \text{blockZ} = \lfloor z / \text{blockRenderSize} \rfloor $$
    Where $\text{blockRenderSize} = \text{\_blockSize} \times 2$ [S1, unofficial].
2.  Determine chunk and local block indices:
    $$ \text{chunkX} = \text{blockX} // \text{\_blocksPerChunk} $$
    $$ \text{localBlockX} = \text{blockX} \% \text{\_blocksPerChunk} $$
    (Similarly for Y and Z) [S1, unofficial].
3.  Retrieve the chunk, then the block within it [S1, unofficial].

**Algorithm Steps:**

1.  **Update Visibility List (`updateVisibilityList`):**
    Iterate through all chunks in the grid using three nested loops over the grid dimension `_size` for $i, j, k$.
    For each chunk:
    *   Get its bounding sphere: `bounds = chunk.getBounds()` [S1, unofficial].
    *   Check intersection with the camera frustum: `side = frustum.checkSphere(bounds)` [S1, unofficial].
    *   If `side` is `INSIDE` or `INTERSECT`:
        *   If the chunk is already set up (`isSetup()` is true), insert it into `_chunkRenderSet` [S1, unofficial].
        *   Otherwise, insert it into `_chunkRebuildSet` [S1, unofficial].

2.  **Rebuild Chunks (`rebuildChunks`):**
    Iterate through `_chunkRebuildSet` with a limit on the number of rebuilds per frame.
    *   Initialize `rebuiltCount = 0`.
    *   While `rebuiltCount < _rebuildsPerFrame` and `_chunkRebuildSet` is not empty:
        *   Select the next chunk from `_chunkRebuildSet`.
        *   Call `chunk->build()` to generate the mesh [S1, unofficial].
        *   Add the chunk to a temporary list `rebuiltList`.
        *   Insert the chunk into `_chunkRenderSet` [S1, unofficial].
        *   Increment `rebuiltCount`.
    *   After the loop, remove all chunks in `rebuiltList` from `_chunkRebuildSet` [S1, unofficial].

**Pitfalls:**
*   **Grid Dimension Calculation:** `_size` is determined by the maximum of $xSize, ySize, zSize$, where $xSize = \lfloor \text{\_blockX} / \text{chunkSize} \rfloor$. This ensures the grid covers the initial block coordinates but may result in a cubic grid even if the world is not cubic, wasting memory for non-cubic worlds [S1, unofficial].
*   **Negative Coordinates:** World coordinates can be negative. The chunk index calculation must clamp negative indices to 0 to prevent out-of-bounds access in the linear vector [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Visibility culling is $O(N)$ where $N$ is the total number of chunks in the grid. Rebuilding is $O(K)$ where $K$ is `_rebuildsPerFrame`, independent of the total number of chunks.
*   **Memory Complexity:** $O(N)$ for storing chunk pointers and metadata. The cubic grid allocation may use more memory than necessary if the world is not cubic [S1, unofficial].
*   **Performance:** By limiting rebuilds per frame, the method guarantees smooth frame rates even during heavy modification, at the cost of visual latency for distant or non-visible chunks.
*   **Strengths:** Predictable frame time; simple implementation; effective culling.
*   **Limitations:** Cubic grid inefficiency; potential visual popping if chunks are not pre-loaded; rebuild latency for large changes.

## Background
This method is derived from the `ChunkManager` implementation in the `VoxelEngine` project by nnarain [S1, unofficial]. It represents a common approach in voxel game development for managing dynamic worlds, balancing memory usage and rendering performance through spatial partitioning and incremental updates.

## Sources
[S1] nnarain/VoxelEngine, https://github.com/nnarain/VoxelEngine, fetched 2026-10-27 (unofficial)
