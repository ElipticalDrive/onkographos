# Chunk Mesh Building Pipeline

Also called: buildMesh, buildMeshData, updateMesh

## Overview
The Chunk Mesh Building Pipeline orchestrates the incremental reconstruction of a chunk's geometry and lighting after voxel modifications, handling palette decoding, mesh generation, GPU upload, and neighbor synchronization [S1, unofficial]. It differs from static meshing by integrating lighting recomputation and vegetation updates into the rebuild cycle, and from simple adjacency rebuilding by managing the full state transition of a chunk including its interaction with adjacent chunk loaders [S1, unofficial].

## How it works
**Inputs:** A reference to the current `ChunkRenderer` instance, a list of modified voxels (implicit in the trigger), and access to neighbor chunk references.
**Outputs:** Updated GPU vertex/index buffers, updated lighting data (sky/block), and updated vegetation geometry.

**Data Structures:**
*   `ChunkRenderer`: Holds weak pointers to neighbor chunks (`WEST`, `EAST`, `SOUTH`, `NORTH`).
*   `needsUpdate`: Boolean flag indicating if the chunk requires rebuilding.
*   `neighborNeedUpdate`: Array of 6 booleans indicating if a specific neighbor requires an update due to adjacency.
*   `FACE_NORMALS`: Constant array of 6 unit vectors defining normals for the 6 cube faces [S1, unofficial].
    *   Index 0: `(0, 0, 1)`
    *   Index 1: `(0, 0, -1)`
    *   Index 2: `(0, 1, 0)`
    *   Index 3: `(0, -1, 0)`
    *   Index 4: `(1, 0, 0)`
    *   Index 5: `(-1, 0, 0)`

**Steps:**
1.  **Trigger Check:** When `updateMesh()` is called, first clear the `needsUpdate` flag to prevent redundant processing during the same frame [S1, unofficial].
2.  **Neighbor Synchronization:** Iterate through the 6 directional neighbors (WEST, EAST, SOUTH, NORTH, UP, DOWN). For each neighbor:
    *   Check if `neighborNeedUpdate[direction]` is true [S1, unofficial].
    *   If true, lock the weak pointer to the neighbor chunk.
    *   Cast the pointer to `ChunkRenderer`.
    *   Verify the neighbor has all its adjacent chunks loaded via `hasAllAdjacentChunkLoaded()` [S1, unofficial].
    *   If valid and loaded, call `buildMesh()` on the neighbor to ensure its boundary faces are updated [S1, unofficial].
3.  **Lighting Recomputation:** Inside `buildMesh()`:
    *   Determine if a full lighting BFS is required. Skip full sky and block light BFS only if `blockLightOnlyRebuild` is true AND the chunk already possesses valid sky and block light data [S1, unofficial].
    *   Otherwise, call `computeSkyLight()` and `computeBlockLight()` to propagate light values from modified voxels [S1, unofficial].
4.  **Mesh Generation:** Generate the vertex and index data for the chunk's visible faces. The vertex shader uses `gl_VertexID` to determine face normals: `face = (gl_VertexID / 6) % 6` [S1, unofficial].
5.  **GPU Upload:** Call `uploadMesh()` to transfer the generated vertex data to GPU buffers (VAO/VBO) [S1, unofficial].
6.  **Vegetation Update:** Call `buildVegetationMesh()` to generate or update geometry for vegetation entities within the chunk [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def update_mesh(self):
    # Step 1: Clear update flag
    self.needs_update = False
    
    # Step 2: Neighbor Synchronization
    directions = ['WEST', 'EAST', 'SOUTH', 'NORTH', 'UP', 'DOWN']
    for direction in directions:
        if self.neighbor_need_update[direction]:
            # Lock weak pointer and cast
            neighbor_chunk = self.get_neighbor_chunk(direction)
            if neighbor_chunk is not None:
                # Check if neighbor is fully loaded
                if neighbor_chunk.has_all_adjacent_chunk_loaded():
                    # Trigger rebuild on neighbor
                    neighbor_chunk.build_mesh()
    
    # Step 3: Lighting and Mesh Generation (called by trigger or neighbor)
    self.build_mesh()

def build_mesh(self):
    # Step 3: Lighting Recomputation
    # Skip full BFS only if specific condition met
    if self.block_light_only_rebuild and self.has_sky_light and self.has_block_light:
        pass # Skip BFS
    else:
        self.compute_sky_light()
        self.compute_block_light()
        
    # Generate mesh data (vertex/index buffers)
    # Uses FACE_NORMALS array for shading
    mesh_data = generate_chunk_mesh_data(self.voxels)
    
    # Step 4: GPU Upload
    self.upload_mesh(mesh_data)
    
    # Step 6: Vegetation
    self.build_vegetation_mesh()
```

**Edge Cases:**
*   **Weak Pointer Failure:** If a neighbor chunk is unloaded or destroyed, the weak pointer lock fails. The code must handle this gracefully by skipping the neighbor update [S1, unofficial].
*   **Incomplete Adjacency:** If a neighbor does not have all adjacent chunks loaded (`hasAllAdjacentChunkLoaded()` returns false), its boundary faces may be incorrect. The pipeline defers synchronization until the neighbor is fully loaded [S1, unofficial].
*   **Lighting State:** If `blockLightOnlyRebuild` is true but the chunk lacks existing light data, a full BFS is still required to avoid rendering artifacts [S1, unofficial].

## Subtopics

### Volume Meshing with Halo Bricks
A method to mesh a whole volume by including a one-cell halo across brick seams to handle culling and ambient occlusion (AO) for boundary faces [S2, unofficial]. It returns a `spall_jobs::JobToken` covering every brick revision and a missing-neighbour sentinel [S2, unofficial]. If a halo brick arrives, the mesh is marked as stale; when stale, the volume is re-meshed, which drops the now-occluded seam [S2, unofficial]. This approach ensures consistent AO and culling across brick boundaries but introduces a dependency on halo data availability [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of visible faces in the chunk. Neighbor synchronization adds a constant factor (up to 6 checks) per update [S1, unofficial].
*   **Memory Complexity:** Proportional to the chunk size (16x16x16 or similar) for voxel data and mesh buffers [S1, unofficial].
*   **Performance:** Incremental updates avoid full-volume regeneration, significantly reducing CPU load for frequent small changes [S1, unofficial].
*   **Strengths:** Handles lighting propagation and vegetation updates automatically; ensures neighbor consistency [S1, unofficial].
*   **Limitations:** Dependency on neighbor chunk loading state can delay updates; weak pointer management adds overhead [S1, unofficial].
*   **When to Use:** Real-time voxel editing where frequent small changes occur [S1, unofficial].
*   **When Not to Use:** Static scenes or full-volume rebuilds where halo-based methods might be more efficient [S2, unofficial].

## Background
The Chunk Mesh Building Pipeline originates from the `ft_minecraft` project, an unofficial implementation of Minecraft-like voxel rendering [S1, unofficial]. It was designed to support real-time voxel editing with efficient mesh updates, integrating lighting and vegetation into the meshing pipeline [S1, unofficial]. The halo-based variant is part of the `spall` project, focusing on visible voxel baselines with robust brick seam handling [S2, unofficial].

## Sources
[S1] ldominiq/ft_minecraft, https://github.com/ldominiq/ft_minecraft, fetched 2026-10-27 (unofficial)
[S2] T05 — Visible voxel baseline (spall_mesh, spall_render), https://github.com/deranjer/spall/pull/10, fetched 2026-10-27 (unofficial)
