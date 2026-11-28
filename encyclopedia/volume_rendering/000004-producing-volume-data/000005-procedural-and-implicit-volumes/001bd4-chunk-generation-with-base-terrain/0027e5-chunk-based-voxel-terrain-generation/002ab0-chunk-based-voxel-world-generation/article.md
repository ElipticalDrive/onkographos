# Chunk-based Voxel World Generation

Also called: Density-based voxel placement, Simplex noise terrain generation

## Overview
Chunk-based Voxel World Generation is a procedural technique that constructs a 3D voxel world by sampling 3D noise functions to determine block density and height, then filtering these values against thresholds to place solid blocks [S1, unofficial]. It differs from static generation by organizing the output into fixed-size chunks, enabling efficient memory management and localized updates, and from pure implicit volumes by explicitly discretizing the space into a grid of binary or typed voxels rather than a continuous field [G].

## How it works
**Inputs and Outputs**
*   **Inputs:** World dimensions (`worldSize`), chunk dimensions (`chunkSize`), block physical size (`blockSize`), water level threshold (`waterlevel`), density cutoff threshold (`density_threshold`), noise scale divisors (`noise_scale_x`, `noise_scale_y`), and a random seed for noise initialization [S1, unofficial].
*   **Outputs:** A collection of chunks, where each chunk contains a list of active block coordinates and types [S1, unofficial].

**Data Structures**
*   **Chunk:** A container holding a list of active blocks. In some implementations, chunks are indexed in a flat array or map (`chunkIdMap`) to avoid sparse allocation overhead [S1, unofficial].
*   **Block:** Defined by its 3D integer coordinates $(x, y, z)$ and an optional type identifier [S1, unofficial].
*   **Voxel Array:** A 3D array `self.voxels` of size `[WORLD_VOL, CHUNK_VOL]` of type `uint8` may be used to store block IDs directly, indexed by chunk and local position [S2, unofficial].

**Parameters**
*   `worldSize`: 192 (blocks) [S1, unofficial].
*   `chunkSize`: 16 (blocks per dimension) [S1, unofficial].
*   `blockSize`: 0.1 (world units per block) [S1, unofficial].
*   `waterlevel`: 0 (Y-coordinate threshold) [S1, unofficial].
*   `density_threshold`: 0.5 (minimum simplex noise value) [S1, unofficial].
*   `noise_scale_x`: 100 (X divisor for height noise) [S1, unofficial].
*   `noise_scale_y`: 100 (Y divisor for density noise) [S1, unofficial].

**Algorithm Steps**
1.  **Initialize:** Set random seed for the noise function and initialize chunk storage structures (`chunks`, `chunkIdMap`) [S1, unofficial].
2.  **Iterate World Grid:** Loop $x$ from $0$ to `worldSize`-1 and $z$ from $0$ to `worldSize`-1 [S1, unofficial].
3.  **Calculate Height:** Compute the surface height $H$ using 3D simplex noise:
    $$ H = \text{simplex3}\left(\frac{x}{\text{noise\_scale\_x}}, \frac{x}{200}, \frac{z}{\text{noise\_scale\_x}}\right) \times \text{worldSize} $$
    Note: The second coordinate uses a fixed divisor of 200 in the reference implementation, distinct from `noise_scale_x` [S1, unofficial].
4.  **Iterate Vertical Column:** Loop $y$ from $0$ to $\max(0, \lfloor H \rfloor - 1)$ to handle negative simplex noise returns [G].
5.  **Apply Water Level:** If $y \ge \text{waterlevel}$, proceed to density check; otherwise, skip [S1, unofficial].
6.  **Calculate Density:** Compute density $D$ using a separate 3D simplex noise evaluation:
    $$ D = \text{simplex3}\left(\frac{x}{\text{noise\_scale\_x}}, \frac{y}{\text{noise\_scale\_y}}, \frac{z}{\text{noise\_scale\_x}}\right) $$
7.  **Place Block:** If $D > \text{density\_threshold}$, create a block at $(x, y, z)$ and add it to the appropriate chunk [S1, unofficial].
8.  **Rebuild Chunks:** After iterating all world coordinates, iterate through the `chunkIdMap` and trigger a rebuild operation for each valid chunk to generate mesh data [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_world(world_size, chunk_size, water_level, density_threshold, noise_scale_x, noise_scale_y):
    # Initialize noise seed
    noise_state = random_seed()
    
    # Initialize storage
    chunks = {}
    chunk_id_map = []
    
    # Precompute chunk grid dimensions
    world_w = world_size // chunk_size
    world_area = world_w * world_w
    
    # Iterate over X and Z dimensions
    for x in range(world_size):
        for z in range(world_size):
            # Calculate height using simplex noise
            # Note: Second parameter uses fixed divisor 200 as per source [S1, unofficial]
            height_noise = simplex3(x / noise_scale_x, x / 200, z / noise_scale_x, noise_state)
            height = height_noise * world_size
            
            # Iterate Y from 0 to height, handling negative noise returns
            max_y = max(0, int(height))
            for y in range(max_y):
                # Apply water level threshold
                if y >= water_level:
                    # Calculate density using separate noise function
                    density_noise = simplex3(x / noise_scale_x, y / noise_scale_y, z / noise_scale_x, noise_state)
                    
                    # Place block if density exceeds threshold
                    if density_noise > density_threshold:
                        chunk_id = (x // chunk_size) + (y // chunk_size) * world_w + (z // chunk_size) * world_area
                        
                        if chunk_id not in chunks:
                            chunks[chunk_id] = []
                            chunk_id_map.append(chunk_id)
                            
                        chunks[chunk_id].append((x, y, z))
    
    # Rebuild meshes for all active chunks
    for chunk_id in chunk_id_map:
        # Mesh generation is implementation-specific [G]
        pass
```

**Edge Cases and Pitfalls**
*   **Precision:** Floating-point noise values must be compared carefully against thresholds; the threshold 0.5 is arbitrary and affects terrain porosity [S1, unofficial].
*   **Boundaries:** The loops are bounded by `worldSize`; infinite worlds require chunk-based generation with dynamic coordinate scaling [S2, unofficial].
*   **Negative Height:** Simplex noise returns values in [-1, 1], so `height` can be negative; the loop must clamp to 0 to prevent iteration errors [G].

## Subtopics

### [Chunk System (Dynamic Loading)](13544-chunk-system-dynamic-loading.md)
Manages the lifecycle of chunks based on camera position, enabling infinite worlds by loading/unloading data on demand.

### [Chunk-Based Voxel Terrain Generation](4140-chunk-based-voxel-terrain-generation.md)
The parent method, which adds octaved noise, biome logic, and run-length encoding to the basic chunk generation.

## Operating characteristics
*   **Time Complexity:** $O(W^2 \cdot H)$, where $W$ is `worldSize` and $H$ is the average height, as each voxel column is evaluated once [G].
*   **Memory Complexity:** Proportional to the number of placed blocks, stored in chunk containers or a 3D `uint8` array [S2, unofficial].
*   **Performance:** The reference implementation runs in JavaScript/WebGL for small worlds (192x192) [S1, unofficial].
*   **Strengths:** Simple to implement; deterministic with seed; thresholds control terrain porosity [S1, unofficial].
*   **Limitations:** Single-octave noise produces smooth terrain without biome variation [S1, unofficial].

## Background
Originating from early WebGL voxel engine experiments around 2014, this method was demonstrated in the "vox2" project by Nergal (Lallassu) [S1, unofficial].

## Sources
[S1] Show HN: Basic voxel engine in WebGL/JavaScript with full source, https://github.com/Lallassu/vox2, fetched 2026-10-26 (unofficial)
[S2] StanislavPetrovV/Minecraft, https://github.com/StanislavPetrovV/Minecraft, fetched 2026-10-26 (unofficial)
