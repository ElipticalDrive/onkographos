# Chunk-Based Parallel Voxel Culling & Rendering
Also called: Concurrent Chunk Culling, Parallel Chunk Rendering
## Overview
This method accelerates voxel rendering by partitioning the world into independent chunks, performing face culling and instance collection concurrently, and batching visible geometry per chunk for GPU submission. It differs from [Remesh Budget](3119-remesh-budget.md) by focusing on parallel spatial partitioning and immediate culling of interior faces, rather than dynamic mesh simplification based on complexity budgets.
## How it works
**Inputs:** World voxel data, chunk grid structure, camera position.
**Outputs:** Rendered scene.

**Data Structures:**
*   `chunksToRender`: Dictionary mapping `Vector3` (chunk position) to `ShapeInstance[]` (array of visible block instances).
*   `ShapeInstance`: Contains face visibility flags (`faceBack`, `faceFront`, `faceTop`, `faceBottom`, `faceLeft`, `faceRight`), tint colors for each face (`Vector3`), and a reference to the `IShape` geometry.
*   `renderDataByBlock`: Static array indexed by block ID, storing render metadata.

**Steps:**
1.  **Partition:** Divide the voxel world into fixed-size chunks.
2.  **Parallel Culling:** For each chunk, execute a culling task concurrently.
    *   Iterate through voxels in the chunk.
    *   Determine face visibility: A face is visible if the adjacent voxel is empty or outside the chunk boundary (treated as empty) [S1, unofficial].
    *   Construct `ShapeInstance` objects for visible faces, including texture IDs and tints [S1, unofficial].
    *   Collect instances into a local list.
3.  **Aggregation:** Store the resulting instance arrays in `chunksToRender`, keyed by chunk position [S1, unofficial].
4.  **Synchronization:** Wait for all parallel culling tasks to complete [S1, unofficial].
5.  **Rendering:** Iterate over `chunksToRender`. For each chunk, pass the chunk position and its array of `ShapeInstance` objects to the shader for rendering [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Concurrent culling and collection
tasks = []
for chunk_pos in world_chunks:
    task = spawn_task(lambda: cull_chunk(chunk_pos))
    tasks.append(task)

# Wait for all tasks to finish
wait_for_all(tasks)

# Render collected chunks
for chunk_pos, blocks in chunksToRender.items():
    shader.RenderChunk(chunk_pos, blocks)

def cull_chunk(chunk_pos):
    visible_blocks = []
    for block in get_voxels_in_chunk(chunk_pos):
        # Check neighbors; treat out-of-bounds or unallocated chunks as empty
        if is_face_visible(block, Direction.Front):
            visible_blocks.append(create_shape_instance(block, Direction.Front))
        # ... repeat for other 5 directions
    return visible_blocks
```

**Parameters:**
*   Chunk size: Not given in the sources.
*   Parallelism: Number of threads equals number of chunks or hardware cores [G].

**Edge Cases:**
*   **Chunk Boundaries:** Unallocated neighbor chunks are treated as empty space, allowing immediate culling of inner faces at chunk edges [S1, unofficial].
*   **Precision:** Standard floating-point coordinates for chunk positions [G].

## Subtopics

### [Voxel Face Culling with Chunked Rendering Order](4159-voxel-face-culling-with-chunked-rendering-order.md)
Generates mesh faces only when adjacent to empty voxels or chunk boundaries, iterating in camera-relative front-to-back order to optimize GPU depth sorting. It eliminates interior geometry and handles chunk boundaries dynamically. Choose this when GPU depth sorting is a bottleneck.

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of voxels, parallelized across chunks.
*   **Memory Complexity:** $O(V)$ where $V$ is the number of visible voxels.
*   **Performance:** Reduces GPU draw calls by batching visible faces per chunk. Prevents non-visible geometry from reaching the GPU .
*   **Strengths:** High parallelism, efficient culling of interior faces.
*   **Limitations:** Requires managing chunk synchronization.
*   **When to use:** Large voxel worlds with high update rates or large static areas.
*   **When not to use:** Small worlds where overhead outweighs benefits.
## Background
Originated in community-driven voxel rendering projects, specifically the `Cubic-Grid_Voxel_Renderer` repository. It represents an optimization strategy for real-time voxel engines using modern parallel processing techniques [S1, unofficial].
## Sources
[S1] rarefushion/Cubic-Grid_Voxel_Renderer, https://github.com/rarefushion/Cubic-Grid_Voxel_Renderer, fetched 2026-10-27 (unofficial)
