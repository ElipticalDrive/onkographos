# Minecraft-style Static Lighting Algorithm

Also called: Global Illumination Approximation, Minecraft Lighting Algorithm, VoxBox Lighting

## Overview
This method approximates global illumination in voxel-based environments by baking static lighting data directly into mesh vertices [S1, unofficial]. It solves the problem of efficient indirect illumination in large, dense voxel worlds by decoupling direct sunlight calculation from indirect light diffusion [S1, unofficial]. Unlike dynamic lighting systems that recalculate illumination every frame, this approach precomputes light values, offering a significant performance trade-off for static scenes at the cost of inability to handle dynamic light sources or moving geometry [S1, unofficial]. It is distinct from [Light Propagation Volumes](4332-light-propagation-volumes.md) in its use of a discrete, iterative diffusion pass rather than continuous volume propagation, and from [Two-Pass Shadow Casting (Kajiya and Von Herzen)](2196-two-pass-shadow-casting-kajiya-and-von-herzen.md) by operating on a discrete grid with integer light levels rather than continuous ray marching [G].

## How it works
The algorithm operates in two distinct passes: a direct sunlight pass and an indirect light spread pass [S1, unofficial]. It assumes a static world divided into chunks, where each chunk contains a 3D array of voxels [S1, unofficial].

### Inputs and Outputs
*   **Input:** A voxel grid representing the world geometry. Each voxel has a boolean property `BlocksLight` (true if opaque/occluding, false if transparent/translucent) [S1, unofficial].
*   **Output:** A 3D array of integer light values, one per voxel, with values ranging from 0 (no light) to 16 (full light) [S1, unofficial]. This array is typically baked into the vertex colors of the chunk mesh [S1, unofficial].

### Data Structures
*   **Voxel Grid:** A 3D array `Voxel[x][y][z]` [S1, unofficial]. The specific implementation in the source uses a chunk-based system where chunks are stored in a hash map keyed by chunk coordinates [S1, unofficial]. Each chunk is a fixed-size cube (e.g., $16 \times 16 \times 16$ voxels) [S1, unofficial].
*   **Light Array:** A 3D array `Light[x][y][z]` of integers, same dimensions as the voxel grid [S1, unofficial]. Initialized to 0 [S1, unofficial].

### Steps

#### Pass 1: Direct Sunlight
1.  Initialize `Light` array to 0 for all voxels [S1, unofficial].
2.  For each column $(x, z)$ in the world/chunk:
    3.  Trace from the highest $y$ (sky) down to the lowest $y$ (ground) [S1, unofficial].
    4.  For each voxel $(x, y, z)$ encountered:
        5.  If `Voxel[x][y][z].BlocksLight` is true, stop tracing this column [S1, unofficial].
        6.  Otherwise, set `Light[x][y][z] = 16` [S1, unofficial].
    *   *Note:* The source describes this as "decrementing as it goes down, or simply setting to 16 until blocked" [S1, unofficial]. The implementation uses a constant 16 for all unblocked voxels in the column [S1, unofficial].

#### Pass 2: Indirect Light Spread
This pass simulates the diffusion of light from lit voxels to their neighbors [S1, unofficial]. It runs for a fixed number of iterations equal to the maximum light spread distance [S1, unofficial].

1.  Set `MaxSpread = 16` [S1, unofficial].
2.  For `pass` from `MaxSpread` down to 1:
    3.  For each voxel $(x, y, z)$ in the grid:
        4.  If `Light[x][y][z] == pass`:
            5.  For each of the 6 neighbors $(nx, ny, nz)$:
                6.  If `Voxel[nx][ny][nz].BlocksLight` is false:
                    7.  If `Light[nx][ny][nz] < Light[x][y][z] - 1`:
                        8.  Set `Light[nx][ny][nz] = Light[x][y][z] - 1` [S1, unofficial].

### Formula
The core update rule for indirect lighting is:
$$ L_{neighbor} = \max(L_{neighbor}, L_{current} - 1) $$
Subject to the condition that the neighbor does not block light [S1, unofficial]. The algorithm iterates from high light values to low values to ensure that light propagates correctly from sources outward [S1, unofficial].

### Parameters
*   **Light Range:** 16 blocks. This is the maximum distance indirect light spreads [S1, unofficial].
*   **Light Levels:** Integers from 0 to 16 [S1, unofficial].
*   **Chunk Size:** Typically $16 \times 16 \times 16$ voxels, but can be configured [S1, unofficial].

### Edge Cases and Pitfalls
*   **Opaque Blocks:** Voxels with `BlocksLight = true` do not receive light from neighbors in Pass 2, nor do they transmit light [S1, unofficial].
*   **Marshalling Errors:** When implementing this in native code (e.g., C/C++ DLLs from C#), incorrect handling of multidimensional array boundaries or memory layout can cause lighting anomalies and incorrect shadows [S1, unofficial].
*   **Precision:** The use of integer arithmetic avoids floating-point precision issues but limits the granularity of light levels to 17 distinct values [G].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def calculate_static_lighting(voxel_grid, chunk_size=16):
    """
    voxel_grid: 3D list/array of objects with attribute 'blocks_light' (bool)
    chunk_size: size of the chunk in each dimension
    Returns: 3D list/array of integers representing light levels
    """
    # Initialize light array with 0
    light_grid = [[[0 for _ in range(chunk_size)] for _ in range(chunk_size)] for _ in range(chunk_size)]
    
    # Pass 1: Direct Sunlight (trace from top y down to bottom y)
    for x in range(chunk_size):
        for z in range(chunk_size):
            for y in range(chunk_size - 1, -1, -1):
                if voxel_grid[x][y][z].blocks_light:
                    break
                else:
                    light_grid[x][y][z] = 16
    
    # Pass 2: Indirect Light Spread
    max_spread = 16
    for pass_val in range(max_spread, 0, -1):
        for x in range(chunk_size):
            for y in range(chunk_size):
                for z in range(chunk_size):
                    if light_grid[x][y][z] == pass_val:
                        # Check 6 neighbors
                        neighbors = [
                            (x+1, y, z), (x-1, y, z),
                            (x, y+1, z), (x, y-1, z),
                            (x, y, z+1), (x, y, z-1)
                        ]
                        for nx, ny, nz in neighbors:
                            # Check bounds
                            if 0 <= nx < chunk_size and 0 <= ny < chunk_size and 0 <= nz < chunk_size:
                                # Check if neighbor blocks light
                                if not voxel_grid[nx][ny][nz].blocks_light:
                                    # Update neighbor light if it's lower than current - 1
                                    if light_grid[nx][ny][nz] < light_grid[x][y][z] - 1:
                                        light_grid[nx][ny][nz] = light_grid[x][y][z] - 1
    return light_grid
```

## Operating characteristics
*   **Time Complexity:** Pass 1 is $O(N)$ where $N$ is the number of voxels [S1, unofficial]. Pass 2 is $O(N \cdot K)$ where $K$ is the light spread distance (16) [S1, unofficial]. Total complexity is $O(N)$ [G].
*   **Memory Complexity:** $O(N)$ for the light array [S1, unofficial]. The source notes that storing chunk positions explicitly saves memory compared to storing coordinates for every voxel, but the light array itself is dense [S1, unofficial].
*   **Performance:** In the source implementation, calculating lighting for 5,214 chunks (approx. 20 million voxels) took 11.79 seconds on an i7-4712HQ, or ~2.26ms per chunk [S1, unofficial]. A native C version was 2-3x faster but suffered from bugs [S1, unofficial].
*   **Strengths:** Very fast for static scenes; simple to implement; low memory overhead per voxel (1 byte for light level) [G].
*   **Limitations:** Cannot handle dynamic light sources or moving geometry; limited to 17 light levels; indirect light spread is limited to 16 blocks; no attenuation based on distance beyond the integer step [S1, unofficial].

## Background
The algorithm is adapted from the lighting system used in the game Minecraft [S1, unofficial]. It was implemented in the VoxBox project, a procedural voxel world generator [S1, unofficial]. The system is described as a "crude global illumination approximation" baked into chunk mesh vertices [S1, unofficial]. The source notes that dynamic lighting is handled separately by the Unity3D engine, while this static system handles the bulk of the illumination for the voxel geometry [S1, unofficial]. The implementation details, including the 16-pass diffusion and the specific pitfalls of native code marshalling, are documented in a thesis from the University of Waikato [S1, unofficial].

## Sources
[S1] Procedural Generation of Voxel Worlds with Castles, https://researchcommons.waikato.ac.nz/entities/publication/70d98783-1629-43df-9d01-b973ddbb5d75, fetched 2026-10-28 (unofficial)
