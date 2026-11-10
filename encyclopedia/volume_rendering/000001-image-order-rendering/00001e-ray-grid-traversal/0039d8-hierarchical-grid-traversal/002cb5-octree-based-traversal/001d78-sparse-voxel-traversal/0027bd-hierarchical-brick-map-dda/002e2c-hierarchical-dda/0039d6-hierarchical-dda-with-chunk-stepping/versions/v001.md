# Hierarchical DDA with Chunk Stepping

Also called: 3D Voxel Traversal, Chunked DDA Ray Marching, Voxel Ray Marching with DDA & Chunk Stepping

## Overview
Hierarchical DDA with Chunk Stepping is a ray traversal method that accelerates rendering in sparse voxel octrees by stepping through large empty regions (chunks) using a 3D Digital Differential Analyzer (DDA) algorithm, rather than visiting every individual voxel [S1, unofficial]. It builds on [Hierarchical DDA](5710-hierarchical-dda.md) by treating the lowest level of the hierarchy as solid blocks of $8 \times 8 \times 8$ voxels, allowing the ray to skip over entire chunks in $O(1)$ time per chunk boundary [S1, unofficial]. This approach is distinct from standard voxel traversal because it decouples the coarse-grained spatial skipping (chunk stepping) from the fine-grained sampling (inside the chunk), optimizing for scenes with large empty volumes [S1, unofficial].

## How it works
The method traces a ray through a hierarchical voxel map. It maintains a DDA state to determine when the ray crosses a chunk boundary. Upon entering a chunk, it converts the world-space entry point to chunk-local coordinates, steps through the $8 \times 8 \times 8$ voxel grid, and handles refraction or termination.

### Inputs and Outputs
**Inputs:**
- Ray origin: $\mathbf{rayPos}$ (world space)
- Ray direction: $\mathbf{rayDir}$ (normalized)
- Inverse ray direction: $\mathbf{invRayDir} = 1.0 / \mathbf{rayDir}$
- Current chunk position: $\mathbf{pos}$ (integer voxel coordinates of the chunk's origin)
- Map bounds: defined by the extent of the loaded chunks

**Outputs:**
- Boolean: `true` if a hit was found within the chunk; `false` if the ray exited the map or chunk without hitting.
- Updated ray state: $\mathbf{rayPos}$, $\mathbf{hitNormal}$, $\mathbf{voxel}$ data, and refraction flags.

### Data Structures
The implementation uses specific packed structures for memory efficiency [S1, unofficial].

**`DNcompressedVoxel`** (64 bits total):
- `normal`: 32 bits. Packed as:
  - Material ID: 8 bits
  - Normal X ($n_x$): 8 bits
  - Normal Y ($n_y$): 8 bits
  - Normal Z ($n_z$): 8 bits
- `albedo`: 32 bits. Packed as:
  - Red ($r$): 8 bits
  - Green ($g$): 8 bits
  - Blue ($b$): 8 bits
  - Unused: 8 bits

**`DNchunk`** (Variable size, fixed voxel count):
- `pos`: `DNivec3` (3D integer vector) storing the chunk's world-space origin in voxels.
- `updated`: `bool` indicating if the chunk data is current.
- `numVoxels`: `uint32_t` count of active voxels.
- `numVoxelsGpu`: `uint32_t` count for GPU transfer.
- `voxels`: Array of $8 \times 8 \times 8 = 512$ `DNcompressedVoxel` elements.

### Parameters
- `CHUNK_SIZE`: Linear dimension of a chunk in voxels. Value: $8$ [S1, unofficial].
- `INV_CHUNK_SIZE`: Reciprocal of `CHUNK_SIZE` ($1/8 = 0.125$), used to scale local coordinates back to world space [S1, unofficial].
- `EPSILON`: Small floating-point offset to prevent self-intersection and clamp boundary coordinates. Exact value not specified in the sources [S1, unofficial].
- `ignoreFirst`: Boolean flag. Initialized to `true` for the first DDA step to skip the entry voxel, then set to `false` [S1, unofficial].

### Algorithm Steps

1.  **Initialize DDA State**:
    Before the main loop, initialize the DDA variables using the ray properties and the current chunk position $\mathbf{pos}$ [S1, unofficial]:
    - Compute $\mathbf{deltaDist} = \mathbf{invRayDir} \times \text{CHUNK\_SIZE}$ (distance to cross one chunk in each axis).
    - Compute $\mathbf{rayStep}$: vector of $1$ or $-1$ indicating the direction of travel along each axis.
    - Compute $\mathbf{sideDist}$: distance to the next chunk boundary along each axis.
    - Set `ignoreFirst = true`.

2.  **Main Traversal Loop**:
    Iterate while the current voxel position $\mathbf{pos}$ is within map bounds [S1, unofficial]:
    ```python
    while in_map_bounds(pos):
        # 1. Retrieve chunk index and handle
        mapIndex = get_map_index(pos)
        mapTile = map[mapIndex]
        
        # 2. Check if chunk is solid
        # Flags bit 0 and 1: if (flags & 3) == 2, the chunk is solid
        if (mapTile.flags & 3) == 2:
            # 3. Compute exact entry point into the chunk
            # Step back along the ray to the exact boundary crossing
            min_side_dist = min(lastSideDist.x, lastSideDist.y, lastSideDist.z)
            updatedRayPos = rayPos + rayDir * (min_side_dist - EPSILON)
            
            # 4. Convert to chunk-local coordinates
            chunkRayPos = (updatedRayPos - pos) * INV_CHUNK_SIZE
            
            # 5. Clamp to avoid boundary artifacts
            # Temporary workaround to reduce noise
            chunkRayPos = clamp(chunkRayPos, EPSILON, CHUNK_SIZE - EPSILON)
            
            # 6. Step through the chunk
            hit_found = step_chunk(
                pos=pos,
                mapIndex=mapIndex,
                rayDir=rayDir,
                invRayDir=invRayDir,
                chunkRayPos=chunkRayPos,
                ignoreFirst=ignoreFirst,
                maxDepth=max_depth,
                hitNormal=hitNormal,
                voxel=voxel,
                colorAdd=colorAdd,
                colorMult=colorMult,
                refracted=refracted
            )
            
            # 7. Handle chunk step result
            if hit_found:
                # Convert back to world coordinates and terminate
                rayPos = pos + chunkRayPos * CHUNK_SIZE
                return True
            
            if refracted:
                # Update position, reinitialize DDA, reset lastSideDist
                rayPos = updatedRayPos # or updated based on refraction logic
                init_DDA(rayDir, invRayDir, rayPos, pos, deltaDist, rayStep, sideDist)
                lastSideDist = vec3(0.0)
                continue # Restart loop with new DDA state
            
            # If not hit and not refracted, continue to next chunk
            
        elif mapTile.flags & 1: # Pending load flag
            # Mark chunk as requested
            map[mapIndex].flags = 3
            requested = True
            # Note: Implementation may block or yield here. 
            # For continuous traversal, assume we skip or handle asynchronously.
            # Based on notes, we proceed to DDA advance if not blocking.
        
        # 8. Advance DDA to next chunk
        lastSideDist = sideDist
        iterate_DDA(deltaDist, rayStep, sideDist, pos, hitNormal)
        ignoreFirst = False
    ```

3.  **`step_chunk` Logic** (Internal to chunk):
    - Uses standard 3D DDA to traverse the $8 \times 8 \times 8$ grid starting from `chunkRayPos`.
    - Samples `DNcompressedVoxel` at each step.
    - If a solid voxel is found, returns `true`.
    - If refraction is enabled and the voxel is refractive, calculates new direction via Snell's law. If Total Internal Reflection (TIR) occurs, falls back to original direction to prevent infinite loops [S1, unofficial].

### Edge Cases and Pitfalls
- **Boundary Noise**: Clamping `chunkRayPos` to `[EPSILON, CHUNK_SIZE - EPSILON]` is a temporary workaround to reduce noise at chunk boundaries. The source explicitly marks this as a TODO to remove noise entirely [S1, unofficial].
- **First Voxel Skip**: `ignoreFirst` must be set to `false` after the initial DDA step. If left `true`, valid subsequent hits within the chunk may be skipped [S1, unofficial].
- **TIR Handling**: Without a fallback to the original direction upon Total Internal Reflection, rays may bounce infinitely, degrading performance [S1, unofficial].

## Subtopics

### [Voxel Refraction & Total Internal Reflection Handling](8624-voxel-refraction-total-internal-reflection-handling.md)
Modifies the ray direction upon entering a refractive voxel using Snell's law, with a fallback to the original direction if total internal reflection occurs to prevent infinite ray bouncing and performance degradation.

## Operating characteristics
- **Time Complexity**: $O(1)$ per chunk boundary check. The cost is proportional to the number of chunks traversed, not the number of voxels. Inside a chunk, complexity is proportional to the number of voxels intersected by the ray within the $8 \times 8 \times 8$ block [S1, unofficial].
- **Memory Complexity**: Fixed memory per chunk: $512$ compressed voxels. Each voxel is 64 bits, so $512 \times 8$ bytes = 4 KB per chunk, plus overhead for metadata (`DNchunk` struct) [S1, unofficial].
- **Performance**: Optimized for sparse scenes with large empty volumes. The DDA allows skipping entire chunks efficiently. Async chunk loading is supported via flags, though synchronization details are not specified [S1, unofficial].
- **Strengths**: Fast traversal of empty space; low memory footprint per chunk; simple data structure.
- **Limitations**: Boundary artifacts require clamping workarounds; fixed chunk size ($8^3$) may not be optimal for all scene densities; refraction handling is basic [S1, unofficial].

## Background
This method is used in the DoonEngine voxel rendering pipeline for map and chunk traversal [S1, unofficial]. It is an implementation of hierarchical ray marching specifically tailored for sparse voxel octrees with a fixed chunk size. The approach leverages the efficiency of DDA for grid traversal while adding a layer of hierarchy to skip empty regions.

## Sources
[S1] frozein/DoonEngine, https://github.com/frozein/DoonEngine, fetched 2026-10-26 (unofficial)
