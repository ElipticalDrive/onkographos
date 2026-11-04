# Remesh Budget

Also called: RemeshQueue

## Overview
Remesh Budget is a rendering optimization for chunked voxel worlds that limits the computational cost of mesh regeneration per frame by processing only the most visually significant dirty chunks [S1]. It solves the problem of frame-time spikes caused by rebuilding geometry for distant or less important chunks by enforcing a strict time budget, ensuring that mesh updates are prioritized based on camera proximity [S1]. It differs from sibling methods like [Chunk-Based Parallel Voxel Culling & Rendering](3994-chunk-based-parallel-voxel-culling-rendering.md) by focusing on temporal budgeting and spatial prioritization of rebuilds rather than parallel culling strategies [G].

## How it works
**Inputs and Outputs:**
- Input: A list of dirty chunks requiring mesh regeneration [S1].
- Input: The current camera position [S1].
- Input: The frame time budget allocated for remeshing [S1].
- Output: A set of chunks whose meshes have been regenerated and prepared for rendering [S1].
- Output: The remaining frame time budget [G].

**Data Structures:**
- `RemeshQueue`: A priority queue storing dirty chunks, ordered by distance to the camera [S1].
- `Chunk`: Represents a sub-world section containing geometric data and a dirty flag [S1].
- Exact field layouts, sizes, and bit packing are not given in the sources. [G]

**Steps:**
1. Identify all chunks marked as dirty [S1].
2. Insert all dirty chunks into the `RemeshQueue`, ordered by distance to the camera, with the nearest chunk at the front [S1].
3. Initialize the time budget for remeshing to 3 ms per frame [S1].
4. While the `RemeshQueue` is not empty and the time spent on remeshing is less than the budget: dequeue the nearest chunk, generate its mesh, upload the mesh data, and mark the chunk as clean [S1].
5. Constraint: Always mesh at least one chunk per frame, even if the budget is exhausted immediately after the first chunk [S1].

**Parameters:**
- `Frame Budget`: 3 ms per frame [S1]. This is a fixed value in the reference implementation [S1].

**Edge Cases and Pitfalls:**
- Budget Exhaustion: If the budget is exhausted after processing one chunk, the remaining dirty chunks are left for subsequent frames [S1].
- No Dirty Chunks: If there are no dirty chunks, no remeshing occurs, and the full budget is unused [G].
- Precision: Time measurement should use high-resolution timers to accurately track the 3 ms budget [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import time

def remesh_budget(dirty_chunks, camera_position, frame_time_budget_ms=3.0):
    """
    Remesh chunks within a time budget, prioritizing proximity to the camera.
    
    Args:
        dirty_chunks: List of Chunk objects that need remeshing.
        camera_position: Vector3 representing the camera's position.
        frame_time_budget_ms: Time budget in milliseconds for remeshing.
        
    Returns:
        List of Chunk objects that were successfully remeshed.
    """
    if not dirty_chunks:
        return []

    # Step 2: Sort by proximity to camera (nearest first)
    sorted_chunks = sorted(dirty_chunks, key=lambda c: distance(c.center, camera_position))

    remeshed_chunks = []
    time_spent_ms = 0.0

    # Step 4 & 5: Process Queue, ensuring at least one chunk is processed
    for i, chunk in enumerate(sorted_chunks):
        # Always process at least one chunk; break if budget exceeded for subsequent chunks
        if i > 0 and time_spent_ms >= frame_time_budget_ms:
            break

        start_time = time.time()
        generate_chunk_mesh(chunk)
        upload_mesh_to_gpu(chunk.mesh)
        time_spent_ms += (time.time() - start_time) * 1000
        chunk.dirty = False
        remeshed_chunks.append(chunk)

    return remeshed_chunks

def distance(pos1, pos2):
    """Calculate Euclidean distance between two points."""
    dx = pos1.x - pos2.x
    dy = pos1.y - pos2.y
    dz = pos1.z - pos2.z
    return (dx*dx + dy*dy + dz*dz) ** 0.5

def generate_chunk_mesh(chunk):
    # Mesh generation logic (panels, strips, triangles, caps) not specified in sources [G]
    pass

def upload_mesh_to_gpu(mesh):
    # Standard GPU buffer upload operation not specified in sources [G]
    pass
```

## Subtopics

### [Chunk Mesh Rebuild Queue](3911-chunk-mesh-rebuild-queue.md)
A queue-based system to batch and process chunk mesh rebuilds asynchronously or on tick. It differs from its parent [Remesh Budget](3119-remesh-budget.md) by providing the specific data structures and step-by-step logic for managing the dirty chunk list, while the parent provides the scheduling constraints (time budget and distance priority) [G].

## Operating characteristics
- **Time Complexity:** Not given in the sources. [G]
- **Memory Complexity:** Not given in the sources. [G]
- **Measured Performance:** In the "Outpost" scene, which consists of 4 chunks and 10k triangles, the method ensures that remeshing stays within the 3 ms budget [S1]. The scene results in 37 draws and 200k triangles on Low quality, 114 draws and 879k triangles on Balanced, and 153 draws and 1.87M triangles on Ultra quality [S1].
- **Strengths:** Guarantees frame rate stability by limiting remeshing time [S1]. Prioritizes visual fidelity for chunks near the camera [S1].
- **Limitations:** Not given in the sources. [G]
- **When to Use:** Not given in the sources. [G]
- **When Not to Use:** Not given in the sources. [G]

## Background
Remesh Budget was introduced in the "Claude Survivor Builder" project by Austin Bryan Price [S1]. It is part of Phase 2c of the project's development, focusing on optimizing the rendering of beveled chunk meshes with tiered tiles and ambient occlusion [S1]. The method is implemented in the `remesh-queue.ts` file within the project's source code [S1].

## Sources
[S1] Draw the blocks: beveled chunk meshing, tier tiles, remesh budget (Phase 2c), https://github.com/austinbryanprice-droid/Claude-Survivor-Builder/pull/8, fetched 2026-10-26
