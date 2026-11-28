# Optimised Voxel Ray Marching

Also called: Chunk-based Voxel Traversal, Fixed-step voxel ray marching, Optimised CPU Ray Marching, Sector's Edge, Voxel Volume Ray Tracing, VoxelsRayTracingTutorial

## Overview
Optimised Voxel Ray Marching is a ray traversal algorithm for voxel-based scenes that advances a ray to the next voxel boundary to check for occupancy at each step [S2, S3, unofficial]. It solves the problem of inefficiently stepping through empty space in large voxel worlds by optimizing memory access patterns, specifically by keeping block lookups within the current working chunk to minimize bitshifts, masks, and multiplication operations [S2, S3, unofficial]. It sets itself apart from [Regular Grid Traversal](5939-regular-grid-traversal.md) by maintaining a reference to the current working chunk and updating it only when the ray crosses a chunk boundary, thereby avoiding repeated chunk index calculations for every voxel step [S2, S3, unofficial].

## How it works
The method implements a ray marcher optimized for CPU collision detection by maintaining a reference to the current working chunk and updating it only when the ray crosses a chunk boundary [S2, S3, unofficial].

### Inputs and Outputs
**Inputs:**
*   `start`: Ray origin in world/voxel space (Vector3) [S2, S3, unofficial].
*   `velocity`: Ray direction vector (normalized) [S2, S3, unofficial].
*   `max`: Maximum ray length in world units [S2, S3, unofficial].
*   `chunks`: 3D array of Chunk references indexed by `[chunkIndexX, chunkIndexY, chunkIndexZ]` [S3, unofficial].
*   `MAP_SIZE`: World dimensions (e.g., $512 \times 160 \times 512$) [S3, unofficial].
*   `CHUNK_SIZE`: Side length of a chunk in blocks (typically 32) [S2, S3, unofficial].

**Outputs:**
*   `hit`: Boolean indicating if a solid block was intersected [S2, S3, unofficial].
*   `axis`: The axis (X, Y, or Z) of the face hit (used for reflection) [S2, S3, unofficial].

### Data Structures
**Chunk:**
*   Contains a 1D array `data` of size $CHUNK\_SIZE^3$ (e.g., 32768 for size 32) [S2, S3, unofficial].
*   `data` stores `Block` structs sequentially [S2, S3, unofficial].

**Block:**
*   Single byte field `kind`. Value 0 indicates empty; non-zero indicates solid [S2, S3, unofficial].

**Map:**
*   3D array `chunks[,,]` of Chunk references [S3, unofficial].
*   Chunk indices are derived from world coordinates via bit-shifting: $idx = coord \gg SHIFT$, where $SHIFT = \log_2(CHUNK\_SIZE)$ [S3, unofficial].
*   Chunk-relative coordinates are derived via bitmask: $rel = coord \& MASK$, where $MASK = CHUNK\_SIZE - 1$ [S3, unofficial].

### Parameters
*   `EPS`: $5 \times 10^{-5}$, used to offset ray origins to prevent self-intersection [S1, unofficial].
*   `CHUNK_SIZE`: 32 [S2, S3, unofficial].
*   `SHIFT`: 5 [S2, S3, unofficial].
*   `MASK`: 0x1f (31) [S2, S3, unofficial].
*   `MAP_SIZE_X/Z`: 512, `MAP_SIZE_Y`: 160 [S3, unofficial].

### Algorithm Steps
1.  **Initialization:**
    *   Convert `start` to integer voxel coordinates: $x = \lfloor start.x \rfloor$, $y = \lfloor start.y \rfloor$, $z = \lfloor start.z \rfloor$ [S2, S3, unofficial].
    *   Check bounds: If $(x, y, z)$ is outside `MAP_SIZE`, return `hit = false` [S2, S3, unofficial].
    *   Calculate chunk indices: $cx = x \gg SHIFT$, $cy = y \gg SHIFT$, $cz = z \gg SHIFT$ [S3, unofficial].
    *   Retrieve current chunk: $c = chunks[cx, cy, cz]$ [S3, unofficial].
    *   Calculate chunk-relative coordinates: $i = x \& MASK$, $j = y \& MASK$, $k = z \& MASK$ [S3, unofficial].
    *   Calculate linear access index in chunk: $access = j + i \cdot CHUNK\_SIZE + k \cdot CHUNK\_SIZE^2$ [S3, unofficial].
    *   Check start block: If $c$ is null or $c.data[access].kind == 0$, return `hit = false` (unless start/end are same block, handled separately) [S2, S3, unofficial].

2.  **Direction Setup:**
    *   Normalize `velocity`.
    *   Calculate inverted components: $invX = |1/velocity.x|$, $invY = |1/velocity.y|$, $invZ = |1/velocity.z|$ [S3, unofficial].
    *   Initialize direction-dependent variables for each axis based on the sign of the velocity component:
        *   If $velocity.axis > 0$:
            *   $step = 1$
            *   $comparison = CHUNK\_SIZE$ (boundary exit condition)
            *   $reset = 0$
            *   $accessReset = -CHUNK\_SIZE$ (for X/Z) or $-1$ (for Y)
            *   $accessIncrement = CHUNK\_SIZE$ (for X) or $CHUNK\_SIZE^2$ (for Z) or $1$ (for Y)
            *   $dist = (coord - start.coord + 1)$
        *   If $velocity.axis < 0$:
            *   $step = -1$
            *   $comparison = -1$
            *   $reset = CHUNK\_SIZE - 1$
            *   $accessReset = CHUNK\_SIZE$ (for X/Z) or $1$ (for Y)
            *   $accessIncrement = -CHUNK\_SIZE$ (for X) or $-CHUNK\_SIZE^2$ (for Z) or $-1$ (for Y)
            *   $dist = (start.coord - coord)$
    *   Calculate initial distance to next boundary in $t$-units:
        *   $tDist_{axis} = velocity.axis == 0 ? \infty : inv_{axis} \cdot dist_{axis}$ [S2, S3, unofficial].

3.  **Marching Loop:**
    *   Initialize $t = 0.0$.
    *   While $t \le max$:
        *   **Occupancy Check:** If $c$ is not null and $c.data[access].kind \neq 0$, set `hit = true`, return.
        *   **Find Closest Boundary:** Compare $tDistX, tDistY, tDistZ$.
        *   **Advance Axis:**
            *   Let $axis$ be the axis with the minimum $tDist$.
            *   Update relative coordinate: $coord_{rel} += step_{axis}$.
            *   Update access index: $access += step_{axis}$ (for Y) or $access += accessIncrement_{axis}$ (for X/Z).
            *   **Chunk Boundary Check:** If $coord_{rel} == comparison_{axis}$:
                *   Reset relative coordinate: $coord_{rel} = reset_{axis}$.
                *   Update access index: $access += accessReset_{axis}$.
                *   Update chunk index: $chunkIndex_{axis} += step_{axis}$.
                *   Check new chunk bounds: If $chunkIndex_{axis}$ is out of map bounds, return `hit = false`.
                *   Load new chunk: $c = chunks[cx, cy, cz]$.
            *   Update progress: $t = tDist_{axis}$.
            *   Update distance to next boundary: $tDist_{axis} += inv_{axis}$.
            *   Store hit axis: $axis = axis$.

4.  **Termination:**
    *   If loop completes without hit, return `hit = false`.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_march(start: Vector3, velocity: Vector3, max_dist: float, chunks: Array3D, map_size: Vector3i, chunk_size: int) -> bool:
    INF = 1e9
    # 1. Initialization
    x = int(start.x)
    y = int(start.y)
    z = int(start.z)
    
    # Bounds check
    if x < 0 or x >= map_size.x or y < 0 or y >= map_size.y or z < 0 or z >= map_size.z:
        return False

    shift = 5  # log2(chunk_size)
    mask = 31  # chunk_size - 1
    
    cx = x >> shift
    cy = y >> shift
    cz = z >> shift
    
    c = chunks[cx, cy, cz]
    if c is None:
        return False
        
    i = x & mask
    j = y & mask
    k = z & mask
    
    access = j + i * chunk_size + k * (chunk_size * chunk_size)
    
    # Handle single-block ray
    end_x = int(start.x + velocity.x * max_dist)
    end_y = int(start.y + velocity.y * max_dist)
    end_z = int(start.z + velocity.z * max_dist)
    
    if x == end_x and y == end_y and z == end_z:
        return c.data[access].kind != 0

    # 2. Direction Setup
    velocity = normalize(velocity)
    inv_x = abs(1.0 / velocity.x) if velocity.x != 0 else INF
    inv_y = abs(1.0 / velocity.y) if velocity.y != 0 else INF
    inv_z = abs(1.0 / velocity.z) if velocity.z != 0 else INF
    
    # Setup axis variables for X
    if velocity.x > 0:
        x_step, x_comparison, x_reset = 1, chunk_size, 0
        x_access_reset, x_access_inc = -(chunk_size * chunk_size), chunk_size
        x_dist = x - start.x + 1
    else:
        x_step, x_comparison, x_reset = -1, -1, chunk_size - 1
        x_access_reset, x_access_inc = chunk_size * chunk_size, -chunk_size
        x_dist = start.x - x
        
    # Setup axis variables for Y
    if velocity.y > 0:
        y_step, y_comparison, y_reset = 1, chunk_size, 0
        y_access_reset, y_access_inc = -1, 1
        y_dist = y - start.y + 1
    else:
        y_step, y_comparison, y_reset = -1, -1, chunk_size - 1
        y_access_reset, y_access_inc = 1, -1
        y_dist = start.y - y
        
    # Setup axis variables for Z
    if velocity.z > 0:
        z_step, z_comparison, z_reset = 1, chunk_size, 0
        z_access_reset, z_access_inc = -(chunk_size * chunk_size * chunk_size), chunk_size * chunk_size
        z_dist = z - start.z + 1
    else:
        z_step, z_comparison, z_reset = -1, -1, chunk_size - 1
        z_access_reset, z_access_inc = chunk_size * chunk_size * chunk_size, -chunk_size * chunk_size
        z_dist = start.z - z
        
    t_dist_x = inv_x * x_dist if velocity.x != 0 else INF
    t_dist_y = inv_y * y_dist if velocity.y != 0 else INF
    t_dist_z = inv_z * z_dist if velocity.z != 0 else INF
    
    t = 0.0
    hit_axis = None

    # 3. Marching Loop
    while t <= max_dist:
        if c is not None and c.data[access].kind != 0:
            return True
            
        # Find closest boundary
        if t_dist_y < t_dist_x:
            if t_dist_y < t_dist_z:
                # Advance Y
                j += y_step
                access += y_step
                
                if j == y_comparison:
                    j = y_reset
                    access += y_access_reset
                    cy += y_step
                    if cy < 0 or cy >= map_size.y // chunk_size:
                        return False
                    c = chunks[cx, cy, cz]
                    
                t = t_dist_y
                t_dist_y += inv_y
                hit_axis = 'Y'
            else:
                # Advance Z
                k += z_step
                access += z_access_inc
                
                if k == z_comparison:
                    k = z_reset
                    access += z_access_reset
                    cz += z_step
                    if cz < 0 or cz >= map_size.z // chunk_size:
                        return False
                    c = chunks[cx, cy, cz]
                    
                t = t_dist_z
                t_dist_z += inv_z
                hit_axis = 'Z'
        else:
            if t_dist_x < t_dist_z:
                # Advance X
                i += x_step
                access += x_access_inc
                
                if i == x_comparison:
                    i = x_reset
                    access += x_access_reset
                    cx += x_step
                    if cx < 0 or cx >= map_size.x // chunk_size:
                        return False
                    c = chunks[cx, cy, cz]
                    
                t = t_dist_x
                t_dist_x += inv_x
                hit_axis = 'X'
            else:
                # Advance Z
                k += z_step
                access += z_access_inc
                
                if k == z_comparison:
                    k = z_reset
                    access += z_access_reset
                    cz += z_step
                    if cz < 0 or cz >= map_size.z // chunk_size:
                        return False
                    c = chunks[cx, cy, cz]
                    
                t = t_dist_z
                t_dist_z += inv_z
                hit_axis = 'Z'
                
    return False
```

### Edge Cases and Pitfalls
*   **Self-Intersection:** When computing normals or shadow rays, offset the origin by $n \cdot EPS$ to avoid hitting the surface itself [S1, unofficial].
*   **Null Chunks:** Empty chunks may be represented by `null` references. The algorithm must check for null before accessing `data` [S2, S3, unofficial].
*   **Boundary Precision:** Using `floor` on the start position is standard, but floating-point inaccuracies at chunk boundaries can cause skipping or double-checking. The `EPS` offset helps mitigate this [S1, unofficial].
*   **Zero Velocity:** If a velocity component is 0, the distance to the next boundary is infinity, preventing advancement on that axis [S2, S3, unofficial].

## Subtopics

### Incremental 2x2x2 Cell Block Ray Walking
Optimizes ray marching through a voxel grid by avoiding redundant cell checks when traversing a 2x2x2 block [S5, unofficial]. It steps a 2x2x2 block of grid cells; after stepping along one axis, half of the block was tested in the previous step, so it tests only the 4 cells on the leading side [S5, unofficial]. This reduces the number of grid-walk calls and improves cache locality [S5, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of voxels traversed. The cost per step is very low due to bitwise operations for chunk indexing [G].
*   **Memory Complexity:** Depends on the sparse voxel grid implementation. Chunks are allocated on-demand, reducing memory usage for empty worlds [G].
*   **Performance:** On a Ryzen 5 1600 CPU, ray marching for 1-10 blocks takes ~250 nanoseconds, allowing 64,000 rays per frame per thread at 60fps. For longer rays (200-400 blocks), it takes ~3400 nanoseconds, allowing 4,700 rays per frame [S2, S3, unofficial].
*   **Strengths:** Extremely fast for CPU-based collision detection in voxel games. Minimizes cache misses by keeping access local to the current chunk [S2, S3, unofficial].
*   **Limitations:** Fixed-step nature means it may step through empty space if the grid is sparse and chunks are not hierarchical (unlike Octree-based methods). Not suitable for high-precision continuous collision detection without sub-stepping [G].

## Background
The method is based on "A Fast Voxel Traversal Algorithm for Ray Tracing" by John Amanatides and Andrew Woo [S2, S3, unofficial]. It was specifically optimized for CPU implementation in the game *Sector's Edge* by Vercidium to handle particle and projectile collisions efficiently [S2, S3, unofficial]. The optimization focuses on reducing the computational overhead of chunk boundary checks by maintaining a "current working chunk" reference [S2, S3, unofficial].

## Sources
[S1] AndrewB330/VoxelsRayTracingTutorial, https://github.com/AndrewB330/VoxelsRayTracingTutorial, fetched 2026-10-24 (unofficial)
[S2] Optimised CPU Ray Marching, https://vercidium.com/blog/optimised-voxel-raymarching/, fetched 2026-10-24 (unofficial)
[S3] Vercidium/voxel-ray-marching, https://github.com/Vercidium/voxel-ray-marching, fetched 2026-10-24 (unofficial)
[S4] voxel-gi-particles: ray-traced lightbox about twice as fast, same image, https://github.com/Siroko/kansei/pull/114, fetched 2026-10-26 (unofficial)
