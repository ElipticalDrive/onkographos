# Chunk-Based Parallel Voxel Culling & Rendering
Also called: Concurrent Chunk Culling, Parallel Chunk Rendering
## Overview
This method accelerates voxel rendering by partitioning the world into independent chunks, performing face culling and instance collection concurrently, and batching visible geometry per chunk for GPU submission [S1, unofficial]. It differs from [Remesh Budget](3119-remesh-budget.md) by focusing on parallel spatial partitioning and immediate culling of interior faces, rather than dynamic mesh simplification based on complexity budgets [G].
## How it works
**Inputs:** Not given in the sources.
**Outputs:** Not given in the sources.

**Data Structures:**
*   `chunksToRender`: Dictionary mapping `Vector3` (chunk position) to `ShapeInstance[]` (array of visible block instances) [S1, unofficial].
*   `ShapeInstance`: Contains face visibility flags (`faceBack`, `faceFront`, `faceTop`, `faceBottom`, `faceLeft`, `faceRight`), tint colors for each face (`Vector3`), and a reference to the `IShape` geometry [S1, unofficial].
*   `renderDataByBlock`: Static array indexed by block ID, storing render metadata [S1, unofficial].

**Steps:**
1. Collect visible instances per chunk into a dictionary `chunksToRender` keyed by chunk position [S1, unofficial].
2. Execute culling tasks concurrently and wait for completion [S1, unofficial].
3. Iterate over completed chunks and pass their instance arrays to the shader for rendering [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
chunksToRender = {}
tasks = []
for chunk_pos in world_chunks:
    def cull_chunk(cp):
        visible_blocks = []
        for block in get_voxels_in_chunk(cp):
            for face in [Direction.Front, Direction.Back, Direction.Top, Direction.Bottom, Direction.Left, Direction.Right]:
                if is_face_visible(block, face):
                    visible_blocks.append(create_shape_instance(block, face))
        return visible_blocks
    tasks.append(spawn_task(cull_chunk, chunk_pos))
wait_for_all(tasks)
for chunk_pos, blocks in chunksToRender.items():
    shader.RenderChunk(chunk_pos, blocks)
```

**Parameters:**
*   Chunk size: Not given in the sources.
*   Parallelism: Number of threads equals number of chunks or hardware cores [G].

**Edge Cases:**
*   **Precision:** Standard floating-point coordinates for chunk positions [G].

## Subtopics

### [Voxel Face Culling with Chunked Rendering Order](4159-voxel-face-culling-with-chunked-rendering-order.md)
Generates mesh faces only when adjacent to empty voxels or chunk boundaries, iterating in camera-relative front-to-back order to optimize GPU depth sorting. It eliminates interior geometry and handles chunk boundaries dynamically. Choose this when GPU depth sorting is a bottleneck.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Reduces GPU draw calls by batching visible faces per chunk [G].
*   **Strengths:** High parallelism, efficient culling of interior faces [G].
*   **Limitations:** Requires managing chunk synchronization [G].
*   **When to use:** Large voxel worlds with high update rates or large static areas [G].
*   **When not to use:** Small worlds where overhead outweighs benefits [G].
## Background
Originated in community-driven voxel rendering projects, specifically the `Cubic-Grid_Voxel_Renderer` repository. It represents an optimization strategy for real-time voxel engines using modern parallel processing techniques [S1, unofficial].
## Sources
[S1] rarefushion/Cubic-Grid_Voxel_Renderer, https://github.com/rarefushion/Cubic-Grid_Voxel_Renderer, fetched 2026-10-27 (unofficial)
