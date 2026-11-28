# Voxel Chunk Management

Also called: Voxel Grid Block

## Overview
Voxel Chunk Management handles the storage, state tracking, and dirty-flag propagation for a fixed-size 3D block of voxel data within a sparse voxel architecture [S1, unofficial]. It solves the problem of efficiently determining when a chunk's geometric representation or logical state requires regeneration after voxel modifications, specifically by isolating boundary updates to prevent cascading recomputations across the voxel grid [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** Chunk dimensions $N \times N \times N$, a default fill voxel ID, and a voxel data array [S1, unofficial].
*   **Output:** Updated chunk state flags (`_terrainDirty`, `_objectsDirty`), neighbor dirtiness propagation, and queued mesh regeneration requests [S1, unofficial].

**Data Structures**
The chunk maintains the following internal state [S1, unofficial]:
1.  `voxels`: A 3D array of shape `[size, size, size]` storing voxel IDs as `Uint16` [S1, unofficial].
2.  `_neighbors`: A 3x3x3 array of neighbor chunk references with lower bounds `[1,1,1]`, where the center element `[1,1,1]` refers to the chunk itself [S1, unofficial].
3.  `_wholeLayerVoxel`: An array of length `size` caching the voxel ID for entire layers or `-1` [S1, unofficial].
4.  `_terrainDirty`: Boolean flag indicating if the terrain mesh needs regeneration [S1, unofficial].
5.  `_objectsDirty`: Boolean flag indicating if object-specific meshes or handlers need updates [S1, unofficial].

**Parameters**
*   `size`: The dimension $N$ of the cubic chunk [S1, unofficial].
*   `fillVoxelID`: The default ID to initialize the voxel array with (commonly `-1`) [S1, unofficial].

**Algorithm Steps**
1.  Allocate `voxels` array as `Uint16Array` wrapped in a 3D structure of shape `[size, size, size]` [S1, unofficial].
2.  Initialize `_neighbors` array as a 27-element structure where the center reference points to itself [S1, unofficial].
3.  Scan voxel data to set `fullyOpaque` and `fullyAir` flags, register object blocks, and queue block handlers [S1, unofficial].
4.  On `set(i,j,k,newID)`: update the array, check registry lookups for solidity/opacity/object/handler, update dirty flags, queue the chunk for remeshing, and propagate dirtiness to edge neighbors if solidity or opacity changed [S1, unofficial].
5.  On `updateMeshes`: call terrain or object meshers if dirty flags are set, then clear the flags [S1, unofficial].
6.  On `dispose`: call `onUnload` handlers and nullify references [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class VoxelChunk:
    def __init__(self, size, fill_voxel_id=-1):
        self.size = size
        # voxels: ndarray of Uint16Array with shape [size, size, size]
        self.voxels = [[[fill_voxel_id] * size for _ in range(size)] for _ in range(size)]
        # _neighbors: ndarray of shape [3,3,3] with lower bounds [1,1,1] (center=1,1,1 is self)
        self._neighbors = [[[None] * 3 for _ in range(3)] for _ in range(3)]
        self._neighbors[1][1][1] = self
        # _wholeLayerVoxel: Array(size) storing cached layer ID or -1
        self._whole_layer_voxel = [-1] * size
        # _terrainDirty/_objectsDirty: boolean flags
        self._terrain_dirty = False
        self._objects_dirty = False
        self._block_handler_locs = set()
        # Registry lookups (simplified for self-containment)
        self.plain_lookup = {0}
        self.opaque_lookup = {1, 2, 3}
        self.object_lookup = {4, 5}
        self.handler_lookup = {6: 'handler_a', 7: 'handler_b'}

    def scan_voxels(self, data):
        fully_opaque = True
        fully_air = True
        for j in range(self.size):
            layer_id = self._whole_layer_voxel[j]
            if layer_id >= 0 and layer_id not in self.object_lookup and layer_id not in self.handler_lookup:
                if layer_id not in self.opaque_lookup:
                    fully_opaque = False
                if layer_id != 0:
                    fully_air = False
                continue
            constant_id = self.voxels[0][j][0]
            for i in range(self.size):
                for k in range(self.size):
                    id_val = data[i][j][k]
                    if constant_id >= 0 and id_val != constant_id:
                        constant_id = -1
                    if id_val == 0:
                        fully_opaque = False
                        continue
                    if id_val in self.plain_lookup:
                        fully_air = False
                        continue
                    fully_opaque = fully_opaque and (id_val in self.opaque_lookup)
                    fully_air = False
                    if id_val in self.object_lookup:
                        self._objects_dirty = True
                    if id_val in self.handler_lookup:
                        self._block_handler_locs.add((i, j, k))
                if constant_id >= 0:
                    self._whole_layer_voxel[j] = constant_id
        return fully_opaque, fully_air

    def set_voxel(self, i, j, k, new_id):
        old_id = self.voxels[i][j][k]
        if old_id == new_id:
            return
        self.voxels[i][j][k] = new_id
        solidity_changed = (old_id in self.plain_lookup) != (new_id in self.plain_lookup)
        opacity_changed = (old_id in self.opaque_lookup) != (new_id in self.opaque_lookup)
        if solidity_changed or opacity_changed:
            self._terrain_dirty = True
        if old_id in self.object_lookup != new_id in self.object_lookup:
            self._objects_dirty = True
        if old_id in self.handler_lookup != new_id in self.handler_lookup:
            self._objects_dirty = True
            if new_id in self.handler_lookup:
                self._block_handler_locs.add((i, j, k))
            else:
                self._block_handler_locs.discard((i, j, k))
        if solidity_changed or opacity_changed:
            self._propagate_boundary_dirtiness(i, j, k)

    def _propagate_boundary_dirtiness(self, i, j, k):
        edge = self.size - 1
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
                    neighbor = self._neighbors[1 + ni][1 + nj][1 + nk]
                    if neighbor is not None:
                        neighbor._terrain_dirty = True
                        # queue_chunk_for_remesh(neighbor)

    def update_meshes(self):
        if self._terrain_dirty:
            # regenerate_terrain_mesh(self)
            self._terrain_dirty = False
        if self._objects_dirty:
            # update_object_meshes(self)
            self._objects_dirty = False

    def dispose(self):
        for loc in self._block_handler_locs:
            # call onUnload handlers
            pass
        self._neighbors = None
        self.voxels = None
```

**Edge Cases and Pitfalls**
*   **Cascading Updates**: Dirtiness propagation must strictly limit updates to immediate neighbors on the chunk boundary to avoid cascading updates [S1, unofficial].
*   **Boundary Checks**: When $i=0$ or $i=N-1$, the neighbor array index must be correctly mapped to the adjacent chunk using the offset logic (`imin`, `imax`, etc.) [S1, unofficial].
*   **Constant Layers**: Caching whole-layer voxel IDs (`_whole_layer_voxel`) optimizes full-chunk scans but must be invalidated when any voxel in that layer changes [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: Initialization requires $O(N^3)$ to scan all voxels [S1, unofficial]. Single voxel updates and neighbor propagation are bounded by constant checks, resulting in $O(1)$ complexity [S1, unofficial].
*   **Memory Complexity**: $O(N^3)$ for voxel storage plus $O(1)$ for neighbor references (fixed 3x3x3 array) [S1, unofficial].
*   **Strengths**: Efficiently isolates updates to chunk boundaries, preventing global scene recomputation [S1, unofficial]. Simple state management with dirty flags [S1, unofficial].
*   **Limitations**: Fixed chunk size limits flexibility [S1, unofficial]. Boundary propagation logic requires careful implementation to avoid incorrect updates [S1, unofficial].
*   **When to Use**: Ideal for terrain or static object voxelization where changes are localized and mesh regeneration is expensive [S1, unofficial].

## Background
Voxel Chunk Management is implemented in the `noa` library by fenomas [S1, unofficial]. It is used for voxel chunk management, terrain/object meshing, and collision handling within that system [S1, unofficial].

## Sources
[S1] fenomas/noa, https://github.com/fenomas/noa, fetched 2026-10-26 (unofficial)
