# Hierarchical DDA with Chunk Stepping

Also called: 3D Voxel Traversal, Chunked DDA Ray Marching, Voxel Ray Marching with DDA & Chunk Stepping

## Overview
Hierarchical DDA with Chunk Stepping traces a ray through a hierarchical voxel map by stepping through chunks using a 3D Digital Differential Analyzer (DDA) algorithm [S1, unofficial]. It builds on [Hierarchical DDA](5710-hierarchical-dda.md) by treating the lowest hierarchy level as solid $8 \times 8 \times 8$ blocks, allowing the ray to skip empty regions in $O(1)$ time per chunk boundary [S1, unofficial]. The method detects solid chunks, converts the world-space entry point to chunk-local coordinates, steps through the voxel grid, and continues until a hit is found or map bounds are exceeded [S1, unofficial].

## How it works
The method maintains a DDA state to determine when the ray crosses a chunk boundary [S1, unofficial]. Upon entering a solid chunk, it computes the exact entry point, converts it to local coordinates, and steps through the $8 \times 8 \times 8$ voxel grid [S1, unofficial]. If the chunk is not solid or has a pending load flag, the algorithm marks it for asynchronous loading and advances the DDA to the next chunk [S1, unofficial].

### Inputs and Outputs
**Inputs:**
- Ray origin: $\mathbf{rayPos}$ (world space) [S1, unofficial]
- Ray direction: $\mathbf{rayDir}$ (normalized) [S1, unofficial]
- Inverse ray direction: $\mathbf{invRayDir} = 1.0 / \mathbf{rayDir}$ [S1, unofficial]
- Current chunk position: $\mathbf{pos}$ (integer voxel coordinates of the chunk's origin) [S1, unofficial]
- Map bounds: defined by the extent of the loaded chunks [S1, unofficial]

**Outputs:**
- Boolean: `true` if a hit was found within the chunk; `false` if the ray exited the map or chunk without hitting [S1, unofficial].
- Updated ray state: $\mathbf{rayPos}$, $\mathbf{hitNormal}$, $\mathbf{voxel}$ data, and refraction flags [S1, unofficial].

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
- `pos`: `DNivec3` (3D integer vector) storing the chunk's world-space origin in voxels [S1, unofficial].
- `updated`: `bool` indicating if the chunk data is current [S1, unofficial].
- `numVoxels`: `uint32_t` count of active voxels [S1, unofficial].
- `numVoxelsGpu`: `uint32_t` count for GPU transfer [S1, unofficial].
- `voxels`: Array of $8 \times 8 \times 8 = 512$ `DNcompressedVoxel` elements [S1, unofficial].

### Parameters
- `CHUNK_SIZE`: Linear dimension of a chunk in voxels. Value: $8$ [S1, unofficial].
- `INV_CHUNK_SIZE`: Reciprocal of `CHUNK_SIZE` ($1/8 = 0.125$), used to scale chunk-local coordinates back to world space [S1, unofficial].
- `EPSILON`: Small floating-point offset used to prevent self-intersection and clamp boundary coordinates. Exact value not specified in the sources [S1, unofficial].
- `ignoreFirst`: Boolean flag that skips the first voxel hit during DDA iteration to prevent re-detecting the entry voxel [S1, unofficial]. Initialized to `true` for the first step, then set to `false` after the initial DDA advance [S1, unofficial].

### Algorithm Steps

1.  **Initialize DDA State**:
    Before the main loop, initialize the DDA variables using the ray properties and the current chunk position $\mathbf{pos}$ [S1, unofficial]:
    - Compute $\mathbf{deltaDist} = \mathbf{invRayDir} \times \text{CHUNK\_SIZE}$ (distance to cross one chunk in each axis) [S1, unofficial].
    - Compute $\mathbf{rayStep}$: vector of $1$ or $-1$ indicating the direction of travel along each axis [S1, unofficial].
    - Compute $\mathbf{sideDist}$: distance to the next chunk boundary along each axis [S1, unofficial].
    - Set `ignoreFirst = true` and `lastSideDist = vec3(0.0)` [S1, unofficial].

2.  **Main Traversal Loop**:
    Iterate while the current voxel position $\mathbf{pos}$ is within map bounds [S1, unofficial]:
    ```python
    CHUNK_SIZE = 8
    INV_CHUNK_SIZE = 1.0 / CHUNK_SIZE
    EPSILON = 1e-6  # Not given in the sources [G]
    
    def get_map_index(pos):
        # Not given in the sources. Typically a spatial hash of pos [G]
        return hash(pos)
    
    def in_map_bounds(pos):
        # Not given in the sources. Checks if pos is within the loaded map extent [G]
        return True  # Placeholder for bounds check [G]
    
    def init_DDA(rayDir, invRayDir, rayPos, pos, deltaDist, rayStep, sideDist):
        deltaDist = invRayDir * CHUNK_SIZE
        rayStep = vec3(1 if rayDir.x >= 0 else -1, 1 if rayDir.y >= 0 else -1, 1 if rayDir.z >= 0 else -1)
        sideDist.x = (pos.x + (1 if rayStep.x > 0 else 0) - rayPos.x) * invRayDir.x
        sideDist.y = (pos.y + (1 if rayStep.y > 0 else 0) - rayPos.y) * invRayDir.y
        sideDist.z = (pos.z + (1 if rayStep.z > 0 else 0) - rayPos.z) * invRayDir.z
    
    def iterate_DDA(deltaDist, rayStep, sideDist, pos, hitNormal):
        if sideDist.x < sideDist.y:
            if sideDist.x < sideDist.z:
                pos.x += rayStep.x
                sideDist.x += deltaDist.x
                hitNormal = vec3(-rayStep.x, 0, 0)
            else:
                pos.z += rayStep.z
                sideDist.z += deltaDist.z
                hitNormal = vec3(0, 0, -rayStep.z)
        else:
            if sideDist.y < sideDist.z:
                pos.y += rayStep.y
                sideDist.y += deltaDist.y
                hitNormal = vec3(0, -rayStep.y, 0)
            else:
                pos.z += rayStep.z
                sideDist.z += deltaDist.z
                hitNormal = vec3(0, 0, -rayStep.z)
    
    def step_chunk(pos, mapIndex, rayDir, invRayDir, chunkRayPos, ignoreFirst, maxDepth, hitNormal, voxel, colorAdd, colorMult, refracted):
        # Standard 3D DDA inside the 8x8x8 chunk [S1, unofficial]
        localPos = chunkRayPos
        localStep = vec3(1 if rayDir.x >= 0 else -1, 1 if rayDir.y >= 0 else -1, 1 if rayDir.z >= 0 else -1)
        localDelta = invRayDir * 1.0
        localSide = (localPos - (localPos - localPos)) * localDelta  # Simplified init [G]
        for _ in range(maxDepth):
            idx = vec3(int(localPos.x), int(localPos.y), int(localPos.z))
            if 0 <= idx.x < CHUNK_SIZE and 0 <= idx.y < CHUNK_SIZE and 0 <= idx.z < CHUNK_SIZE:
                if not ignoreFirst:
                    sampled = map[mapIndex].voxels[idx.x][idx.y][idx.z]
                    if sampled is not empty:
                        hitNormal = sampled.normal
                        voxel = sampled
                        return True
                ignoreFirst = False
            # Advance local DDA
            if localSide.x < localSide.y:
                if localSide.x < localSide.z:
                    localPos.x += localStep.x
                    localSide.x += localDelta.x
                else:
                    localPos.z += localStep.z
                    localSide.z += localDelta.z
            else:
                if localSide.y < localSide.z:
                    localPos.y += localStep.y
                    localSide.y += localDelta.y
                else:
                    localPos.z += localStep.z
                    localSide.z += localDelta.z
        return False

    rayPos = initial_ray_origin
    pos = initial_chunk_origin
    deltaDist = vec3(0.0)
    rayStep = vec3(0.0)
    sideDist = vec3(0.0)
    lastSideDist = vec3(0.0)
    ignoreFirst = True
    refracted = False
    hitNormal = vec3(0.0)
    voxel = None
    colorAdd = vec3(0.0)
    colorMult = vec3(1.0)
    max_depth = 1000  # Not given in the sources [G]

    init_DDA(rayDir, invRayDir, rayPos, pos, deltaDist, rayStep, sideDist)

    while in_map_bounds(pos):
        mapIndex = get_map_index(pos)
        mapTile = map[mapIndex]
        
        if (mapTile.flags & 3) == 2:
            min_side_dist = min(lastSideDist.x, lastSideDist.y, lastSideDist.z)
            updatedRayPos = rayPos + rayDir * (min_side_dist - EPSILON)
            chunkRayPos = (updatedRayPos - pos) * CHUNK_SIZE
            chunkRayPos = clamp(chunkRayPos, vec3(EPSILON), vec3(CHUNK_SIZE - EPSILON))
            
            hit_found = step_chunk(pos, mapIndex, rayDir, invRayDir, chunkRayPos, ignoreFirst, max_depth, hitNormal, voxel, colorAdd, colorMult, refracted)
            
            if hit_found:
                rayPos = pos + chunkRayPos * INV_CHUNK_SIZE
                return True
            
            if refracted:
                rayPos = updatedRayPos
                init_DDA(rayDir, invRayDir, rayPos, pos, deltaDist, rayStep, sideDist)
                lastSideDist = vec3(0.0)
                continue
            
        elif mapTile.flags & 1:
            map[mapIndex].flags = 3
            requested = True
        
        lastSideDist = sideDist
        iterate_DDA(deltaDist, rayStep, sideDist, pos, hitNormal)
        ignoreFirst = False
    ```

3.  **`step_chunk` Logic** (Internal to chunk):
    - Uses standard 3D DDA to traverse the $8 \times 8 \times 8$ grid starting from `chunkRayPos` [S1, unofficial].
    - Samples `DNcompressedVoxel` at each step [S1, unofficial].
    - If a solid voxel is found, returns `true` [S1, unofficial].
    - If refraction is enabled and the voxel is refractive, calculates new direction via Snell's law [S1, unofficial].

### Edge Cases and Pitfalls
- **Boundary Noise**: Clamping `chunkRayPos` to `[EPSILON, CHUNK_SIZE - EPSILON]` reduces boundary noise but is explicitly marked as a temporary workaround in the source code [S1, unofficial].
- **First Voxel Skip**: `ignoreFirst` must be set to `false` after the initial DDA step to prevent skipping valid subsequent hits within the chunk [S1, unofficial].
- **Async Loading**: If a chunk has a pending load flag, the algorithm marks it as requested and continues traversal, relying on asynchronous loading mechanisms not detailed in the sources [S1, unofficial].

## Subtopics

### [Voxel Refraction & Total Internal Reflection Handling](8624-voxel-refraction-total-internal-reflection-handling.md)
Modifies the ray direction upon entering a refractive voxel using Snell's law, ensuring physically plausible light bending within sparse voxel grids. It distinguishes itself from opaque traversal by continuing the ray after interaction and from non-refractive transparency by actively modifying the ray vector. A critical feature is the fallback to the original direction if total internal reflection (TIR) occurs, preventing infinite ray bouncing and performance degradation. It builds on [Hierarchical DDA with Chunk Stepping](8591-hierarchical-dda-with-chunk-stepping.md) by handling the transition between chunks with varying refractive indices.

## Operating characteristics
- **Time Complexity**: $O(1)$ per chunk boundary check for DDA iteration [S1, unofficial]. The cost scales linearly with the number of chunks traversed, not the total voxel count [S1, unofficial].
- **Memory Complexity**: Fixed memory per chunk at 512 compressed voxels [S1, unofficial]. Each voxel occupies 64 bits, resulting in 4 KB per chunk, plus metadata overhead for the `DNchunk` struct [S1, unofficial].
- **Performance**: Optimized for sparse scenes with large empty volumes, as the DDA skips entire chunks efficiently [S1, unofficial]. Async chunk loading is supported via flags, though synchronization details are not specified [S1, unofficial].
- **Strengths**: Fast traversal of empty space; low memory footprint per chunk; simple data structure layout [S1, unofficial].
- **Limitations**: Boundary artifacts require clamping workarounds; fixed chunk size ($8^3$) may not be optimal for all scene densities; refraction handling is basic [S1, unofficial].

## Background
This method is used in the DoonEngine voxel rendering pipeline for map and chunk traversal [S1, unofficial]. It is an implementation of hierarchical ray marching specifically tailored for sparse voxel octrees with a fixed chunk size [S1, unofficial]. The approach leverages the efficiency of DDA for grid traversal while adding a layer of hierarchy to skip empty regions [S1, unofficial].

## Sources
[S1] frozein/DoonEngine, https://github.com/frozein/DoonEngine, fetched 2026-10-26 (unofficial)
