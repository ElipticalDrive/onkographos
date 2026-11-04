# Procedural Cave Generation

Also called: Blobby cavities, Cave Generation, GenerateCaves, Particle-based Cave Generation

## Overview
Procedural cave generation creates subsurface voids in voxel terrain using either subtractive particle tracing or threshold-based noise filtering. Unlike [Simplex Noise Chunk Generation](5016-simplex-noise-chunk-generation.md) which generates the base terrain, this method operates as a post-processing or layered feature to carve out complex, organic underground structures. It differs from [Procedural Object Generation](5028-procedural-object-generation.md) by focusing on large-scale, continuous erosion-like voids rather than discrete static meshes.

## How it works
Two distinct algorithmic families exist in the literature: particle-based erosion and noise-threshold carving.

### Inputs and Outputs
*   **Input:** A voxel dataset (typically initialized by [Simplex Noise Chunk Generation](5016-simplex-noise-chunk-generation.md) or [Chunk Generation with Base Terrain](1110-chunk-generation-with-base-terrain.md)), containing stone blocks.
*   **Output:** A modified voxel dataset where specific blocks are set to "air" (empty) to form caves.

### Method 1: Particle-Based Erosion (Subtractive)
This method traces a decaying sphere through the volume to simulate speleological erosion [S1].

**Data Structures:**
*   `Particle`: Contains position $P$ (vector), radius $r$ (scalar), direction $D$ (vector), gravity $G$ (vector), and velocity $U$ (vector).
*   `VoxelData`: 3D array or sparse buffer mapping coordinates to block types.

**Parameters:**
*   $\lambda$: Decay parameter for radius reduction per iteration [S1].
*   $N$: Number of iterations/steps [S1].
*   $D$: Initial user-specified direction vector [S1].
*   $G$: Gravity vector [S1].
*   $U$: Velocity vector derived from curl noise [S1].

**Steps:**
1.  Initialize the voxel world with solid stone, typically using 3D Perlin or Simplex noise [S1][S3, unofficial].
2.  Ensure a guaranteed spawn room exists at the center of the world [S3, unofficial].
3.  Enclose the world with an indestructible outer shell to prevent void leakage [S3, unofficial].
4.  For each cave particle:
    a. Set initial position $P_0$ and radius $r_0$.
    b. For $t$ from 1 to $N$:
        i. Update position: $P(t) = P(t-1) + t \cdot (D + G + U)$ [S1].
        ii. Update radius: $r(t) = r_0 - t\lambda$ [S1].
        iii. For each voxel $V$ within the bounding box of the sphere at $P(t)$:
            1. Calculate distance $d = \|P(t) - V\|$.
            2. If $d < r(t)$, mark voxel $V$ for removal (set density to negative or block type to air) [S1].
    c. Generate curl noise $U$ for the next step to create organic, non-linear paths [S1].

**Pseudocode (Particle-Based):**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_caves_particles(voxel_data, world_bounds, num_particles, iterations, decay_lambda, gravity, initial_direction):
    for p in range(num_particles):
        # Initialize particle
        pos = get_random_start_position(world_bounds)
        radius = initial_radius
        velocity = curl_noise(pos) # [Bridson et al. 2007]
        direction = initial_direction
        
        for t in range(1, iterations + 1):
            # Update physics
            # Note: Source [S1] uses t as a multiplier for the velocity sum
            displacement = (direction + gravity + velocity) * t
            pos = pos + displacement
            
            # Decay radius
            radius = initial_radius - (t * decay_lambda)
            
            if radius <= 0:
                break
                
            # Subtract voxels within sphere
            min_v = floor(pos - radius)
            max_v = ceil(pos + radius)
            
            for x in range(min_v.x, max_v.x):
                for y in range(min_v.y, max_v.y):
                    for z in range(min_v.z, max_v.z):
                        voxel_pos = Vector3(x, y, z)
                        dist = length(pos - voxel_pos)
                        
                        if dist < radius:
                            # Mark for removal. 
                            # In a subtractive blend, this sets density to negative.
                            # In a direct voxel map, set block type to AIR.
                            voxel_data.set_block(x, y, z, BLOCK_AIR)
            
            # Update velocity for next step using curl noise
            velocity = curl_noise(pos)
```

### Method 2: Noise-Threshold Carving (Additive/Subtractive Hybrid)
This method applies 3D noise to stone blocks and removes them based on a threshold, often avoiding liquid-adjacent blocks [S2, unofficial].

**Data Structures:**
*   `caveMap`: 2D array of size `worldWidth` x `worldWidth` tracking surface cave openings [S2, unofficial].
*   `BlockData`: Integer ID representing block type (e.g., 1 for stone, 0 for air, 4 for liquid) [S2, unofficial].

**Parameters:**
*   `threshold`: Noise value cutoff for removal [S2, unofficial].
*   `compression`: Scaling factor for noise coordinates [S2, unofficial].

**Steps:**
1.  Iterate through all voxels $(x, y, z)$ in the chunk [S2, unofficial].
2.  If the block is stone (`GetBlockData(x, y, z) == 1`):
    a. Check if any adjacent block is liquid. If `liquidAdjacent` is true, skip removal [S2, unofficial].
    b. Calculate noise value: $n = \text{simplex3}(x \cdot \frac{\text{compression}}{100}, y \cdot \frac{\text{compression}}{100}, z \cdot \frac{\text{compression}}{100})$ [S2, unofficial].
    c. If $n > \text{threshold}$, set block $(x, y, z)$ to air (0) [S2, unofficial].
    d. If the removed block is at the surface ($y == \text{heightMap}[x][z]$), mark `caveMap[x][z] = 1` [S2, unofficial].
3.  Ensure a guaranteed spawn room exists at the center [S3, unofficial].
4.  Enclose the world with an indestructible outer shell [S3, unofficial].

**Pseudocode (Noise-Threshold):**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_caves_noise(voxel_data, world_width, world_height, height_map, threshold, compression, liquid_id):
    cave_map = [[0 for z in range(world_width)] for x in range(world_width)]
    
    for x in range(world_width):
        for y in range(world_height):
            for z in range(world_width):
                block_id = voxel_data.get_block(x, y, z)
                
                # Only process stone blocks
                if block_id != 1:
                    continue
                
                # Check for liquid adjacency
                liquid_adjacent = False
                for dx, dy, dz in [(1,0,0), (-1,0,0), (0,1,0), (0,-1,0), (0,0,1), (0,0,-1)]:
                    nx, ny, nz = x + dx, y + dy, z + dz
                    if 0 <= nx < world_width and 0 <= ny < world_height and 0 <= nz < world_width:
                        if voxel_data.get_block(nx, ny, nz) == liquid_id:
                            liquid_adjacent = True
                            break
                
                if liquid_adjacent:
                    continue
                
                # Calculate noise
                nx = x * compression / 100.0
                ny = y * compression / 100.0
                nz = z * compression / 100.0
                noise_value = simplex3(nx, ny, nz)
                
                if noise_value > threshold:
                    voxel_data.set_block(x, y, z, 0) # Set to air
                    
                    # Check if surface
                    if y == height_map[x][z]:
                        cave_map[x][z] = 1
                        
    return cave_map
```

### Edge Cases and Pitfalls
*   **Precision:** In particle methods, small $t$ steps may miss voxels if the grid resolution is high relative to the particle speed. Interpolation or marching cubes on the negative density field is required for smooth surfaces [S1].
*   **Boundaries:** Without an outer shell, caves may leak into the void outside the world bounds [S3, unofficial].
*   **Liquid Interaction:** Noise-based methods must explicitly check for liquid adjacency to prevent caves from forming inside water or lava, which would cause visual or logical errors [S2, unofficial].
*   **Spawn Safety:** Both methods require explicit logic to guarantee a safe spawn area, as procedural generation is stochastic [S3, unofficial].

## Operating characteristics
*   **Time Complexity:**
    *   Particle-based: $O(N \cdot R^3 \cdot V)$, where $N$ is iterations, $R$ is average radius, and $V$ is voxels per sphere volume. Efficient for sparse caves.
    *   Noise-based: $O(W \cdot H \cdot W)$, iterating over the entire chunk volume. Slower for large chunks but highly parallelizable.
*   **Memory Complexity:** $O(1)$ auxiliary space for particle methods (excluding voxel storage). Noise methods require $O(W \cdot W)$ for the `caveMap` if surface tracking is needed [S2, unofficial].
*   **Strengths:** Particle methods produce organic, tunnel-like structures resembling real speleology [S1]. Noise methods are simpler to implement and integrate with existing terrain generators [S2, S3, unofficial].
*   **Limitations:** Particle methods require careful tuning of $\lambda$ and $U$ to avoid overly thin or disconnected tunnels [S1]. Noise methods may produce blocky, less organic shapes unless high-frequency noise is used [S2, unofficial].

## Background
The particle-based approach was introduced by [S1] as an extension to volumetric terrain editing, aiming to simulate speleological erosion without the computational cost of fluid dynamics. It references [Bridson et al. 2007] for curl noise generation. The noise-threshold approach is common in indie voxel games, as seen in [S2, unofficial] and [S3, unofficial], often inspired by the "blobby" cave styles of games like *Dwarf Fortress* [S3, unofficial].

## Sources
[S1] Procedural feature generation for volumetric terrains, https://doi.org/10.1145/3102163.3102216, fetched 2026-10-26
[S2] mayakeele/bad-minecraft, https://github.com/mayakeele/bad-minecraft, fetched 2026-10-27 (unofficial)
[S3] Rabidus4k/UnityVoxelTerrain, https://github.com/Rabidus4k/UnityVoxelTerrain, fetched 2026-10-28 (unofficial)
