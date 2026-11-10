# Optimised Voxel Ray Marching

Also called: Chunk-based Voxel Traversal, Fixed-step voxel ray marching, Optimised CPU Ray Marching, Sector's Edge, Voxel Volume Ray Tracing, VoxelsRayTracingTutorial

## Overview
Optimised Voxel Ray Marching is a fixed-step ray traversal algorithm for voxel-based scenes that accelerates rendering and collision detection by minimizing computational overhead during block lookups [S1, S2, S3, unofficial]. It builds on [Regular Grid Traversal](5939-regular-grid-traversal.md) by organizing the voxel world into fixed-size chunks and maintaining a reference to the "current working chunk" throughout the march, thereby avoiding repeated bit-shifts, masks, and multiplication operations required to map global coordinates to chunk-relative indices [S2, S3, unofficial]. Unlike pure octree traversal, it avoids pointer chasing and dynamic node allocation, and unlike flat grid marching, it reduces the number of visited cells in sparse scenes by skipping empty chunks [S1, unofficial].

## How it works
The method implements a fixed-step traversal loop that advances the ray by the distance to the nearest voxel boundary, checking occupancy within the current chunk. It optimizes the Amanatides-Woo algorithm by caching the chunk reference and updating it only when the ray crosses a chunk boundary [S2, S3, unofficial].

### Inputs and Outputs
**Inputs:**
*   `start`: Ray origin in world/voxel space ($vec3$).
*   `velocity`: Normalized ray direction vector ($vec3$).
*   `max`: Maximum ray length in world units ($double$).
*   `chunks`: 3D array of chunk references ($Chunk[,,]$).
*   `MAP_SIZE`: World dimensions ($x=512, y=160, z=512$) [S3, unofficial].
*   `CHUNK_SIZE`: Side length of a chunk in blocks ($32$) [S3, unofficial].

**Outputs:**
*   `hit`: Boolean indicating if a solid block was encountered.
*   `axis`: The axis ($X, Y, Z$) of the face hit (used for reflection) [S2, S3, unofficial].

### Data Structures
**Chunk:**
*   Contains a 1D array `data` of size $CHUNK\_SIZE^3$ ($32768$) [S2, S3, unofficial].
*   `data` stores `Block` structs.

**Block:**
*   Struct with a single byte field `kind` [S2, S3, unofficial].
*   `kind == 0` indicates empty space; `kind != 0` indicates solid [S2, S3, unofficial].

**Map:**
*   3D array `chunks[,,]` indexed by $[chunkIndexX, chunkIndexY, chunkIndexZ]$ [S2, S3, unofficial].
*   Dimensions: $CHUNK\_AMOUNT\_X=16, CHUNK\_AMOUNT\_Y=5, CHUNK\_AMOUNT\_Z=16$ [S3, unofficial].

**Constants:**
*   $SHIFT = 5$ (since $2^5 = 32$) [S2, S3, unofficial].
*   $MASK = 0x1f$ (31, equivalent to $CHUNK\_SIZE - 1$) [S2, S3, unofficial].

### Algorithm Steps
1.  **Initialization:**
    *   Convert `start` to integer voxel coordinates: $x = \lfloor start.x \rfloor, y = \lfloor start.y \rfloor, z = \lfloor start.z \rfloor$ [S2, S3, unofficial].
    *   Check bounds: If $x, y, z$ are outside $[0, MAP\_SIZE)$, set `hit = false` and return [S2, S3, unofficial].
    *   Calculate chunk indices: $chunkIndexX = x \gg SHIFT$, $chunkIndexY = y \gg SHIFT$, $chunkIndexZ = z \gg SHIFT$ [S2, S3, unofficial].
    *   Get current chunk reference: $c = chunks[chunkIndexX, chunkIndexY, chunkIndexZ]$ [S2, S3, unofficial].
    *   Calculate chunk-relative coordinates: $i = x \& MASK$, $j = y \& MASK$, $k = z \& MASK$ [S2, S3, unofficial].
    *   Calculate initial linear access index in chunk: $access = j + i \times CHUNK\_SIZE + k \times CHUNK\_SIZE\_SQUARED$ [S2, S3, unofficial].
    *   Handle zero-length ray: If $x == \lfloor end.x \rfloor$ and $y == \lfloor end.y \rfloor$ and $z == \lfloor end.z \rfloor$ (where $end = start + velocity \times max$), check if $c$ is null or $c.data[access].kind == 0$. If so, `hit = false`; else `hit = true`. Return [S2, S3, unofficial].

2.  **Direction Setup:**
    *   Initialize direction-dependent variables for each axis based on the sign of `velocity` components [S2, S3, unofficial]:
        *   **Y Axis:**
            *   If $velocity.y > 0$: $jStep = 1, jComparison = CHUNK\_SIZE, jReset = 0, yAccessReset = -CHUNK\_SIZE, yDist = y - start.y + 1$.
            *   Else: $jStep = -1, jComparison = -1, jReset = CHUNK\_SIZE - 1, yAccessReset = CHUNK\_SIZE, yDist = start.y - y$.
        *   **X Axis:**
            *   If $velocity.x > 0$: $iStep = 1, iComparison = CHUNK\_SIZE, iReset = 0, xAccessIncrement = CHUNK\_SIZE, xAccessReset = -CHUNK\_SIZE\_SQUARED, xDist = x - start.x + 1$.
            *   Else: $iStep = -1, iComparison = -1, iReset = CHUNK\_SIZE - 1, xAccessIncrement = -CHUNK\_SIZE, xAccessReset = CHUNK\_SIZE\_SQUARED, xDist = start.x - x$.
        *   **Z Axis:**
            *   If $velocity.z > 0$: $kStep = 1, kComparison = CHUNK\_SIZE, kReset = 0, zAccessIncrement = CHUNK\_SIZE\_SQUARED, zAccessReset = -CHUNK\_SIZE\_CUBED, zDist = z - start.z + 1$.
            *   Else: $kStep = -1, kComparison = -1, kReset = CHUNK\_SIZE - 1, zAccessIncrement = -CHUNK\_SIZE\_SQUARED, zAccessReset = CHUNK\_SIZE\_CUBED, zDist = start.z - z$.
    *   Normalize `velocity` [S2, S3, unofficial].
    *   Calculate inverse absolute velocity components: $xInverted = |1/velocity.x|$, $yInverted = |1/velocity.y|$, $zInverted = |1/velocity.z|$ [S2, S3, unofficial].
    *   Calculate initial distance to next boundary in $t$-units:
        *   If $velocity.axis == 0$, distance is $\infty$.
        *   Else, $xDistance = xInverted \times xDist$, $yDistance = yInverted \times yDist$, $zDistance = zInverted \times zDist$ [S2, S3, unofficial].
    *   Initialize progress $t = 0.0$ [S2, S3, unofficial].

3.  **Traversal Loop:**
    *   While $t \le max$:
        *   **Occupancy Check:** If $c$ is not null and $c.data[access].kind \neq 0$, set `hit = true`, `axis` to the axis of the last boundary crossed, and return [S2, S3, unofficial].
        *   **Find Closest Boundary:**
            *   If $yDistance < xDistance$:
                *   If $yDistance < zDistance$:
                    *   Advance Y: $j += jStep$, $access += jStep$ [S2, S3, unofficial].
                    *   Check Chunk Exit: If $j == jComparison$:
                        *   Reset $j = jReset$ [S2, S3, unofficial].
                        *   Update $access += yAccessReset$ [S2, S3, unofficial].
                        *   Increment $chunkIndexY += jStep$ [S2, S3, unofficial].
                        *   If $chunkIndexY$ out of bounds ($<0$ or $\ge CHUNK\_AMOUNT\_Y$), set `hit = false` and return [S2, S3, unofficial].
                        *   Load new chunk: $c = chunks[chunkIndexX, chunkIndexY, chunkIndexZ]$ [S2, S3, unofficial].
                    *   Update Progress: $t = yDistance$, $yDistance += yInverted$, $axis = Y$ [S2, S3, unofficial].
                *   Else ($zDistance \le yDistance$):
                    *   Advance Z: $k += kStep$, $access += zAccessIncrement$ [S2, S3, unofficial].
                    *   Check Chunk Exit: If $k == kComparison$:
                        *   Reset $k = kReset$ [S2, S3, unofficial].
                        *   Update $access += zAccessReset$ [S2, S3, unofficial].
                        *   Increment $chunkIndexZ += kStep$ [S2, S3, unofficial].
                        *   If $chunkIndexZ$ out of bounds, set `hit = false` and return [S2, S3, unofficial].
                        *   Load new chunk: $c = chunks[chunkIndexX, chunkIndexY, chunkIndexZ]$ [S2, S3, unofficial].
                    *   Update Progress: $t = zDistance$, $zDistance += zInverted$, $axis = Z$ [S2, S3, unofficial].
            *   Else ($xDistance \le yDistance$):
                *   If $xDistance < zDistance$:
                    *   Advance X: $i += iStep$, $access += xAccessIncrement$ [S2, S3, unofficial].
                    *   Check Chunk Exit: If $i == iComparison$:
                        *   Reset $i = iReset$ [S2, S3, unofficial].
                        *   Update $access += xAccessReset$ [S2, S3, unofficial].
                        *   Increment $chunkIndexX += iStep$ [S2, S3, unofficial].
                        *   If $chunkIndexX$ out of bounds, set `hit = false` and return [S2, S3, unofficial].
                        *   Load new chunk: $c = chunks[chunkIndexX, chunkIndexY, chunkIndexZ]$ [S2, S3, unofficial].
                    *   Update Progress: $t = xDistance$, $xDistance += xInverted$, $axis = X$ [S2, S3, unofficial].
                *   Else ($zDistance \le xDistance$):
                    *   Advance Z: $k += kStep$, $access += zAccessIncrement$ [S2, S3, unofficial].
                    *   Check Chunk Exit: If $k == kComparison$:
                        *   Reset $k = kReset$ [S2, S3, unofficial].
                        *   Update $access += zAccessReset$ [S2, S3, unofficial].
                        *   Increment $chunkIndexZ += kStep$ [S2, S3, unofficial].
                        *   If $chunkIndexZ$ out of bounds, set `hit = false` and return [S2, S3, unofficial].
                        *   Load new chunk: $c = chunks[chunkIndexX, chunkIndexY, chunkIndexZ]$ [S2, S3, unofficial].
                    *   Update Progress: $t = zDistance$, $zDistance += zInverted$, $axis = Z$ [S2, S3, unofficial].

4.  **Completion:**
    *   If the loop finishes without hitting a block, set `hit = false` [S2, S3, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_march(start: Vec3, velocity: Vec3, max_dist: float, chunks: Array3D[Chunk], map_size: Vec3i, chunk_size: int) -> Tuple[bool, Axis]:
    # Constants
    SHIFT = 5
    MASK = 0x1f
    CHUNK_SIZE_SQUARED = chunk_size * chunk_size
    CHUNK_SIZE_CUBED = chunk_size * chunk_size * chunk_size
    
    # 1. Initialization
    x = floor(start.x)
    y = floor(start.y)
    z = floor(start.z)
    
    # Bounds check
    if x < 0 or x >= map_size.x or y < 0 or y >= map_size.y or z < 0 or z >= map_size.z:
        return False, Axis.None
        
    # Chunk indices
    chunk_idx_x = x >> SHIFT
    chunk_idx_y = y >> SHIFT
    chunk_idx_z = z >> SHIFT
    
    # Get current chunk
    c = chunks[chunk_idx_x, chunk_idx_y, chunk_idx_z]
    
    # Chunk-relative coords
    i = x & MASK
    j = y & MASK
    k = z & MASK
    
    # Linear access index
    access = j + i * chunk_size + k * CHUNK_SIZE_SQUARED
    
    # End position
    end = start + velocity * max_dist
    end_x = floor(end.x)
    end_y = floor(end.y)
    end_z = floor(end.z)
    
    # Zero-length ray check
    if x == end_x and y == end_y and z == end_z:
        if c is None or c.data[access].kind == 0:
            return False, Axis.None
        return True, Axis.None # Hit same block
        
    # 2. Direction Setup
    # Initialize direction variables
    # Y Axis
    if velocity.y > 0:
        j_step = 1
        j_comparison = chunk_size
        j_reset = 0
        y_access_reset = -chunk_size
        y_dist = y - start.y + 1
    else:
        j_step = -1
        j_comparison = -1
        j_reset = chunk_size - 1
        y_access_reset = chunk_size
        y_dist = start.y - y
        
    # X Axis
    if velocity.x > 0:
        i_step = 1
        i_comparison = chunk_size
        i_reset = 0
        x_access_increment = chunk_size
        x_access_reset = -CHUNK_SIZE_SQUARED
        x_dist = x - start.x + 1
    else:
        i_step = -1
        i_comparison = -1
        i_reset = chunk_size - 1
        x_access_increment = -chunk_size
        x_access_reset = CHUNK_SIZE_SQUARED
        x_dist = start.x - x
        
    # Z Axis
    if velocity.z > 0:
        k_step = 1
        k_comparison = chunk_size
        k_reset = 0
        z_access_increment = CHUNK_SIZE_SQUARED
        z_access_reset = -CHUNK_SIZE_CUBED
        z_dist = z - start.z + 1
    else:
        k_step = -1
        k_comparison = -1
        k_reset = chunk_size - 1
        z_access_increment = -CHUNK_SIZE_SQUARED
        z_access_reset = CHUNK_SIZE_CUBED
        z_dist = start.z - z
        
    # Normalize velocity and calculate inverses
    velocity = normalize(velocity)
    x_inv = abs(1.0 / velocity.x) if velocity.x != 0 else float('inf')
    y_inv = abs(1.0 / velocity.y) if velocity.y != 0 else float('inf')
    z_inv = abs(1.0 / velocity.z) if velocity.z != 0 else float('inf')
    
    # Initial distances in t-units
    x_dist_t = x_inv * x_dist if velocity.x != 0 else float('inf')
    y_dist_t = y_inv * y_dist if velocity.y != 0 else float('inf')
    z_dist_t = z_inv * z_dist if velocity.z != 0 else float('inf')
    
    t = 0.0
    axis = Axis.None
    
    # 3. Traversal Loop
    while t <= max_dist:
        # Occupancy check
        if c is not None and c.data[access].kind != 0:
            return True, axis
            
        # Find closest boundary
        if y_dist_t < x_dist_t:
            if y_dist_t < z_dist_t:
                # Advance Y
                j += j_step
                access += j_step
                
                # Check chunk exit
                if j == j_comparison:
                    j = j_reset
                    access += y_access_reset
                    chunk_idx_y += j_step
                    
                    if chunk_idx_y < 0 or chunk_idx_y >= chunks.shape[1]:
                        return False, Axis.None
                    c = chunks[chunk_idx_x, chunk_idx_y, chunk_idx_z]
                    
                t = y_dist_t
                y_dist_t += y_inv
                axis = Axis.Y
            else:
                # Advance Z
                k += k_step
                access += z_access_increment
                
                # Check chunk exit
                if k == k_comparison:
                    k = k_reset
                    access += z_access_reset
                    chunk_idx_z += k_step
                    
                    if chunk_idx_z < 0 or chunk_idx_z >= chunks.shape[2]:
                        return False, Axis.None
                    c = chunks[chunk_idx_x, chunk_idx_y, chunk_idx_z]
                    
                t = z_dist_t
                z_dist_t += z_inv
                axis = Axis.Z
        else:
            if x_dist_t < z_dist_t:
                # Advance X
                i += i_step
                access += x_access_increment
                
                # Check chunk exit
                if i == i_comparison:
                    i = i_reset
                    access += x_access_reset
                    chunk_idx_x += i_step
                    
                    if chunk_idx_x < 0 or chunk_idx_x >= chunks.shape[0]:
                        return False, Axis.None
                    c = chunks[chunk_idx_x, chunk_idx_y, chunk_idx_z]
                    
                t = x_dist_t
                x_dist_t += x_inv
                axis = Axis.X
            else:
                # Advance Z
                k += k_step
                access += z_access_increment
                
                # Check chunk exit
                if k == k_comparison:
                    k = k_reset
                    access += z_access_reset
                    chunk_idx_z += k_step
                    
                    if chunk_idx_z < 0 or chunk_idx_z >= chunks.shape[2]:
                        return False, Axis.None
                    c = chunks[chunk_idx_x, chunk_idx_y, chunk_idx_z]
                    
                t = z_dist_t
                z_dist_t += z_inv
                axis = Axis.Z
                
    return False, Axis.None
```

### Parameters
*   $EPS = 5e-5$: Used in the shader-based variant [S1, unofficial] to offset ray origins and shadow rays to prevent self-intersection. Not used in the CPU collision variant [S2, S3, unofficial].
*   $CHUNK\_SIZE = 32$: Fixed chunk dimension [S3, unofficial].
*   $SHIFT = 5$: Bit shift for chunk indexing [S3, unofficial].
*   $MASK = 0x1f$: Bitmask for chunk-relative indexing [S3, unofficial].
*   $MAP\_SIZE = (512, 160, 512)$: World dimensions in the reference implementation [S3, unofficial].

### Edge Cases and Pitfalls
*   **Self-Intersection:** In rendering contexts, shadow rays must be offset by $n \times EPS$ to avoid hitting the surface they originated from [S1, unofficial].
*   **Null Chunks:** Empty chunks may be represented as `null` references. The algorithm must check for `null` before accessing `data` [S2, S3, unofficial].
*   **Boundary Conditions:** The algorithm explicitly checks if chunk indices go out of bounds ($<0$ or $\ge CHUNK\_AMOUNT$) upon crossing a chunk boundary [S2, S3, unofficial].
*   **Zero Velocity:** If a velocity component is 0, the distance to the next boundary in that axis is $\infty$, effectively skipping traversal in that axis [S2, S3, unofficial].

## Subtopics

### [Chunked Voxel Traversal](8311-chunked-voxel-traversal.md)
A hierarchical method using a two-level 3D grid to skip empty space. It avoids pointer chasing of octrees while reducing visited cells compared to flat grids. Choose when rendering sparse scenes on CPU where fixed-step traversal is preferred over pointer-based traversal.

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of voxels traversed. The optimization reduces the constant factor by minimizing bit-shifts and multiplications per step [S2, S3, unofficial].
*   **Memory Complexity:** $O(C)$ for the chunk array, where $C$ is the number of chunks. Chunks can be null if empty, saving memory [S2, unofficial].
*   **Performance:** On a Ryzen 5 1600 CPU, short rays (1-10 blocks) take ~250 ns, allowing 64,000 rays/frame/thread at 60fps. Long rays (200-400 blocks) take ~3400 ns, allowing 4,700 rays/frame/thread [S2, S3, unofficial].
*   **Strengths:** Extremely fast for CPU-based collision detection and simple voxel rendering. Minimal branching compared to octrees. No dynamic allocation during traversal [S2, S3, unofficial].
*   **Limitations:** Fixed chunk size may lead to memory waste if the world is very large and sparse. Not suitable for hierarchical rendering where large empty regions need to be skipped entirely (use [Chunked Voxel Traversal](8311-chunked-voxel-traversal.md) or [Octree-based traversal](5940-octree-based-traversal.md) instead).

## Background
The algorithm is based on "A Fast Voxel Traversal Algorithm for Ray Tracing" by John Amanatides and Andrew Woo [S2, S3, unofficial]. It was optimized for the game *Sector's Edge* by keeping block lookups within the current working chunk to minimize bit-shifts, masks, and multiplication operations [S2, S3, unofficial]. It is used in *Sector's Edge* for particle and projectile collision detection [S3, unofficial].

## Sources
[S1] AndrewB330/VoxelsRayTracingTutorial, https://github.com/AndrewB330/VoxelsRayTracingTutorial, fetched 2026-10-24 (unofficial)
[S2] Optimised CPU Ray Marching, https://vercidium.com/blog/optimised-voxel-raymarching/, fetched 2026-10-24 (unofficial)
[S3] Vercidium/voxel-ray-marching, https://github.com/Vercidium/voxel-ray-marching, fetched 2026-10-24 (unofficial)
