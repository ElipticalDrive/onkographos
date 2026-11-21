# Chunk Data Modification & Queue Management


## Overview
This method handles the mutation of voxel data within a chunk-based voxel world and manages the subsequent regeneration of geometry for affected areas. It solves the problem of maintaining consistency between raw voxel data and the rendered mesh when user edits or game logic modify the world state. It differs from [Sparse Voxel Octree Representation](2340-sparse-voxel-octree-representation.md) by operating on fixed-size dense grids rather than hierarchical trees, and from [Compact Voxel Section Storage](4219-compact-voxel-section-storage.md) by explicitly managing a modification queue for mesh updates rather than just storage compression.

## How it works
**Inputs:** A mutable reference to the chunk data structure, a 3D position (`pos`) in world space, and a new block type/value.
**Outputs:** Updated chunk data, updated adjacency sets, and queued chunks for mesh regeneration.

**Data Structures:**
*   `ChunkData`: Contains a flat array of voxel IDs. If sparse, it may start with a single block ID.
*   `adj_chunk_set`: A set of adjacent chunk indices that require mesh updates due to boundary modifications.
*   `mesh_queue`: A queue of chunk indices requiring mesh loading/regeneration.
*   `CHUNK_SIZE_I32`: Integer constant, typically 32.

**Steps:**
1.  **Retrieve Chunk Data:** Calculate the chunk index for the given world position. If the chunk data is `None` (not loaded), skip processing [S1, unofficial].
2.  **Expand Sparse Array:** If the voxel array length is 1, expand it to the full chunk size ($CHUNK\_SIZE\_I32^3$) by duplicating the single block data into all positions [S1, unofficial].
3.  **Calculate Local Index:** Compute the local 3D coordinates $(x, y, z)$ within the chunk and the flat index $i$:
    $$ i = \text{vec3\_to\_index}(local\_pos, CHUNK\_SIZE\_I32) $$
    Where `vec3_to_index` maps $(x, y, z)$ to $x + y \cdot W + z \cdot W \cdot H$ with $W=H=CHUNK\_SIZE\_I32$ [S1, unofficial].
4.  **Update Voxel:** Set the voxel block type at index $i$ to the new value [S1, unofficial].
5.  **Check Boundaries:** If the modification occurs at a chunk edge (i.e., $x=0, x=31, y=0, y=31, z=0, z=31$), add the adjacent chunk indices to `adj_chunk_set` [S1, unofficial].
6.  **Queue for Meshing:** Add the current chunk and all chunks in `adj_chunk_set` to the `mesh_queue` for mesh loading/regeneration [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def modify_chunk(chunk_data, pos, new_block_type, adj_chunk_set, mesh_queue):
    # Step 1: Retrieve chunk data
    # Assuming chunk_index calculation is handled by caller or helper
    if chunk_data is None:
        return

    # Step 2: Expand if sparse
    if len(chunk_data.voxel_array) == 1:
        single_block = chunk_data.voxel_array[0]
        chunk_data.voxel_array = [single_block] * (CHUNK_SIZE_I32 ** 3)

    # Step 3: Calculate local index
    # pos is world position; need local position relative to chunk origin
    local_pos = get_local_position(pos, chunk_data.origin)
    i = vec3_to_index(local_pos, CHUNK_SIZE_I32)

    # Step 4: Update voxel
    chunk_data.voxel_array[i] = new_block_type

    # Step 5: Check boundaries
    x, y, z = local_pos
    is_edge = (x == 0 or x == CHUNK_SIZE_I32 - 1 or
               y == 0 or y == CHUNK_SIZE_I32 - 1 or
               z == 0 or z == CHUNK_SIZE_I32 - 1)

    if is_edge:
        # Add adjacent chunks to set
        # Implementation detail: calculate neighbor chunk indices based on x,y,z
        # and add to adj_chunk_set
        add_adjacent_chunks(pos, adj_chunk_set)

    # Step 6: Queue for meshing
    mesh_queue.push(chunk_data.chunk_index)
    for adj_idx in adj_chunk_set:
        mesh_queue.push(adj_idx)
```

## Subtopics

### Chunk-based Terrain Modification via List & Filtering
Stores user terrain edits as uncompressed lists of larger chunks (e.g., $16^3$ voxels) rather than individual voxels. It renders these changes with bilinear filtering to smooth the subtraction. This approach reduces storage overhead for large edits but produces a "curved stair-step" effect across voxels due to the filtering [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ for single voxel modification and index calculation. Queue management is $O(1)$ amortized per operation.
*   **Memory Complexity:** $O(N)$ where $N$ is the number of modified chunks. Sparse expansion increases memory usage from $O(1)$ to $O(CHUNK\_SIZE^3)$ for previously empty chunks [S1, unofficial].
*   **Strengths:** Simple implementation, efficient for dense worlds, explicit control over mesh regeneration scope.
*   **Limitations:** Sparse expansion can be memory-intensive if many chunks are initially empty. Bilinear filtering variants introduce visual artifacts [S2, unofficial].
*   **When to use:** Real-time voxel editing in games where performance is critical and visual fidelity can tolerate stair-stepping or simple smoothing.

## Background
Originated in voxel-based game engines such as VoxelQuest and demonstrated in the Binary Greedy Voxel Meshing Algorithm demo. It addresses the need for efficient, real-time terrain modification in infinite voxel worlds without the overhead of octree traversal for every edit [S1, unofficial][S2, unofficial].

## Sources
[S1] Binary Greedy Voxel Meshing Algorithm, https://github.com/TanTanDev/binary_greedy_mesher_demo, fetched 2026-10-26 (unofficial)
[S2] How Does It Work?, https://www.voxelquest.com/news/how-does-it-work, fetched 2026-10-29 (unofficial)
