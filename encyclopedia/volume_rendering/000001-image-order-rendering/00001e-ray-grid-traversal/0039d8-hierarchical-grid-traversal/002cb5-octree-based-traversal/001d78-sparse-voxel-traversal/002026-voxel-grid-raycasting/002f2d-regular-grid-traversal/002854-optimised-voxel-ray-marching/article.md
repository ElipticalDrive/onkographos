# Optimised Voxel Ray Marching

Also called: Chunk-based Voxel Traversal, Fixed-step voxel ray marching, Optimised CPU Ray Marching, Sector's Edge, Voxel Volume Ray Tracing, VoxelsRayTracingTutorial

## Overview
Optimised Voxel Ray Marching is a ray tracing algorithm for voxel-based scenes that uses fixed-length steps along the ray to traverse a 3D grid of voxels, checking for occupancy at each step [S1, S2, S3, unofficial]. It is optimized for CPU collision detection by keeping block lookups within the current working chunk to avoid repeated chunk boundary calculations [S2, S3, unofficial]. Unlike the parent [Regular Grid Traversal](5939-regular-grid-traversal.md) which traverses the entire grid uniformly, this method partitions the world into fixed-size chunks, reducing the computational overhead of coordinate-to-index conversions during traversal [S2, S3, unofficial].

## How it works
The method implements two distinct variants found in the sources: a GPU-oriented fixed-step traversal (Sector's Edge / VoxelsRayTracingTutorial) and a CPU-oriented chunk-local traversal (Sector's Edge / Vercidium).

### Inputs and Outputs
**Inputs:**
*   **Ray Origin/Direction:** Defined in world or voxel space.
*   **World Data:** A 3D grid of voxels, either as a flat buffer (GPU variant) or a 3D array of chunks (CPU variant).
*   **Parameters:** `EPS` (small constant for offsetting), `CHUNK_SIZE` (typically 32), `SHIFT` (typically 5, $\log_2(CHUNK\_SIZE)$), `MASK` (typically 0x1f, $CHUNK\_SIZE - 1$).

**Outputs:**
*   **Hit Status:** Boolean indicating if a solid voxel was encountered.
*   **Hit Info:** Position, color, cell coordinates, and local normal (GPU variant); or Axis of collision (CPU variant).

### Data Structures

**GPU Variant (VoxelsRayTracingTutorial):**
*   **VolumeData:** A 1D buffer of `uint` values representing a 3D grid. Index for voxel $(x, y, z)$ is $x \cdot size.y \cdot size.z + y \cdot size.z + z$. Each `uint` stores an RGBA color (e.g., `0xff6b21`) [S1, unofficial].
*   **Size:** `UVec3` uniform defining grid dimensions [S1, unofficial].

**CPU Variant (Sector's Edge / Vercidium):**
*   **Map:** A 3D array `Chunk[,,] chunks` indexed by `[chunkIndexX, chunkIndexY, chunkIndexZ]` [S2, S3, unofficial].
*   **Chunk:** Contains a 1D array `Block[] data` of size $CHUNK\_SIZE^3$ (32,768 for size 32) [S2, S3, unofficial].
*   **Block:** A struct with a single byte field `byte kind` (0 for empty) [S2, S3, unofficial].
*   **Constants:** $CHUNK\_SIZE=32$, $SHIFT=5$, $MASK=0x1f$ [S2, S3, unofficial].

### Algorithm: GPU Fixed-Step Traversal
This variant uses a fixed number of iterations and calculates step sizes based on ray direction [S1, unofficial].

1.  **Initialize:** Compute ray origin in voxel space and normalize direction. Calculate `direction_inv = 1.0 / direction`.
2.  **Offset:** Offset origin by `direction * EPS` to avoid self-intersection [S1, unofficial].
3.  **Setup:** Set `cell = ivec3(floor(origin))`, `time = vec3(0.0)`, `hit = false`.
4.  **Loop:** Iterate up to 256 times (or until boundary check fails):
    *   Check `getVoxel(cell)`. If non-zero, set `hit = true` and break.
    *   Calculate `next_cell_delta = 0.5 + sign(direction) * (0.5 + EPS)` [S1, unofficial].
    *   Calculate time to next boundary: `time = (cell + next_cell_delta - origin) * direction_inv`.
    *   Advance origin: `origin += direction * min(time.x, time.y, time.z)`.
    *   Update cell: `cell = ivec3(floor(origin))`.
5.  **Post-Process:** If hit, extract color from `getVoxel(cell)` and compute local normal from `time` and `direction` [S1, unofficial].

### Algorithm: CPU Chunk-Local Traversal
This variant maintains a reference to the current chunk and updates it only when crossing chunk boundaries, minimizing bit-shifts and masks [S2, S3, unofficial].

1.  **Initialize:** Convert start position to integer coordinates $(x, y, z)$. Check if outside map bounds; if so, return `hit = false` [S2, S3, unofficial].
2.  **Chunk Setup:**
    *   Calculate chunk indices: `chunkIndexX = x >> SHIFT`, etc.
    *   Get reference to current chunk `c = chunks[chunkIndexX, chunkIndexY, chunkIndexZ]`.
    *   Calculate chunk-relative coordinates: `i = x & MASK`, `j = y & MASK`, `k = z & MASK`.
    *   Calculate linear access index: `access = j + i * CHUNK_SIZE + k * CHUNK_SIZE_SQUARED` [S2, S3, unofficial].
3.  **Edge Case:** If start and end positions are in the same voxel, check occupancy directly and return [S2, S3, unofficial].
4.  **Direction Variables:** Initialize step sizes (`iStep`, `jStep`, `kStep`), comparison values (`iComparison`, etc.), reset values (`iReset`, etc.), and access resets (`xAccessReset`, etc.) based on the sign of the velocity components [S2, S3, unofficial].
5.  **Distance Calculation:**
    *   Normalize velocity. Calculate inverted components: $xInverted = |1/v_x|$, etc.
    *   Calculate initial distance to next boundary in $t$-units: $xDistance = xInverted \cdot xDist$, where $xDist$ is distance to the next voxel boundary in map units [S2, S3, unofficial]. If a velocity component is 0, distance is $\infty$.
6.  **March Loop:** While progress $t \le max$:
    *   Check occupancy: If `c != null` and `c.data[access].kind != 0`, set `hit = true` and return [S2, S3, unofficial].
    *   Find closest boundary: Compare $xDistance, yDistance, zDistance$.
    *   **Advance Axis:**
        *   Increment chunk-relative coordinate (e.g., `j += jStep`).
        *   Update access index (e.g., `access += jStep`).
        *   **Chunk Exit Check:** If coordinate reaches boundary (e.g., `j == jComparison`):
            *   Reset coordinate (e.g., `j = jReset`).
            *   Update access index (e.g., `access += yAccessReset`).
            *   Increment chunk index (e.g., `chunkIndexY += jStep`).
            *   If new chunk index out of bounds, return `hit = false`.
            *   Load new chunk `c = chunks[...]`.
        *   Update distance: $yDistance += yInverted$.
        *   Update progress: $t = yDistance$.
7.  **Completion:** If loop finishes without hit, return `hit = false` [S2, S3, unofficial].

### Pseudocode (CPU Variant)
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_march(start: Vector3, velocity: Vector3, max_dist: float) -> HitResult:
    # 1. Initialize integer coordinates
    x = int(start.x)
    y = int(start.y)
    z = int(start.z)
    
    # 2. Boundary check
    if not (0 <= x < MAP_SIZE_X and 0 <= y < MAP_SIZE_Y and 0 <= z < MAP_SIZE_Z):
        return HitResult(hit=False)
        
    # 3. Chunk setup
    chunk_idx_x = x >> SHIFT
    chunk_idx_y = y >> SHIFT
    chunk_idx_z = z >> SHIFT
    
    chunk = chunks[chunk_idx_x, chunk_idx_y, chunk_idx_z]
    
    # 4. Chunk-relative coords and access index
    i = x & MASK
    j = y & MASK
    k = z & MASK
    access = j + i * CHUNK_SIZE + k * (CHUNK_SIZE ** 2)
    
    # 5. Edge case: same voxel
    end = start + velocity
    if x == int(end.x) and y == int(end.y) and z == int(end.z):
        if chunk is None or chunk.data[access].kind == 0:
            return HitResult(hit=False)
        return HitResult(hit=True, axis=Axis.None)
        
    # 6. Direction-dependent variables
    # Simplified for positive Y direction as example; logic mirrors for X/Z
    j_step = 1
    j_comparison = CHUNK_SIZE
    j_reset = 0
    y_access_reset = -CHUNK_SIZE
    y_dist = y - start.y + 1
    
    # Normalize velocity and calculate inverses
    velocity = velocity.normalize()
    x_inv = abs(1.0 / velocity.x) if velocity.x != 0 else float('inf')
    y_inv = abs(1.0 / velocity.y) if velocity.y != 0 else float('inf')
    z_inv = abs(1.0 / velocity.z) if velocity.z != 0 else float('inf')
    
    # Initial distances in t-units
    x_dist = x_inv * (x - start.x + 1) if velocity.x != 0 else float('inf')
    y_dist = y_inv * y_dist
    z_dist = z_inv * (z - start.z + 1) if velocity.z != 0 else float('inf')
    
    t = 0.0
    
    # 7. March loop
    while t <= max_dist:
        # Check occupancy
        if chunk is not None and chunk.data[access].kind != 0:
            return HitResult(hit=True, axis=Axis.Y) # Axis determined by last advance
            
        # Find closest boundary
        if y_dist < x_dist and y_dist < z_dist:
            # Advance Y
            j += j_step
            access += j_step
            
            # Check chunk exit
            if j == j_comparison:
                j = j_reset
                access += y_access_reset
                chunk_idx_y += j_step
                
                if not (0 <= chunk_idx_y < CHUNK_AMOUNT_Y):
                    return HitResult(hit=False)
                chunk = chunks[chunk_idx_x, chunk_idx_y, chunk_idx_z]
                
            y_dist += y_inv
            t = y_dist
        elif x_dist < z_dist:
            # Advance X (logic mirrored)
            # ...
            pass
        else:
            # Advance Z (logic mirrored)
            # ...
            pass
            
    return HitResult(hit=False)
```

### Lighting and AO (GPU Variant)
*   **Lighting:** Diffuse lighting calculated with `LIGHT_DIR = (0.1, -0.5, 0.3)`. Shadow rays are cast from `info.pos + normal * EPS` to check occlusion [S1, unofficial].
*   **Ambient Occlusion:** `computeAo` checks neighbors based on `local_normal`. It calculates UVs from position and mask, determines side/corner occupancy, and interpolates ambient values [S1, unofficial].

### Parameters
*   `EPS`: $5 \times 10^{-5}$ [S1, unofficial].
*   `CHUNK_SIZE`: 32 [S2, S3, unofficial].
*   `SHIFT`: 5 [S2, S3, unofficial].
*   `MASK`: 0x1f (31) [S2, S3, unofficial].
*   Max Iterations (GPU): 256 [S1, unofficial].

### Pitfalls
*   **Self-Intersection:** Shadow rays must be offset by `normal * EPS` [S1, unofficial].
*   **Boundary Access:** Ensure voxel coordinates are within `[0, size)` to prevent out-of-bounds access [S1, unofficial].
*   **Chunk Bounds:** Check if new chunk indices are within `CHUNK_AMOUNT` limits [S2, S3, unofficial].
*   **Null Chunks:** Handle null chunk references (empty chunks) gracefully [S2, S3, unofficial].

## Operating characteristics
*   **Performance:** On a Ryzen 5 1600 CPU, ray marching for 1-10 blocks takes 250 nanoseconds, allowing 64,000 rays per 16ms frame. For 200-400 blocks, it takes 3,400 nanoseconds, allowing 4,700 rays per frame [S2, S3, unofficial].
*   **Complexity:** Time complexity is proportional to ray length in voxels. Memory complexity is $O(N)$ for the voxel data, with chunking reducing cache misses for localized access [S2, S3, unofficial].
*   **Strengths:** High performance on CPU for collision detection; reduced overhead from chunk-local lookups [S2, S3, unofficial].
*   **Limitations:** Fixed chunk size may lead to fragmentation; not suitable for highly dynamic large-scale scenes without chunk management [S2, unofficial].

## Background
The method is based on "A Fast Voxel Traversal Algorithm for Ray Tracing" by John Amanatides and Andrew Woo [S2, S3, unofficial]. It was optimized for the game *Sector's Edge* by keeping block lookups within the current working chunk to minimize bitshifts, masks, and multiplication operations [S2, S3, unofficial]. It is used for particle and projectile collision detection in voxel worlds [S2, S3, unofficial].

## Sources
[S1] VoxelsRayTracingTutorial, https://github.com/AndrewB330/VoxelsRayTracingTutorial, fetched 2026-10-24 (unofficial)
[S2] Optimised CPU Ray Marching, https://vercidium.com/blog/optimised-voxel-raymarching/, fetched 2026-10-24 (unofficial)
[S3] Vercidium/voxel-ray-marching, https://github.com/Vercidium/voxel-ray-marching, fetched 2026-10-24 (unofficial)
