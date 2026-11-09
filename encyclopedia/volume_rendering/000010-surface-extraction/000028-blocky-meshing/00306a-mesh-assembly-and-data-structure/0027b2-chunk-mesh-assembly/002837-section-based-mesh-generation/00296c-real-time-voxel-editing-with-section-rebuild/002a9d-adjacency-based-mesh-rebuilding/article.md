# Adjacency-Based Mesh Rebuilding

Also called: Incremental Section Mesh Update, Section Rebuild

## Overview
Adjacency-based mesh rebuilding is an incremental optimization for chunk-based voxel rendering that regenerates mesh geometry only for chunks adjacent to a modified voxel [S1, unofficial]. It solves the performance bottleneck of full-scene mesh regeneration by limiting updates to the immediate neighborhood of a block change, ensuring real-time responsiveness during voxel editing [S1, unofficial]. Unlike full rebuilds, it relies on boundary detection to determine which neighboring chunks require recalculation [S1, unofficial].

## How it works
The method triggers upon a voxel addition or removal. It identifies the local coordinates of the modified voxel within its chunk and determines if the voxel lies on any chunk boundary. If a boundary is crossed, the adjacent chunk sharing that face is marked for rebuild. Finally, the mesh for the chunk containing the modified voxel is regenerated [S1, unofficial].

**Inputs:**
- `chunk_size`: Integer dimension of the chunk (e.g., 16).
- `modified_voxel_pos`: World coordinates $(wx, wy, wz)$ of the changed voxel.
- `chunk_grid`: Map or array of active chunk renderers.

**Outputs:**
- Updated vertex/index buffers for affected chunks.

**Data Structures:**
- `ChunkRenderer`: Object managing a single chunk's mesh data. Contains flags `needsUpdate` and neighbor flags `neighborNeedUpdate` for WEST, EAST, SOUTH, NORTH [S2, unofficial].

**Steps:**
1. Calculate local voxel coordinates $(lx, ly, lz)$ from world coordinates:
   $$lx = wx \pmod{\text{chunk\_size}}$$
   $$ly = wy \pmod{\text{chunk\_size}}$$
   $$lz = wz \pmod{\text{chunk\_size}}$$
   [S1, unofficial]

2. Determine if the voxel is on a boundary. If $lx == 0$, the chunk at $(wx - \text{chunk\_size}, wy, wz)$ must be rebuilt [S1, unofficial]. If $lx == \text{chunk\_size} - 1$, the chunk at $(wx + \text{chunk\_size}, wy, wz)$ must be rebuilt [S1, unofficial]. Apply similar logic for $ly$ and $lz$ axes [S1, unofficial].

3. Rebuild the mesh for the chunk containing the modified voxel [S1, unofficial].

4. For each flagged neighbor chunk, perform the **Chunk Mesh Building Pipeline** [S2, unofficial]:
   a. Lock the weak pointer to the neighbor `ChunkRenderer` [S2, unofficial].
   b. Verify the neighbor has all adjacent chunks loaded via `hasAllAdjacentChunkLoaded()` [S2, unofficial].
   c. Call `buildMesh()` on the neighbor [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def rebuild_adjacent_chunks(modified_world_pos, chunk_size, chunk_grid):
    wx, wy, wz = modified_world_pos
    lx = wx % chunk_size
    ly = wy % chunk_size
    lz = wz % chunk_size

    # Identify neighbors to rebuild based on boundary conditions
    neighbors_to_rebuild = []

    if lx == 0:
        neighbors_to_rebuild.append((wx - chunk_size, wy, wz))
    if lx == chunk_size - 1:
        neighbors_to_rebuild.append((wx + chunk_size, wy, wz))
    
    if ly == 0:
        neighbors_to_rebuild.append((wx, wy - chunk_size, wz))
    if ly == chunk_size - 1:
        neighbors_to_rebuild.append((wx, wy + chunk_size, wz))
        
    if lz == 0:
        neighbors_to_rebuild.append((wx, wy, wz - chunk_size))
    if lz == chunk_size - 1:
        neighbors_to_rebuild.append((wx, wy, wz + chunk_size))

    # Rebuild the chunk containing the modified voxel
    current_chunk = chunk_grid.get_chunk_at(wx, wy, wz)
    if current_chunk:
        current_chunk.build_mesh()

    # Rebuild adjacent chunks if they exist and are valid
    for nx, ny, nz in neighbors_to_rebuild:
        neighbor_chunk = chunk_grid.get_chunk_at(nx, ny, nz)
        if neighbor_chunk:
            # Check if neighbor needs update and is fully loaded
            if neighbor_chunk.needs_update and neighbor_chunk.has_all_adjacent_chunk_loaded():
                neighbor_chunk.build_mesh()
```

## Subtopics

### Chunk Mesh Building Pipeline
This subtopic details the internal steps of `buildMesh()` for a single chunk [S2, unofficial]. It orchestrates lighting recomputation, palette decoding, mesh generation, GPU upload, vegetation building, and neighbor synchronization [S2, unofficial].
- **Lighting:** Skips full sky/block-light BFS only if `blockLightOnlyRebuild` is true AND the chunk already has sky and block light; otherwise, calls `computeSkyLight()` and `computeBlockLight()` [S2, unofficial].
- **GPU Upload:** Calls `uploadMesh()` to transfer vertex data to GPU buffers (VAO/VBO) [S2, unofficial].
- **Vegetation:** Calls `buildVegetationMesh()` to generate vegetation geometry [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ relative to the total scene size, proportional only to the surface area of the modified chunk and its immediate neighbors.
- **Memory Complexity:** $O(1)$ additional memory for temporary rebuild state per chunk.
- **Performance:** Enables real-time voxel editing by avoiding full-scene regeneration [S1, unofficial].
- **Limitations:** Requires robust neighbor tracking and boundary detection logic [S1, unofficial]. Does not handle non-adjacent changes (e.g., teleportation or large-scale terrain generation) without additional logic.

## Background
This method originates from implementations of real-time voxel editing in Minecraft-like engines, specifically focusing on efficient chunk management [S1, S2, unofficial]. It is used in engines requiring immediate feedback during block placement and removal, such as the `ft_minecraft` and `StanislavPetrovV/Minecraft` projects [S1, S2, unofficial].

## Sources
[S1] StanislavPetrovV/Minecraft, https://github.com/StanislavPetrovV/Minecraft, fetched 2026-10-26 (unofficial)
[S2] ldominiq/ft_minecraft, https://github.com/ldominiq/ft_minecraft, fetched 2026-10-27 (unofficial)
