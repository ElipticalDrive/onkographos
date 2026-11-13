# Voxel Chunk Management

Also called: Voxel Grid Block

## Overview
Voxel Chunk Management organizes sparse voxel data into fixed-size cubic blocks (chunks) to enable efficient memory allocation, neighbor traversal, and incremental mesh generation. It solves the problem of managing large, sparse 3D grids by isolating updates to local regions, ensuring that changes to voxel properties trigger only necessary remeshing of affected chunk boundaries. Unlike dense voxel grids, which allocate memory for the entire volume, this method uses a sparse structure where chunks are allocated on demand, and it differs from SVO-based approaches by maintaining a flat, fixed-resolution block structure rather than a hierarchical tree [S1, unofficial].

## How it works
### Inputs and Outputs
*   **Input**: A world coordinate system, voxel data updates (set operations), and configuration parameters (chunk size, default voxel ID).
*   **Output**: Updated voxel arrays, neighbor references, dirty flags, and queued mesh generation requests.

### Data Structures
The core data structure is the **Chunk**, which contains the following fields:

1.  **`voxels`**: A 3D array of type `Uint16Array` with shape `[size, size, size]`. It stores the voxel ID for each position within the chunk. The default fill value is `fillVoxelID` (typically `-1` to indicate empty/uninitialized).
2.  **`_neighbors`**: A 3D array of shape `[3, 3, 3]` containing references to adjacent chunks. The center element `[1, 1, 1]` refers to the chunk itself. Indices `[0,0,0]` to `[2,2,2]` correspond to relative offsets from `-1` to `+1` in each dimension. This allows O(1) access to the 26 neighbors.
3.  **`_wholeLayerVoxel`**: An array of length `size` (indexed by `j`). It caches the voxel ID for an entire horizontal layer if all voxels in that layer share the same ID. If the layer is not uniform, the value is `-1`. This optimization speeds up opacity and solidity checks.
4.  **`_terrainDirty`**: A boolean flag indicating that the chunk's geometry needs regeneration.
5.  **`_objectsDirty`**: A boolean flag indicating that object-specific mesh data needs update.
6.  **`_blockHandlerLocs`**: A collection (e.g., set or list) storing coordinates `(i, j, k)` of voxels that require special handler execution (e.g., physics or logic updates).

### Parameters
*   **`size`**: The dimension $N$ of the cubic chunk ($N \times N \times N$). Typical values are powers of two (e.g., 16, 32) for efficient memory alignment, though not strictly required.
*   **`fillVoxelID`**: The default ID for empty space. Default is `-1`.

### Algorithm Steps

#### 1. Initialization
1.  Allocate `voxels` as a `Uint16Array` of size $N^3$.
2.  Fill `voxels` with `fillVoxelID`.
3.  Initialize `_neighbors` as a `[3, 3, 3]` array. Set `_neighbors[1][1][1]` to `this`. Initialize other entries to `null`.
4.  Initialize `_wholeLayerVoxel` as an array of length $N$ with value `-1`.
5.  Set `_terrainDirty` and `_objectsDirty` to `false`.

#### 2. Voxel Update (`set(i, j, k, newID)`)
When a voxel at local coordinates $(i, j, k)$ is updated to `newID`:
1.  Update `voxels[i][j][k] = newID`.
2.  Determine if the change affects solidity or opacity. This requires looking up properties of `newID` and the old ID in a global registry (`solidityLookup`, `opacityLookup`).
3.  Set `_terrainDirty = true` and queue the chunk for remeshing.
4.  **Neighbor Propagation**: If the voxel is on a chunk boundary (i.e., $i=0, i=N-1, j=0, j=N-1, k=0, k=N-1$) AND the change affects solidity or opacity, propagate dirtiness to adjacent chunks.
    *   Define boundary offsets:
        $$
        \begin{aligned}
        imin &= (i == 0) ? -1 : 0 \\
        imax &= (i == N-1) ? 1 : 0 \\
        jmin &= (j == 0) ? -1 : 0 \\
        jmax &= (j == N-1) ? 1 : 0 \\
        kmin &= (k == 0) ? -1 : 0 \\
        kmax &= (k == N-1) ? 1 : 0
        \end{aligned}
        $$
    *   Iterate $ni$ from $imin$ to $imax$, $nj$ from $jmin$ to $jmax$, $nk$ from $kmin$ to $kmax$.
    *   Skip if $ni=nj=nk=0$ (self).
    *   If `_neighbors[ni+1][nj+1][nk+1]` exists:
        *   Set `neighbor._terrainDirty = true`.
        *   Queue `neighbor` for remeshing.

#### 3. Mesh Generation (`updateMeshes`)
Triggered when `_terrainDirty` or `_objectsDirty` is true.
1.  If `_terrainDirty`:
    *   Scan `voxels` to determine visibility and solidity.
    *   Use `_wholeLayerVoxel` to optimize checks for uniform layers.
    *   Generate terrain geometry (e.g., marching cubes, face culling) based on voxel IDs.
    *   Clear `_terrainDirty`.
2.  If `_objectsDirty`:
    *   Identify object blocks using `objectLookup`.
    *   Generate object-specific geometry.
    *   Clear `_objectsDirty`.

#### 4. Disposal
1.  Call `onUnload` handlers for all blocks in `_blockHandlerLocs`.
2.  Nullify `_neighbors` references to allow garbage collection.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class VoxelChunk:
    def __init__(self, size, fillVoxelID=-1):
        self.size = size
        self.fillVoxelID = fillVoxelID
        
        # Data structures
        # voxels: 1D Uint16Array of size size*size*size, or 3D array [size][size][size]
        self.voxels = [fillVoxelID] * (size * size * size)
        
        # _neighbors: 3x3x3 array of Chunk references
        self._neighbors = [[[None for _ in range(3)] for _ in range(3)] for _ in range(3)]
        self._neighbors[1][1][1] = self  # Center is self
        
        # _wholeLayerVoxel: Array of length size, initialized to -1
        self._wholeLayerVoxel = [-1] * size
        
        # Dirty flags
        self._terrainDirty = False
        self._objectsDirty = False
        
        # Handlers
        self._blockHandlerLocs = set()

    def set_voxel(self, i, j, k, newID, registry):
        """
        Update a voxel and handle neighbor propagation.
        registry: dict mapping ID to properties {solidity, opacity, is_object, handlers}
        """
        oldID = self.voxels[i * self.size * self.size + j * self.size + k]
        self.voxels[i * self.size * self.size + j * self.size + k] = newID
        
        # Check if properties changed
        old_props = registry.get(oldID, {})
        new_props = registry.get(newID, {})
        
        solidity_changed = old_props.get('solidity') != new_props.get('solidity')
        opacity_changed = old_props.get('opacity') != new_props.get('opacity')
        
        # Always dirty for terrain if voxel changes
        self._terrainDirty = True
        queue_chunk_for_remesh(self)
        
        # Propagate to neighbors if on boundary and properties changed
        if solidity_changed or opacity_changed:
            edge = self.size - 1
            imin = -1 if i == 0 else 0
            imax = 1 if i == edge else 0
            jmin = -1 if j == 0 else 0
            jmax = 1 if j == edge else 0
            kmin = -1 if k == 0 else 0
            kmax = 1 if k == edge else 0
            
            for ni in range(imin, imax + 1):
                for nj in range(jmin, jmax + 1):
                    for nk in range(kmin, kmax + 1):
                        if ni == 0 and nj == 0 and nk == 0:
                            continue
                        
                        neighbor = self._neighbors[ni + 1][nj + 1][nk + 1]
                        if neighbor is not None:
                            neighbor._terrainDirty = True
                            queue_chunk_for_remesh(neighbor)

    def update_meshes(self, terrain_mesher, object_mesher, registry):
        """
        Regenerate geometry if dirty.
        """
        if self._terrainDirty:
            self._scan_and_generate_terrain(terrain_mesher, registry)
            self._terrainDirty = False
            
        if self._objectsDirty:
            self._generate_objects(object_mesher, registry)
            self._objectsDirty = False

    def _scan_and_generate_terrain(self, mesher, registry):
        """
        Scan voxels to determine terrain geometry.
        Uses _wholeLayerVoxel for optimization.
        """
        fullyOpaque = True
        fullyAir = True
        
        # Optimization: Check layers
        for j in range(self.size):
            layerID = self._wholeLayerVoxel[j]
            
            # If layer is cached and uniform
            if layerID >= 0:
                props = registry.get(layerID, {})
                if not props.get('opaque', False):
                    fullyOpaque = False
                if layerID != 0: # Assuming 0 is air
                    fullyAir = False
                continue
            
            # Check if layer is uniform
            constantID = self.voxels[0 * self.size * self.size + j * self.size + 0]
            is_uniform = True
            
            for i in range(self.size):
                for k in range(self.size):
                    idx = i * self.size * self.size + j * self.size + k
                    id = self.voxels[idx]
                    
                    if constantID >= 0 and id != constantID:
                        is_uniform = False
                        constantID = -1
                        break
                    
                    if id == 0: # Air
                        fullyOpaque = False
                        continue
                    
                    props = registry.get(id, {})
                    if not props.get('opaque', False):
                        fullyOpaque = False
                    
                    if id != 0:
                        fullyAir = False
                    
                    if props.get('is_object', False):
                        self._objectsDirty = True
                        object_mesher.set_object_block(self, id, i, j, k)
                    
                    if props.get('handlers'):
                        self._blockHandlerLocs.add((i, j, k))
                
                if not is_uniform:
                    break
            
            if is_uniform and constantID >= 0:
                self._wholeLayerVoxel[j] = constantID

        # Call mesher with chunk data
        mesher.generate(self)

    def dispose(self):
        """
        Clean up resources.
        """
        for (i, j, k) in self._blockHandlerLocs:
            call_block_handler(self, 'onUnload', i, j, k)
        
        # Nullify neighbors
        for x in range(3):
            for y in range(3):
                for z in range(3):
                    self._neighbors[x][y][z] = None

def queue_chunk_for_remesh(chunk):
    """
    Add chunk to a global queue for processing in the next frame or batch.
    """
    pass # Implementation detail of the engine
```

### Edge Cases and Pitfalls
*   **Cascading Updates**: Neighbor dirtiness propagation must be limited to immediate neighbors on the chunk boundary. Propagating to all neighbors or recursively can cause performance degradation. The check `i==0`, `i==edge`, etc., ensures only boundary voxels trigger neighbor updates [S1, unofficial].
*   **Precision**: Voxel IDs are `Uint16`, limiting the number of distinct voxel types to 65,535. Ensure the registry fits within this range.
*   **Boundary Conditions**: When accessing `_neighbors`, always check for `None` to avoid null pointer exceptions at the world boundary.

## Operating characteristics
*   **Time Complexity**:
    *   `set_voxel`: $O(1)$ for the update, plus $O(1)$ for neighbor propagation (max 26 neighbors, but typically only 1-6 are affected).
    *   `update_meshes`: $O(N^3)$ for scanning the chunk, where $N$ is the chunk size. This is done only when dirty.
*   **Memory Complexity**: $O(N^3)$ per chunk for the voxel array. The neighbor array is $O(1)$ (constant 27 references).
*   **Scaling**: Memory usage scales linearly with the number of active chunks. For sparse worlds, this is significantly more efficient than dense grids.
*   **Strengths**:
    *   Efficient incremental updates.
    *   Simple neighbor traversal.
    *   Good cache locality within a chunk.
*   **Limitations**:
    *   Fixed resolution within a chunk.
    *   Boundary artifacts if not handled correctly (e.g., lighting, normals).
    *   Not suitable for highly dynamic, high-frequency changes across many chunks without a robust batching system.

## Background
Voxel Chunk Management is a common technique in voxel-based games and simulations, particularly in engines like Noa (Node.js Open Architecture) [S1, unofficial]. It builds on the principles of sparse voxel storage by dividing the world into manageable blocks. This approach allows for efficient culling and localized updates, which is critical for real-time applications. The method is often used in conjunction with terrain generation algorithms and physics engines.

## Sources
[S1] fenomas/noa, https://github.com/fenomas/noa, fetched 2026-10-26 (unofficial)
