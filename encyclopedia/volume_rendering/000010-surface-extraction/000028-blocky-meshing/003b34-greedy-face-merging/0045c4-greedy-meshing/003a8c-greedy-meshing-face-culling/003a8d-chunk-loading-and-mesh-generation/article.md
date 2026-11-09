# Chunk Loading and Mesh Generation


## Overview
This method manages the lifecycle of voxel chunk geometry by loading chunk data from a world, generating its mesh via a mesher, and caching the resulting GPU resources. It solves the problem of efficiently updating visible geometry when chunks are first accessed or when their content changes, ensuring that only chunks with non-empty geometry are rendered. It builds on [Greedy Meshing / Face Culling](8783-greedy-meshing-face-culling.md) for the actual triangle generation.

## How it works
The method operates on a per-chunk basis, handling both initial loading and subsequent updates. It relies on a 3D world coordinate system where chunks are indexed by integer coordinates.

**Inputs and Outputs:**
*   **Input:** Chunk coordinates $(cx, cy, cz)$, world data structure (block map), atlas properties (texture atlas layout), and world material.
*   **Output:** A `Model` object containing a `GPUMesh`, stored in a `chunk_models` map keyed by chunk coordinates.

**Data Structures:**
*   **Chunk Key:** A 64-bit integer key representing chunk coordinates. The coordinates are extracted using bit shifts. Let $K$ be the key.
    *   $cx = (K \gg 32) \& 0xFFFFFFFF$ (assuming 32-bit chunks, exact bit layout depends on implementation [S1, unofficial]).
    *   $cy, cz$ extracted similarly.
*   **ChunkMesher:** An object instantiated with a $3 \times 3 \times 3$ neighborhood of chunks and atlas properties. It generates raw mesh data (vertices, indices).
*   **GPUMesh:** A GPU buffer containing vertex and index data.
*   **Model:** A high-level object wrapping the `GPUMesh` and material.
*   **chunk_models:** A map from chunk key to `Model` (or a struct containing mesh, model, transform, and bounds).

**Numbered Steps for Initial Load:**
1.  **Extract Coordinates:** Compute chunk coordinates $(cx, cy, cz)$ from the 64-bit chunk key using bit shifts [S1, unofficial].
2.  **Build Neighborhood:** Construct a $3 \times 3 \times 3$ grid of chunk keys centered at $(cx, cy, cz)$. This includes the target chunk and its 26 neighbors.
3.  **Instantiate Mesher:** Create a `ChunkMesher` instance, passing the neighborhood data and atlas properties.
4.  **Generate Mesh:** Call the mesher to generate mesh data (vertices and indices) for the target chunk.
5.  **Check Empty:** If the generated mesh has zero vertices, remove the chunk from the render map (if present) and return [S1, unofficial].
6.  **Create GPU Resource:** Create a `GPUMesh` from the generated vertex and index data.
7.  **Create Model:** Instantiate a `Model` using the `GPUMesh` and the world material.
8.  **Compute Transform:** Calculate the world transform matrix. This is a translation by $(cx \cdot S, cy \cdot S, cz \cdot S)$, where $S$ is the chunk size in world units [S1, unofficial].
9.  **Store:** Insert the mesh, model, transform, and bounding box into the `chunk_models` map, keyed by the chunk key [S1, unofficial].

**Pseudocode for Initial Load:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def load_chunk(chunk_key, world_data, atlas_props, material):
    # 1. Extract coordinates from 64-bit key
    # Implementation detail: bit shifts depend on specific key layout.
    # Assuming cx, cy, cz are packed into the 64-bit integer.
    cx, cy, cz = extract_chunk_coords(chunk_key)
    
    # 2. Build 3x3x3 neighborhood
    neighborhood = []
    for dx in range(-1, 2):
        for dy in range(-1, 2):
            for dz in range(-1, 2):
                neighbor_key = get_chunk_key(cx + dx, cy + dy, cz + dz)
                neighborhood.append(neighbor_key)
    
    # 3. Instantiate Mesher
    mesher = ChunkMesher(neighborhood, atlas_props)
    
    # 4. Generate mesh data
    mesh_data = mesher.generate()
    
    # 5. Check for empty mesh
    if mesh_data.vertex_count == 0:
        if chunk_key in chunk_models:
            del chunk_models[chunk_key]
        return None
    
    # 6. Create GPUMesh
    gpu_mesh = create_gpu_mesh(mesh_data.vertices, mesh_data.indices)
    
    # 7. Create Model
    model = create_model(gpu_mesh, material)
    
    # 8. Compute world transform
    chunk_size = 16 # Typical value, depends on world definition
    transform = translate(Vec3(cx * chunk_size, cy * chunk_size, cz * chunk_size))
    
    # 9. Store in map
    chunk_models[chunk_key] = {
        'mesh': gpu_mesh,
        'model': model,
        'transform': transform,
        'bounds': compute_bounds(transform, mesh_data)
    }
    
    return model
```

**Edge Cases and Pitfalls:**
*   **Empty Chunks:** Chunks with no solid blocks must be explicitly removed from the render map to avoid rendering empty geometry or wasting memory [S1, unofficial].
*   **Boundary Conditions:** The $3 \times 3 \times 3$ neighborhood must handle chunks that do not exist (e.g., outside the loaded world area) by treating them as empty or air blocks.
*   **Precision:** Chunk coordinates are integers, but world transforms use floats. Ensure consistent precision when computing bounds and transforms.

## Subtopics

### Remeshing Affected Chunks
This variant handles updates when a single block changes. It identifies all chunks that need regeneration because the modified block is either inside them or adjacent to them (affecting face culling).
*   **Steps:**
    1.  Compute the chunk coordinates $(cx, cy, cz)$ containing the modified block [S1, unofficial].
    2.  Compute local coordinates $(lx, ly, lz)$ within the chunk using bitwise AND with $(S-1)$, where $S$ is chunk size [S1, unofficial].
    3.  Determine the range of chunk offsets to check. For each axis, the offset is $-1$ if the local coordinate is $0$, $0$ if $0 < local < S-1$, and $1$ if $local == S-1$. This ensures the modified block and its neighbors are covered [S1, unofficial].
    4.  Loop over the $3 \times 3 \times 3$ offsets relative to $(cx, cy, cz)$ to generate unique chunk keys [S1, unofficial].
    5.  For each unique chunk key, call `load_chunk` (or a specialized update function) to reload and regenerate the mesh [S1, unofficial].

**Pseudocode for Remeshing:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def remesh_affected_chunks(modified_block_pos, world_data, atlas_props, material):
    # modified_block_pos is (bx, by, bz) in world coordinates
    chunk_size = 16
    cx = modified_block_pos.x // chunk_size
    cy = modified_block_pos.y // chunk_size
    cz = modified_block_pos.z // chunk_size
    
    # Local coordinates
    lx = modified_block_pos.x % chunk_size
    ly = modified_block_pos.y % chunk_size
    lz = modified_block_pos.z % chunk_size
    
    # Determine offsets
    offsets = []
    for local, coord in [(lx, 'x'), (ly, 'y'), (lz, 'z')]:
        if local == 0:
            offsets.append(-1)
        elif local == chunk_size - 1:
            offsets.append(1)
        else:
            offsets.append(0)
            
    # Generate unique chunk keys
    affected_keys = set()
    for dx in offsets:
        for dy in offsets:
            for dz in offsets:
                key = get_chunk_key(cx + dx, cy + dy, cz + dz)
                affected_keys.add(key)
                
    # Reload each affected chunk
    for key in affected_keys:
        load_chunk(key, world_data, atlas_props, material)
```

## Operating characteristics
*   **Time Complexity:** Initial load is $O(N)$ where $N$ is the number of blocks in the chunk and its neighborhood, due to mesh generation. Remeshing is $O(1)$ relative to the world size, but $O(M)$ where $M$ is the number of blocks in the affected chunks (typically up to 9 chunks).
*   **Memory Complexity:** Proportional to the number of loaded chunks. Each chunk stores a `GPUMesh` and `Model`.
*   **Strengths:** Efficiently handles dynamic worlds by only regenerating necessary chunks. Caching avoids redundant mesh generation.
*   **Limitations:** Requires careful management of the `chunk_models` map. Loading a large number of chunks simultaneously can cause frame drops.
*   **When to Use:** When rendering dynamic voxel worlds where blocks are frequently modified.
*   **When Not to Use:** For static worlds where a single mesh for the entire world is feasible.

## Background
This method is derived from the implementation in the HelloVoxel project [S1, unofficial]. It represents a standard approach in voxel game engines for managing chunk-based geometry.

## Sources
[S1] Linirby/HelloVoxel, https://github.com/Linirby/HelloVoxel, fetched 2026-10-27 (unofficial)
