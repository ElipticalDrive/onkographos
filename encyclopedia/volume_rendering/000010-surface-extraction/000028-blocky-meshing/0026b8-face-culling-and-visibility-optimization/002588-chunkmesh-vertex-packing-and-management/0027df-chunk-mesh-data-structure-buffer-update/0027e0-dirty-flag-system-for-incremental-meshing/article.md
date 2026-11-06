# Dirty Flag System for Incremental Meshing

Also called: Chunked Incremental Re-meshing, Dirty Chunk System, Lazy Rebuild

## Overview
The Dirty Flag System optimizes surface extraction in voxel terrain by tracking modified chunks to avoid remeshing unchanged geometry, enabling per-chunk incremental updates [S1, S2, unofficial]. It differs from full-scene re-meshing by only processing chunks containing modified density values, and from parallel approaches by serializing the rebuild phase per dirty chunk to ensure buffer consistency [S2, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A voxel world grid of size $M_x \times M_y \times M_z$, divided into chunks of size $C \times C \times C$. A set of modified voxel coordinates or a bounding box of modifications.
- Output: Updated GPU vertex buffers for only the affected chunks.

**Data Structures:**
- `Chunk`: Contains a boolean flag `dirty` indicating whether the chunk's geometry has changed since the last mesh pass [S1, unofficial].
- `dirtyChunks`: A set or list storing indices $(c_x, c_y, c_z)$ of chunks that require rebuilding [S2, unofficial].
- `TempStorage`: Temporary vertex storage allocated per chunk for meshing [S1, unofficial].

**Steps:**
1. **Marking Dirty Chunks:** When density is modified in a voxel range $[minX, maxX] \times [minY, maxY] \times [minZ, maxZ]$, calculate the affected chunk indices. Because a voxel at index $x$ affects the cell to its left ($x-1$), the minimum chunk index is calculated using integer division:
   $$ c_{minX} = \lfloor (minX - 1) / C \rfloor $$
   Similarly for $Y$ and $Z$. The maximum chunk index is:
   $$ c_{maxX} = \lfloor maxX / C \rfloor $$
   Iterate through all chunk indices from $(c_{minX}, c_{minY}, c_{minZ})$ to $(c_{maxX}, c_{maxY}, c_{maxZ})$ and add them to the `dirtyChunks` set [S2, unofficial].

2. **Meshing Cycle (LateUpdate):** Iterate through `dirtyChunks`. For each chunk:
   a. Call `PreMeshing()`: Assert the chunk is dirty and call `UnsetDirty()` to clear the flag [S1, unofficial].
   b. Call `Rebuild()`: Perform Marching Cubes on the chunk's voxels. Store resulting vertices in `TempStorage` [S2, unofficial].
   c. Call `PostMeshing()`: Check `TempStorage.Length`. If $> 0$, create a new `VoxelVertexBuffer` and copy data via `BufferData()`. Recycle temporary mesh visitor and vertex storage via `MeshVisitorAllocator.Recycle` and `TempStorageAllocator<VoxelVertex>.Recycle` [S1, unofficial].
   d. Remove the chunk from `dirtyChunks` [S2, unofficial].

**Parameters:**
- `chunkSize` ($C$): Typically 32, as chunks are always allocated in $32 \times 32 \times 32$ blocks [S1, unofficial].
- `dirty`: Boolean flag in `Chunk` struct [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Boundary Effects:** The chunk index calculation must account for the fact that a voxel at index $x$ affects the cell to its left ($x-1$). Failing to subtract 1 before division causes incorrect chunk marking for voxels at the start of a chunk [S2, unofficial].
- **Empty Chunks:** If `TempStorage.Length` is 0 after meshing, no buffer update is needed, but the chunk must still be removed from the dirty list [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mark_dirty(minX, maxX, minY, maxY, minZ, maxZ, chunkSize, chunks, dirty_chunks):
    # Calculate affected chunk range
    # Pitfall: voxel x affects cell x-1, so subtract 1 before division
    c_min_x = (minX - 1) // chunkSize
    c_min_y = (minY - 1) // chunkSize
    c_min_z = (minZ - 1) // chunkSize
    
    c_max_x = maxX // chunkSize
    c_max_y = maxY // chunkSize
    c_max_z = maxZ // chunkSize
    
    # Add all chunks in range to dirty set
    for cx in range(c_min_x, c_max_x + 1):
        for cy in range(c_min_y, c_max_y + 1):
            for cz in range(c_min_z, c_max_z + 1):
                dirty_chunks.add((cx, cy, cz))
                chunks[cx, cy, cz].dirty = True

def process_dirty_chunks(dirty_chunks, chunks, chunk_size):
    for (cx, cy, cz) in list(dirty_chunks):
        chunk = chunks[cx, cy, cz]
        
        # Pre-meshing: clear flag
        chunk.pre_meshing()
        chunk.dirty = False
        
        # Rebuild mesh
        temp_storage = chunk.rebuild()
        
        # Post-meshing: update buffer if needed
        if len(temp_storage) > 0:
            new_buffer = create_voxel_vertex_buffer(temp_storage)
            new_buffer.buffer_data()
            # Recycle allocators
            mesh_visitor_allocator.recycle(chunk.mesh_visitor)
            temp_storage_allocator.recycle(temp_storage)
        
        dirty_chunks.remove((cx, cy, cz))
```

## Operating characteristics
- **Time Complexity:** $O(K)$ where $K$ is the number of dirty chunks, rather than $O(N)$ for total chunks.
- **Memory Complexity:** $O(1)$ additional memory per dirty chunk for temporary storage, reused via allocators.
- **Strengths:** Significant performance gain in dynamic scenes with sparse modifications.
- **Limitations:** Overhead of managing dirty sets; not suitable for static scenes or full-scene updates.
- **When to Use:** Dynamic voxel terrain with frequent small changes.
- **When Not to Use:** Static scenes or when all chunks change simultaneously.

## Background
Originating from implementations in voxel terrain engines, this system is used in projects like `vercidium-patreon/meshing` and `Rabidus4k/UnityVoxelTerrain` to optimize Marching Cubes surface extraction [S1, S2, unofficial].

## Sources
[S1] vercidium-patreon/meshing, https://github.com/vercidium-patreon/meshing, fetched 2026-10-26 (unofficial)
[S2] Rabidus4k/UnityVoxelTerrain, https://github.com/Rabidus4k/UnityVoxelTerrain, fetched 2026-10-28 (unofficial)
