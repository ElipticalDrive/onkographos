# Optimised Voxel Ray Marching

Also called: Chunk-based Voxel Traversal, Fixed-step voxel ray marching, Optimised CPU Ray Marching, Sector's Edge, Voxel Volume Ray Tracing, VoxelsRayTracingTutorial

## Overview
Optimised Voxel Ray Marching is a CPU-oriented ray traversal algorithm for voxel-based scenes that accelerates intersection testing by organizing voxels into fixed-size 3D sub-grids called "chunks" [S1, unofficial][S2, unofficial][S3, unofficial]. It solves the problem of inefficiently stepping through empty space in large voxel worlds by allowing the ray to skip entire chunks that contain no geometry, reducing the number of intersection checks in empty regions [S1, unofficial]. It sets itself apart from pure 3D grid traversal by maintaining a reference to the current working chunk and updating it only upon crossing chunk boundaries, thereby minimizing repeated bit-shifts, masks, and multiplication operations [S2, unofficial][S3, unofficial]. It differs from octree-based traversal by using fixed-size inner grids which simplifies implementation and memory layout [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Ray start position `start` (float3), ray velocity/direction `velocity` (float3, normalized), maximum ray length `max` (float), and a global `Map` structure containing a 3D array of `Chunk` references.
*   **Output:** Boolean `hit` (true if a solid block is intersected), and `axis` (enum indicating the face normal axis of the hit, used for reflection).

**Data Structures:**
The world is partitioned into a 3D grid of chunks.
*   **Map:** Contains a 3D array `Chunk[,,] chunks` indexed by `[chunkIndexX, chunkIndexY, chunkIndexZ]` [S2, unofficial][S3, unofficial].
*   **Chunk:** Contains a 1D array `Block[] data` of size `CHUNK_SIZE_CUBED` (32768) [S2, unofficial][S3, unofficial].
*   **Block:** A struct with a single byte field `byte kind` [S2, unofficial][S3, unofficial]. Layout is sequential with pack size 1.
*   **Constants:**
    *   `CHUNK_SIZE = 32` [S2, unofficial][S3, unofficial].
    *   `CHUNK_SIZE_SQUARED = 1024` [S2, unofficial][S3, unofficial].
    *   `CHUNK_SIZE_CUBED = 32768` [S2, unofficial][S3, unofficial].
    *   `SHIFT = 5` (log2(CHUNK_SIZE)) [S2, unofficial][S3, unofficial].
    *   `MASK = 0x1f` (31, CHUNK_SIZE - 1) [S2, unofficial][S3, unofficial].
    *   Map dimensions: `MAP_SIZE_X = 512`, `MAP_SIZE_Y = 160`, `MAP_SIZE_Z = 512` [S3, unofficial].

**Algorithm Steps:**

1.  **Initialization:**
    *   Convert `start` to integer voxel coordinates: `x = floor(start.X)`, `y = floor(start.Y)`, `z = floor(start.Z)` [S2, unofficial][S3, unofficial].
    *   Check bounds: If `x < 0` or `x >= MAP_SIZE_X` or `y < 0` or `y >= MAP_SIZE_Y` or `z < 0` or `z >= MAP_SIZE_Z`, set `hit = false` and return [S2, unofficial][S3, unofficial].
    *   Calculate chunk indices: `chunkIndexX = x >> SHIFT`, `chunkIndexY = y >> SHIFT`, `chunkIndexZ = z >> SHIFT` [S2, unofficial][S3, unofficial].
    *   Retrieve current chunk: `c = chunks[chunkIndexX, chunkIndexY, chunkIndexZ]` [S2, unofficial][S3, unofficial].
    *   Calculate chunk-relative coordinates: `i = x & MASK`, `j = y & MASK`, `k = z & MASK` [S2, unofficial][S3, unofficial].
    *   Calculate initial linear access index in chunk data: `access = j + i * CHUNK_SIZE + k * CHUNK_SIZE_SQUARED` [S2, unofficial][S3, unofficial].
    *   Calculate end position: `end = start + velocity` [S2, unofficial][S3, unofficial].
    *   **Short-circuit:** If `floor(end.X) == x` and `floor(end.Y) == y` and `floor(end.Z) == z`:
        *   If `c` is null or `c.data[access].kind == 0`, set `hit = false`.
        *   Else, set `hit = true`.
        *   Return [S2, unofficial][S3, unofficial].

2.  **Direction Setup:**
    *   Initialize direction-dependent variables for each axis based on the sign of `velocity` components [S2, unofficial][S3, unofficial].
    *   **Y Axis:**
        *   If `velocity.Y > 0`: `jStep = 1`, `jComparison = CHUNK_SIZE`, `jReset = 0`, `yAccessReset = -CHUNK_SIZE`, `yDist = (y - start.Y + 1)` [S2, unofficial][S3, unofficial].
        *   Else: `jStep = -1`, `jComparison = -1`, `jReset = CHUNK_SIZE - 1`, `yAccessReset = CHUNK_SIZE`, `yDist = (start.Y - y)` [S2, unofficial][S3, unofficial].
    *   **X Axis:**
        *   If `velocity.X > 0`: `iStep = 1`, `iComparison = CHUNK_SIZE`, `iReset = 0`, `xAccessReset = -CHUNK_SIZE_SQUARED`, `xAccessIncrement = CHUNK_SIZE`, `xDist = (x - start.X + 1)` [S2, unofficial][S3, unofficial].
        *   Else: `iStep = -1`, `iComparison = -1`, `iReset = CHUNK_SIZE - 1`, `xAccessReset = CHUNK_SIZE_SQUARED`, `xAccessIncrement = -CHUNK_SIZE`, `xDist = (start.X - x)` [S2, unofficial][S3, unofficial].
    *   **Z Axis:**
        *   If `velocity.Z > 0`: `kStep = 1`, `kComparison = CHUNK_SIZE`, `kReset = 0`, `zAccessIncrement = CHUNK_SIZE_SQUARED`, `zAccessReset = -CHUNK_SIZE_CUBED`, `zDist = (z - start.Z + 1)` [S2, unofficial][S3, unofficial].
        *   Else: `kStep = -1`, `kComparison = -1`, `kReset = CHUNK_SIZE - 1`, `zAccessIncrement = -CHUNK_SIZE_SQUARED`, `zAccessReset = CHUNK_SIZE_CUBED`, `zDist = (start.Z - z)` [S2, unofficial][S3, unofficial].

3.  **Distance Calculation:**
    *   Normalize `velocity` [S2, unofficial][S3, unofficial].
    *   Calculate inverse absolute velocity components: `xInverted = |1 / velocity.X|`, `yInverted = |1 / velocity.Y|`, `zInverted = |1 / velocity.Z|` [S2, unofficial][S3, unofficial].
    *   Calculate initial distance to next voxel boundary in `t`-units:
        *   If `velocity.X == 0`, `xDistance = PositiveInfinity`, else `xDistance = xInverted * xDist` [S2, unofficial][S3, unofficial].
        *   If `velocity.Y == 0`, `yDistance = PositiveInfinity`, else `yDistance = yInverted * yDist` [S2, unofficial][S3, unofficial].
        *   If `velocity.Z == 0`, `zDistance = PositiveInfinity`, else `zDistance = zInverted * zDist` [S2, unofficial][S3, unofficial].
    *   Initialize progress `t = 0.0` [S2, unofficial][S3, unofficial].

4.  **Traversal Loop:**
    *   While `t <= max`:
        *   **Hit Check:** If `c` is not null and `c.data[access].kind != 0`:
            *   Set `hit = true`.
            *   Return [S2, unofficial][S3, unofficial].
        *   **Find Closest Boundary:**
            *   If `yDistance < xDistance`:
                *   If `yDistance < zDistance`:
                    *   **Advance Y:**
                        *   `j += jStep` [S2, unofficial][S3, unofficial].
                        *   `access += jStep` [S2, unofficial][S3, unofficial].
                        *   **Chunk Exit Check:** If `j == jComparison`:
                            *   `j = jReset` [S2, unofficial][S3, unofficial].
                            *   `access += yAccessReset` [S2, unofficial][S3, unofficial].
                            *   `chunkIndexY += jStep` [S2, unofficial][S3, unofficial].
                            *   If `chunkIndexY < 0` or `chunkIndexY >= CHUNK_AMOUNT_Y`: Set `hit = false`, Return [S2, unofficial][S3, unofficial].
                            *   `c = chunks[chunkIndexX, chunkIndexY, chunkIndexZ]` [S2, unofficial][S3, unofficial].
                        *   `t = yDistance` [S2, unofficial][S3, unofficial].
                        *   `yDistance += yInverted` [S2, unofficial][S3, unofficial].
                        *   `axis = Axis.Y` [S2, unofficial][S3, unofficial].
                    *   Else (`zDistance <= yDistance`):
                        *   **Advance Z:**
                            *   `k += kStep` [S2, unofficial][S3, unofficial].
                            *   `access += zAccessIncrement` [S2, unofficial][S3, unofficial].
                            *   **Chunk Exit Check:** If `k == kComparison`:
                                *   `k = kReset` [S2, unofficial][S3, unofficial].
                                *   `access += zAccessReset` [S2, unofficial][S3, unofficial].
                                *   `chunkIndexZ += kStep` [S2, unofficial][S3, unofficial].
                                *   If `chunkIndexZ < 0` or `chunkIndexZ >= CHUNK_AMOUNT_Z`: Set `hit = false`, Return [S2, unofficial][S3, unofficial].
                                *   `c = chunks[chunkIndexX, chunkIndexY, chunkIndexZ]` [S2, unofficial][S3, unofficial].
                            *   `t = zDistance` [S2, unofficial][S3, unofficial].
                            *   `zDistance += zInverted` [S2, unofficial][S3, unofficial].
                            *   `axis = Axis.Z` [S2, unofficial][S3, unofficial].
                *   Else (`xDistance <= yDistance`):
                    *   **Advance X:**
                        *   `i += iStep` [S2, unofficial][S3, unofficial].
                        *   `access += xAccessIncrement` [S2, unofficial][S3, unofficial].
                        *   **Chunk Exit Check:** If `i == iComparison`:
                            *   `i = iReset` [S2, unofficial][S3, unofficial].
                            *   `access += xAccessReset` [S2, unofficial][S3, unofficial].
                            *   `chunkIndexX += iStep` [S2, unofficial][S3, unofficial].
                            *   If `chunkIndexX < 0` or `chunkIndexX >= CHUNK_AMOUNT_X`: Set `hit = false`, Return [S2, unofficial][S3, unofficial].
                            *   `c = chunks[chunkIndexX, chunkIndexY, chunkIndexZ]` [S2, unofficial][S3, unofficial].
                        *   `t = xDistance` [S2, unofficial][S3, unofficial].
                        *   `xDistance += xInverted` [S2, unofficial][S3, unofficial].
                        *   `axis = Axis.X` [S2, unofficial][S3, unofficial].
            *   Else (`xDistance <= yDistance`):
                *   If `xDistance < zDistance`:
                    *   **Advance X:** (Same logic as above)
                *   Else (`zDistance <= xDistance`):
                    *   **Advance Z:** (Same logic as above)

5.  **Completion:**
    *   If the loop completes without hitting a block, set `hit = false` [S2, unofficial][S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_march(start: Vector3, velocity: Vector3, max_dist: float, map_chunks: Array3D[Chunk]) -> Tuple[bool, Axis]:
    # Constants
    CHUNK_SIZE = 32
    CHUNK_SIZE_SQ = 1024
    CHUNK_SIZE_CB = 32768
    SHIFT = 5
    MASK = 31
    MAP_SIZE_X = 512
    MAP_SIZE_Y = 160
    MAP_SIZE_Z = 512
    CHUNK_AMOUNT_X = 16
    CHUNK_AMOUNT_Y = 5
    CHUNK_AMOUNT_Z = 16

    # 1. Initialization
    x = floor(start.x)
    y = floor(start.y)
    z = floor(start.z)

    # Bounds check
    if x < 0 or x >= MAP_SIZE_X or y < 0 or y >= MAP_SIZE_Y or z < 0 or z >= MAP_SIZE_Z:
        return False, Axis.None

    # Chunk indices
    chunk_idx_x = x >> SHIFT
    chunk_idx_y = y >> SHIFT
    chunk_idx_z = z >> SHIFT

    # Current chunk
    c = map_chunks[chunk_idx_x, chunk_idx_y, chunk_idx_z]

    # Chunk-relative coords
    i = x & MASK
    j = y & MASK
    k = z & MASK

    # Access index
    access = j + i * CHUNK_SIZE + k * CHUNK_SIZE_SQ

    # End position
    end = start + velocity
    end_x = floor(end.x)
    end_y = floor(end.y)
    end_z = floor(end.z)

    # Short-circuit: start and end in same voxel
    if x == end_x and y == end_y and z == end_z:
        if c is None or c.data[access].kind == 0:
            return False, Axis.None
        else:
            return True, Axis.None

    # 2. Direction Setup
    # Y Axis
    if velocity.y > 0:
        j_step = 1
        j_comparison = CHUNK_SIZE
        j_reset = 0
        y_access_reset = -CHUNK_SIZE
        y_dist = (y - start.y + 1)
    else:
        j_step = -1
        j_comparison = -1
        j_reset = CHUNK_SIZE - 1
        y_access_reset = CHUNK_SIZE
        y_dist = (start.y - y)

    # X Axis
    if velocity.x > 0:
        i_step = 1
        i_comparison = CHUNK_SIZE
        i_reset = 0
        x_access_reset = -CHUNK_SIZE_SQ
        x_access_increment = CHUNK_SIZE
        x_dist = (x - start.x + 1)
    else:
        i_step = -1
        i_comparison = -1
        i_reset = CHUNK_SIZE - 1
        x_access_reset = CHUNK_SIZE_SQ
        x_access_increment = -CHUNK_SIZE
        x_dist = (start.x - x)

    # Z Axis
    if velocity.z > 0:
        k_step = 1
        k_comparison = CHUNK_SIZE
        k_reset = 0
        z_access_increment = CHUNK_SIZE_SQ
        z_access_reset = -CHUNK_SIZE_CB
        z_dist = (z - start.z + 1)
    else:
        k_step = -1
        k_comparison = -1
        k_reset = CHUNK_SIZE - 1
        z_access_increment = -CHUNK_SIZE_SQ
        z_access_reset = CHUNK_SIZE_CB
        z_dist = (start.z - z)

    # 3. Distance Calculation
    velocity = normalize(velocity)
    
    x_inv = abs(1.0 / velocity.x) if velocity.x != 0 else float('inf')
    y_inv = abs(1.0 / velocity.y) if velocity.y != 0 else float('inf')
    z_inv = abs(1.0 / velocity.z) if velocity.z != 0 else float('inf')

    x_dist_t = x_inv * x_dist if velocity.x != 0 else float('inf')
    y_dist_t = y_inv * y_dist if velocity.y != 0 else float('inf')
    z_dist_t = z_inv * z_dist if velocity.z != 0 else float('inf')

    t = 0.0
    axis = Axis.None

    # 4. Traversal Loop
    while t <= max_dist:
        # Hit Check
        if c is not None and c.data[access].kind != 0:
            return True, axis

        # Find Closest Boundary
        if y_dist_t < x_dist_t:
            if y_dist_t < z_dist_t:
                # Advance Y
                j += j_step
                access += j_step
                
                if j == j_comparison:
                    j = j_reset
                    access += y_access_reset
                    chunk_idx_y += j_step
                    
                    if chunk_idx_y < 0 or chunk_idx_y >= CHUNK_AMOUNT_Y:
                        return False, Axis.None
                    
                    c = map_chunks[chunk_idx_x, chunk_idx_y, chunk_idx_z]
                
                t = y_dist_t
                y_dist_t += y_inv
                axis = Axis.Y
            else:
                # Advance Z
                k += k_step
                access += z_access_increment
                
                if k == k_comparison:
                    k = k_reset
                    access += z_access_reset
                    chunk_idx_z += k_step
                    
                    if chunk_idx_z < 0 or chunk_idx_z >= CHUNK_AMOUNT_Z:
                        return False, Axis.None
                    
                    c = map_chunks[chunk_idx_x, chunk_idx_y, chunk_idx_z]
                
                t = z_dist_t
                z_dist_t += z_inv
                axis = Axis.Z
        else:
            if x_dist_t < z_dist_t:
                # Advance X
                i += i_step
                access += x_access_increment
                
                if i == i_comparison:
                    i = i_reset
                    access += x_access_reset
                    chunk_idx_x += i_step
                    
                    if chunk_idx_x < 0 or chunk_idx_x >= CHUNK_AMOUNT_X:
                        return False, Axis.None
                    
                    c = map_chunks[chunk_idx_x, chunk_idx_y, chunk_idx_z]
                
                t = x_dist_t
                x_dist_t += x_inv
                axis = Axis.X
            else:
                # Advance Z
                k += k_step
                access += z_access_increment
                
                if k == k_comparison:
                    k = k_reset
                    access += z_access_reset
                    chunk_idx_z += k_step
                    
                    if chunk_idx_z < 0 or chunk_idx_z >= CHUNK_AMOUNT_Z:
                        return False, Axis.None
                    
                    c = map_chunks[chunk_idx_x, chunk_idx_y, chunk_idx_z]
                
                t = z_dist_t
                z_dist_t += z_inv
                axis = Axis.Z

    return False, Axis.None
```

**Edge Cases and Pitfalls:**
*   **Self-intersection:** In rendering contexts (e.g., [S1, unofficial]), shadow rays are offset by `n * EPS` (where `EPS = 5e-5`) to avoid hitting the surface they originated from [S1, unofficial].
*   **Boundary Checking:** The `checkBoundaries` function ensures voxel coordinates are within `[0, size)` to prevent out-of-bounds memory access [S1, unofficial]. In the chunked version, chunk index bounds are checked explicitly upon chunk exit [S2, unofficial][S3, unofficial].
*   **Null Chunks:** Chunks containing no blocks may be null (uninitialized) to save memory. The algorithm must check for `c == null` before accessing `c.data` [S2, unofficial][S3, unofficial].
*   **Zero Velocity Components:** If a velocity component is 0, the distance to the next boundary in that axis is set to `PositiveInfinity` to prevent division by zero and infinite loops [S2, unofficial][S3, unofficial].

## Subtopics

### [Chunked Voxel Traversal](8311-chunked-voxel-traversal.md)
Chunked Voxel Traversal is a hierarchical ray traversal method for sparse voxel grids that accelerates intersection testing by organizing voxels into a 3D grid of smaller 3D sub-grids called "chunks". It solves the problem of inefficiently stepping through empty space in large voxel worlds by allowing the ray to skip entire chunks that contain no geometry. It sets itself apart from pure 3D grid traversal by reducing the number of intersection checks in empty regions, and from octree-based traversal by using fixed-size inner grids which simplifies implementation and memory layout.

## Operating characteristics
*   **Time Complexity:** The traversal step per voxel is $O(1)$, dominated by bitwise operations and array lookups. The total time is proportional to the number of voxels traversed.
*   **Memory Complexity:** $O(N^3)$ for the full grid, but optimized by using sparse chunks. Only chunks containing geometry are allocated, reducing memory footprint for sparse worlds [S2, unofficial].
*   **Measured Performance:** On a Ryzen 5 1600 CPU, average particle collision ray march (1-10 blocks) takes 250 nanoseconds, allowing 64,000 rays per frame per thread at 60fps. Ray length 200-400 blocks takes 3400 nanoseconds, yielding 4,700 rays per 16ms frame [S2, unofficial][S3, unofficial].
*   **Strengths:** Fast CPU performance due to minimized arithmetic operations (bit-shifts/masks instead of divisions/multiplications for chunk indexing). Good for collision detection in voxel games [S2, unofficial][S3, unofficial].
*   **Limitations:** Fixed chunk size may lead to memory overhead for very sparse worlds if many chunks are partially filled. Not suitable for GPU rendering where coalesced memory access and warp divergence are critical concerns (unlike [Voxel-based Ray Tracing for GI and Specular Reflections](4091-voxel-based-ray-tracing-for-gi-and-specular-reflections.md)).
*   **When to use:** CPU-based collision detection, particle physics, and voxel games where memory bandwidth and cache locality for chunk-local data are important [S2, unofficial][S3, unofficial].
*   **When not to use:** GPU ray tracing, highly dense uniform grids where overhead of chunk management outweighs benefits, or scenes requiring hierarchical LOD.

## Background
Optimised Voxel Ray Marching is based on "A Fast Voxel Traversal Algorithm for Ray Tracing" by John Amanatides and Andrew Woo [S2, unofficial][S3, unofficial]. It was specifically optimized for CPU collision detection in the game "Sector's Edge" by keeping block lookups within the current working chunk to avoid repeated chunk boundary calculations [S2, unofficial][S3, unofficial]. The implementation has been documented in tutorials and open-source repositories such as "VoxelsRayTracingTutorial" and "Vercidium/voxel-ray-marching" [S1, unofficial][S2, unofficial][S3, unofficial].

## Sources
[S1] AndrewB330/VoxelsRayTracingTutorial, https://github.com/AndrewB330/VoxelsRayTracingTutorial, fetched 2026-10-24 (unofficial)
[S2] Optimised CPU Ray Marching, https://vercidium.com/blog/optimised-voxel-raymarching/, fetched 2026-10-24 (unofficial)
[S3] Vercidium/voxel-ray-marching, https://github.com/Vercidium/voxel-ray-marching, fetched 2026-10-24 (unofficial)
