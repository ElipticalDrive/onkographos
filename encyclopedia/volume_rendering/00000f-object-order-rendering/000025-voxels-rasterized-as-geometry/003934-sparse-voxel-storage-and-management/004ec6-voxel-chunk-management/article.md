# Voxel Chunk Management

Also called: Voxel Grid Block

## Overview
Voxel Chunk Management handles the storage, state tracking, and dirty-flag propagation for a fixed-size 3D block of voxel data within a sparse voxel architecture [S1, unofficial]. It solves the problem of efficiently determining when a chunk's geometric representation (mesh) or logical state (colliders, handlers) requires regeneration after voxel modifications, specifically by isolating boundary updates to prevent cascading recomputations across the voxel grid [S1, unofficial]. It differs from sibling methods by focusing on the internal block state and neighbor coordination rather than the global sparse structure (SVO) or the rasterization pipeline itself [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A 3D array of voxel IDs (uint16), chunk dimensions $N \times N \times N$, and a registry of voxel properties (solidity, opacity, object type, handler IDs).
*   **Output:** Updated chunk state flags (`_terrainDirty`, `_objectsDirty`), neighbor dirtiness propagation, and queued mesh regeneration requests.

**Data Structures**
The chunk maintains the following internal state [S1, unofficial]:
1.  `voxels`: A 3D array (e.g., `ndarray`) of shape `[size, size, size]` storing voxel IDs as `Uint16`.
2.  `_neighbors`: A 3x3x3 `ndarray` of neighbor chunk references. The center element `[1,1,1]` refers to the chunk itself. This allows O(1) access to adjacent chunks for boundary checks.
3.  `_wholeLayerVoxel`: An array of length `size` caching the voxel ID for entire layers (used for optimization during full-layer scans).
4.  `_terrainDirty`: Boolean flag indicating if the terrain mesh needs regeneration.
5.  `_objectsDirty`: Boolean flag indicating if object-specific meshes or handlers need updates.

**Parameters**
*   `size`: The dimension $N$ of the cubic chunk.
*   `fillVoxelID`: The default ID to initialize the voxel array with (commonly -1 for empty/air) [S1, unofficial].

**Algorithm Steps**

1.  **Initialization**:
    *   Allocate `voxels` array of size $N^3$.
    *   Initialize `_neighbors` array.
    *   Set `_terrainDirty` and `_objectsDirty` to `false`.
    *   Initialize `_wholeLayerVoxel` with -1.

2.  **Voxel Scan (Initial or Full Update)**:
    *   Iterate through all voxels $(i, j, k)$ in the chunk.
    *   Determine global chunk properties:
        *   `fullyOpaque`: True if all non-air voxels are opaque.
        *   `fullyAir`: True if all voxels are air (ID 0 or empty).
    *   Check for "constant layers": If an entire layer $j$ has the same voxel ID, cache it in `_wholeLayerVoxel[j]`.
    *   For each voxel ID:
        *   If it is an "object" voxel, register the chunk as an object block and set `_objectsDirty = true`.
        *   If it has block handlers, register the position $(i,j,k)$ and call `onLoad` handlers.
    *   If `fullyOpaque` or `fullyAir` status changes, set `_terrainDirty = true` [S1, unofficial].

3.  **Voxel Update (Single Voxel Change)**:
    *   When `set(i, j, k, newID)` is called:
        *   Update `voxels[i, j, k] = newID`.
        *   Check if the change affects solidity or opacity.
        *   If solidity/opacity changed, propagate dirtiness to neighboring chunks on the chunk boundary.
        *   Set `_terrainDirty = true` if the chunk's overall opacity/air status changed.
        *   Set `_objectsDirty = true` if the new/old ID involves object blocks or handlers [S1, unofficial].

4.  **Neighbor Dirtiness Propagation**:
    *   If a voxel at $(i, j, k)$ changes solidity or opacity, and it lies on a chunk boundary, notify adjacent chunks.
    *   Determine boundary offsets:
        *   If $i=0$, neighbor $i-1$ exists; if $i=N-1$, neighbor $i+1$ exists.
        *   Same for $j$ and $k$.
    *   For each existing neighbor chunk reference in `_neighbors`:
        *   Set `neighbor._terrainDirty = true`.
        *   Queue the neighbor chunk for remeshing [S1, unofficial].

5.  **Mesh Regeneration**:
    *   When `updateMeshes` is called:
        *   If `_terrainDirty`, regenerate terrain mesh and clear flag.
        *   If `_objectsDirty`, update object meshes/handlers and clear flag.

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class VoxelChunk:
    def __init__(self, size, fill_voxel_id=-1):
        self.size = size
        self.voxels = create_ndarray_3d(size, size, size, dtype=Uint16, fill=fill_voxel_id)
        self._neighbors = create_ndarray_3d(3, 3, 3, dtype=ChunkRef)
        self._neighbors[1, 1, 1] = self  # Center is self
        self._whole_layer_voxel = [-1] * size
        self._terrain_dirty = False
        self._objects_dirty = False
        self._block_handler_locs = set()

    def set_voxel(self, i, j, k, new_id):
        old_id = self.voxels[i, j, k]
        if old_id == new_id:
            return

        self.voxels[i, j, k] = new_id
        
        # Check if solidity or opacity changed
        solidity_changed = self._is_solid(old_id) != self._is_solid(new_id)
        opacity_changed = self._is_opaque(old_id) != self._is_opaque(new_id)

        # Update local dirty flags
        if solidity_changed or opacity_changed:
            self._terrain_dirty = True
        
        # Handle object/handler changes
        if self._is_object(old_id) != self._is_object(new_id):
            self._objects_dirty = True
            if self._is_object(new_id):
                self._register_object_block(i, j, k, new_id)
            else:
                self._unregister_object_block(i, j, k)
        
        if self._has_handlers(old_id) != self._has_handlers(new_id):
            self._objects_dirty = True
            if self._has_handlers(new_id):
                self._block_handler_locs.add((i, j, k))
                self._call_handlers(new_id, 'onLoad', i, j, k)
            else:
                self._block_handler_locs.remove((i, j, k))

        # Propagate dirtiness to neighbors if on boundary
        if solidity_changed or opacity_changed:
            self._propagate_boundary_dirtiness(i, j, k)

    def _propagate_boundary_dirtiness(self, i, j, k):
        edge = self.size - 1
        # Determine neighbor offsets: -1 if on boundary, 0 otherwise
        imin = -1 if i == 0 else 0
        jmin = -1 if j == 0 else 0
        kmin = -1 if k == 0 else 0
        imax = 1 if i == edge else 0
        jmax = 1 if j == edge else 0
        kmax = 1 if k == edge else 0

        for ni in range(imin, imax + 1):
            for nj in range(jmin, jmax + 1):
                for nk in range(kmin, kmax + 1):
                    if ni == 0 and nj == 0 and nk == 0:
                        continue
                    neighbor = self._neighbors[1 + ni, 1 + nj, 1 + nk]
                    if neighbor is not None:
                        neighbor._terrain_dirty = True
                        queue_chunk_for_remesh(neighbor)

    def update_meshes(self):
        if self._terrain_dirty:
            regenerate_terrain_mesh(self)
            self._terrain_dirty = False
        if self._objects_dirty:
            update_object_meshes(self)
            self._objects_dirty = False

    def dispose(self):
        for loc in self._block_handler_locs:
            self._call_handlers(self.voxels[loc], 'onUnload', *loc)
        self._neighbors = None
        self.voxels = None
```

**Edge Cases and Pitfalls**
*   **Cascading Updates**: Dirtiness propagation must strictly limit updates to immediate neighbors on the chunk boundary. Updating internal neighbors or propagating beyond one hop causes performance degradation [S1, unofficial].
*   **Boundary Checks**: When $i=0$ or $i=N-1$, the neighbor array index must be correctly mapped to the adjacent chunk. Failure to check boundaries leads to null references or incorrect chunk updates [S1, unofficial].
*   **Constant Layers**: Caching whole-layer voxel IDs (`_whole_layer_voxel`) optimizes full-chunk scans but must be invalidated when any voxel in that layer changes [S1, unofficial].

## Operating characteristics
*   **Time Complexity**:
    *   Initialization: $O(N^3)$ to scan all voxels.
    *   Single Voxel Update: $O(1)$ for local update, $O(1)$ for neighbor propagation (max 26 neighbors, but only boundary neighbors are updated).
    *   Mesh Regeneration: Depends on the meshing algorithm, but triggered only when dirty flags are set.
*   **Memory Complexity**: $O(N^3)$ for voxel storage plus $O(1)$ for neighbor references (fixed 3x3x3 array).
*   **Strengths**: Efficiently isolates updates to chunk boundaries, preventing global scene recomputation. Simple state management with dirty flags.
*   **Limitations**: Fixed chunk size limits flexibility. Boundary propagation logic is complex to implement correctly. Not suitable for highly dynamic scenes with frequent voxel changes across many chunks without careful batching [S1, unofficial].
*   **When to Use**: Ideal for terrain or static object voxelization where changes are localized and mesh regeneration is expensive.
*   **When Not to Use**: Unsuitable for real-time, high-frequency voxel updates across the entire grid where a more global sparse structure (like SVO) might be more efficient [S1, unofficial].

## Background
Voxel Chunk Management is a component of the `noa` (Node Object Architecture) library, developed by fenomas [S1, unofficial]. It is part of a broader ecosystem for voxel-based game development and terrain generation, where efficient chunk-based storage and meshing are critical for performance [S1, unofficial]. The method emphasizes boundary-aware updates to maintain consistency across chunk boundaries without global synchronization [S1, unofficial].

## Sources
[S1] fenomas/noa, https://github.com/fenomas/noa, fetched 2026-10-26 (unofficial)
