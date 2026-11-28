# Adjacency-Based Mesh Rebuilding

Also called: Incremental Section Mesh Update, Section Rebuild

## Overview
Adjacency-Based Mesh Rebuilding is a method for maintaining chunked voxel meshes by regenerating geometry only for the chunk containing a modified voxel and any immediately adjacent chunks that share a face with it [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** The local coordinates $(lx, ly, lz)$ of the modified voxel within its chunk, the world coordinates $(wx, wy, wz)$ of that voxel, and the chunk dimensions $(CHUNK\_SIZE)$ [S1, unofficial].
*   **Output:** Updated mesh data structures for the affected chunks [S1, unofficial].

**Data Structures:**
*   **Chunk:** A cubic block of voxels with dimensions $CHUNK\_SIZE \times CHUNK\_SIZE \times CHUNK\_SIZE$ [S1, unofficial].
*   **Voxel Position:** Represented as local $(lx, ly, lz)$ relative to the chunk origin and world $(wx, wy, wz)$ relative to the global grid [S1, unofficial].

**Steps:**
1. Get the local position $(lx, ly, lz)$ and world position $(wx, wy, wz)$ of the modified voxel [S1, unofficial].
2. If $lx == 0$, rebuild the adjacent chunk at $(wx - 1, wy, wz)$ [S1, unofficial].
3. If $lx == CHUNK\_SIZE - 1$, rebuild the adjacent chunk at $(wx + 1, wy, wz)$ [S1, unofficial].
4. If $ly == 0$, rebuild the adjacent chunk at $(wx, wy - 1, wz)$ [S1, unofficial].
5. If $ly == CHUNK\_SIZE - 1$, rebuild the adjacent chunk at $(wx, wy + 1, wz)$ [S1, unofficial].
6. If $lz == 0$, rebuild the adjacent chunk at $(wx, wy, wz - 1)$ [S1, unofficial].
7. If $lz == CHUNK\_SIZE - 1$, rebuild the adjacent chunk at $(wx, wy, wz + 1)$ [S1, unofficial].
8. Rebuild the mesh of the chunk containing the modified voxel [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def rebuild_adjacent_chunks(modified_voxel_local_pos, modified_voxel_world_pos, chunk_size):
    lx, ly, lz = modified_voxel_local_pos
    wx, wy, wz = modified_voxel_world_pos

    # Rebuild the chunk containing the modified voxel
    generate_chunk_mesh((wx, wy, wz))

    # Check X boundaries
    if lx == 0:
        generate_chunk_mesh((wx - 1, wy, wz))
    if lx == chunk_size - 1:
        generate_chunk_mesh((wx + 1, wy, wz))

    # Check Y boundaries
    if ly == 0:
        generate_chunk_mesh((wx, wy - 1, wz))
    if ly == chunk_size - 1:
        generate_chunk_mesh((wx, wy + 1, wz))

    # Check Z boundaries
    if lz == 0:
        generate_chunk_mesh((wx, wy, wz - 1))
    if lz == chunk_size - 1:
        generate_chunk_mesh((wx, wy, wz + 1))

def generate_chunk_mesh(chunk_coords):
    # Standard mesh generation for the chunk at chunk_coords
    pass
```

**Parameters:**
*   $CHUNK\_SIZE$: The dimension of a single chunk in voxels. Not given in the sources.

**Edge Cases:**
*   **Boundary Checks:** The logic relies on exact equality with $0$ and $CHUNK\_SIZE - 1$ [S1, unofficial]. If a voxel is not on the boundary, no adjacent chunks are rebuilt [S1, unofficial].
*   **Degenerate Input:** Not given in the sources.

## Subtopics

### [Chunk Mesh Building Pipeline](10914-chunk-mesh-building-pipeline.md)
Orchestrates the full incremental reconstruction including lighting and vegetation updates, managing state transitions and neighbor synchronization. Choose this when the rebuild cycle requires more than just geometry regeneration.

## Operating characteristics

## Background

## Sources
[S1] StanislavPetrovV/Minecraft, https://github.com/StanislavPetrovV/Minecraft, fetched 2026-10-26 (unofficial)
