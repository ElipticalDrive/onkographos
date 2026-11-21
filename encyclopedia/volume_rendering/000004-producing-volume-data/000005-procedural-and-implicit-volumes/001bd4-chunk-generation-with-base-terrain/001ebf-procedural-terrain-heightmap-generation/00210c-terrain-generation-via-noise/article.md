# Terrain Generation via Noise

Also called: Perlin Noise Terrain Generation, Perlin Noise for Terrain, Noise Terrain, Heightmap Terrain

## Overview
Terrain Generation via Noise generates a 3D voxel volume from a 2D noise heightmap, extruding terrain-like structures into voxel space by filling blocks from a base level up to a calculated height [S2, S5, unofficial]. It solves the problem of creating deterministic, pseudo-random terrain landscapes suitable for voxel engines [S10, unofficial]. It is distinguished from sibling methods by its reliance on 2D spatial noise functions (typically Perlin or gradient noise) to define surface elevation, whereas siblings like [Procedural Terrain Feature Generation](2572-procedural-terrain-feature-generation.md) may use different displacement models, and [Smooth Terrain Grid](5026-smooth-terrain-grid.md) focuses on mesh smoothing rather than raw block placement.

## How it works
**Inputs:**
*   Chunk dimensions: $W$ (width in x), $H$ (height in y), $D$ (depth in z).
*   World coordinates of the chunk origin $(O_x, O_z)$.
*   Noise parameters: Seed, Frequency ($f$), Amplitude ($A$), Octaves ($N$), Lacunarity ($L$), Gain ($G$).
*   Sea level ($S$) or base height offset.
*   Block type definitions (e.g., Air, Grass, Dirt, Stone, Bedrock, Snow).

**Outputs:**
*   A 3D array or buffer of block IDs (or densities) with dimensions $W \times H \times D$.

**Data Structures:**
*   `BlockType`: Enum or integer ID mapping to material properties.
*   `NoiseState`: Object encapsulating the permutation table and gradient vectors for the noise function.
*   `ChunkBuffer`: 1D or 3D array of size $W \times H \times D$.

**Algorithm:**

1.  **Initialize Noise:** Create a Perlin noise instance with a fixed seed to ensure deterministic output for the same coordinates [S10, unofficial]. The permutation table typically contains 512 entries, where indices 0-255 are a random permutation of 0-255, repeated to allow tiling [S10, unofficial].

2.  **Iterate Chunk Grid:** For each local coordinate $(x, z)$ where $x \in [0, W-1]$ and $z \in [0, D-1]$:
    a.  **Calculate Global Coordinates:**
        $$gx = O_x + x$$
        $$gz = O_z + z$$
    b.  **Sample Noise (Fractal Brownian Motion):**
        Compute the base height using multiple octaves of 2D noise.
        $$height = 0$$
        $$current\_amplitude = A$$
        $$current\_frequency = f$$
        $$max\_noise = -\infty$$
        $$min\_noise = \infty$$
        
        For $i$ from 0 to $N-1$:
        $$sample\_x = gx \cdot current\_frequency$$
        $$sample\_z = gz \cdot current\_frequency$$
        $$n = \text{Perlin2D}(sample\_x, sample\_z)$$
        $$height += n \cdot current\_amplitude$$
        
        Update min/max for normalization:
        $$max\_noise = \max(max\_noise, height)$$
        $$min\_noise = \min(min\_noise, height)$$
        
        Update parameters for next octave:
        $$current\_amplitude = current\_amplitude \cdot G$$
        $$current\_frequency = current\_frequency \cdot L$$

    c.  **Normalize and Scale Height:**
        Normalize the accumulated noise to $[0, 1]$ range:
        $$normalized = \frac{height - min\_noise}{max\_noise - min\_noise}$$
        *(Note: Some implementations normalize to $[-1, 1]$ first, then map to $[0, 1]$ via $(n+1)/2$ [S11, unofficial].)*
        
        Apply non-linear shaping if required (e.g., cubic falloff) [S8, S9, unofficial]:
        $$shaped = \text{pow}(normalized, 3)$$
        
        Calculate final terrain height in voxels:
        $$h = \text{clamp}(\text{RoundToInt}(shaped \cdot \text{HeightMultiplier} + \text{BaseOffset}), 0, H-1)$$
        *(Specific implementations use formulas like $h = \text{clamp}(((n + 1.0) \cdot 0.5 \cdot \text{CHUNK\_HEIGHT} \cdot 0.6 + \text{CHUNK\_HEIGHT} \cdot 0.15), 0, \text{CHUNK\_HEIGHT} - 1)$ [S5, unofficial], or $h = \text{RoundToInt}(noise \cdot 70)$ [S8, unofficial].)*

    d.  **Apply Island Mask (Optional):**
        If generating an island, apply a falloff based on distance from center $(C_x, C_z)$:
        $$dist = \sqrt{(gx - C_x)^2 + (gz - C_z)^2}$$
        $$mask = \frac{1}{\text{pow}(k \cdot dist, p) + \epsilon}$$
        $$h = \text{RoundToInt}(h \cdot \text{clamp}(mask, 0, 1))$$
        *(Parameters $k, p, \epsilon$ vary; e.g., $k=0.0025, p=20$ [S3, unofficial].)*

    e.  **Fill Voxels:**
        For each $y$ from 0 to $H-1$:
        i.  If $y < h$:
            Set block type based on depth layers:
            -   If $y == 0$: `Bedrock` [S7, S11, unofficial].
            -   If $y > h - 3$: `Dirt` [S7, S11, unofficial].
            -   Else: `Stone` [S7, S11, unofficial].
        ii. If $y == h$:
            Determine surface type:
            -   If $y > \text{SnowThreshold} + \text{Variance}$: `Snow` [S8, S9, unofficial].
            -   Else: `Grass` [S7, S11, unofficial].
        iii. If $y > h$:
            Set block type to `Air` (or density 0/255) [S8, S9, unofficial].

3.  **Post-Processing:**
    -   Apply player edits or saved block states [S7, unofficial].
    -   Generate trees or features if a tree generation pass is integrated [S7, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_terrain_chunk(chunk_origin_x, chunk_origin_z, chunk_width, chunk_height, chunk_depth, noise_seed, frequency, amplitude, octaves, lacunarity, gain, sea_level, height_multiplier):
    # Initialize noise generator with seed
    noise_gen = PerlinNoise(seed=noise_seed)
    
    # Pre-calculate min/max for normalization if using full range normalization
    # Note: For performance, many implementations skip full min/max pass and use fixed scaling
    # Here we assume a normalized approach similar to [S5, S11, unofficial]
    
    chunk_data = [] # 3D array [x][y][z] or flattened
    
    for x in range(chunk_width):
        for z in range(chunk_depth):
            # Global coordinates
            gx = chunk_origin_x + x
            gz = chunk_origin_z + z
            
            # Fractal Brownian Motion
            total_noise = 0.0
            current_amp = amplitude
            current_freq = frequency
            max_val = -1.0
            min_val = 1.0
            
            for o in range(octaves):
                # Sample 2D Perlin noise
                # Perlin2D returns value in [-1, 1] typically
                n = noise_gen.noise2d(gx * current_freq, gz * current_freq)
                
                total_noise += n * current_amp
                
                if total_noise > max_val:
                    max_val = total_noise
                if total_noise < min_val:
                    min_val = total_noise
                
                current_amp *= gain
                current_freq *= lacunarity
            
            # Normalize to [0, 1]
            # Handle case where max == min to avoid division by zero
            if max_val == min_val:
                normalized = 0.5
            else:
                normalized = (total_noise - min_val) / (max_val - min_val)
            
            # Apply non-linearity (optional, e.g., cubic)
            # normalized = pow(normalized, 3) 
            
            # Scale to height
            # Formula from [S5, unofficial]: h = clamp(((n + 1.0) * 0.5 * CHUNK_HEIGHT * 0.6 + CHUNK_HEIGHT * 0.15), 0, CHUNK_HEIGHT - 1)
            # Using generic multiplier approach from [S8, S9, unofficial]:
            height = int(round(normalized * height_multiplier + sea_level))
            
            # Clamp height to chunk bounds
            height = max(0, min(height, chunk_height - 1))
            
            # Fill column
            for y in range(chunk_height):
                if y < height:
                    if y == 0:
                        block = BLOCK_BEDROCK
                    elif y > height - 3:
                        block = BLOCK_DIRT
                    else:
                        block = BLOCK_STONE
                elif y == height:
                    # Surface type
                    if height > 20: # Snow threshold example
                        block = BLOCK_SNOW
                    else:
                        block = BLOCK_GRASS
                else:
                    block = BLOCK_AIR
                
                # Store block (layout depends on engine, e.g., x + z*width + y*width*depth)
                chunk_data[x, y, z] = block

    return chunk_data
```

**Parameters & Typical Values:**
*   **Frequency:** Controls spatial scale. Low values (e.g., 0.01–0.05) create large landmasses; high values create rough, detailed terrain [S8, S9, S11, unofficial].
*   **Amplitude:** Controls vertical scale. Higher values create taller mountains [S10, unofficial].
*   **Octaves:** Number of noise layers. 4–6 is common for detail [S8, S9, unofficial].
*   **Lacunarity:** Frequency multiplier between octaves. Typically 2.0 [S5, unofficial].
*   **Gain:** Amplitude multiplier between octaves. Typically 0.5 [S5, unofficial].
*   **Height Multiplier:** Scales noise to world units. E.g., 70 [S8, S9, unofficial].

**Edge Cases & Pitfalls:**
*   **Repetition:** Without a large permutation table or unique seeds per chunk, terrain may tile visibly [S10, unofficial].
*   **Variance Consistency:** Using global random number generators for surface details (like snow variance) without seeding per-chunk causes identical patterns across chunks [S9, unofficial].
*   **Sharp Peaks:** Non-linear functions like `pow(noise, 3)` can create flat plateaus or sharp peaks depending on the noise distribution [S9, unofficial].
*   **Boundary Conditions:** Ensure height clamping prevents out-of-bounds memory access in the voxel array [S11, unofficial].

## Subtopics

### [Procedural Cave Height and Layer Generation](3202-procedural-cave-height-and-layer-generation.md)
Modifies voxel density to create a horizontal cave layer with variable height, stalactites, and Perlin noise displacement. It builds on [Terrain Generation via Noise](2412-terrain-generation-via-noise.md) by introducing a Z-axis dependent noise profile that carves out a specific elevation band, distinct from the surface height generation [G].

## Operating characteristics
*   **Time Complexity:** $O(W \cdot H \cdot D)$ per chunk, as every voxel must be evaluated or filled. The noise calculation is $O(W \cdot D \cdot N)$ where $N$ is octaves.
*   **Memory Complexity:** $O(W \cdot H \cdot D)$ for the voxel buffer.
*   **Performance:** Fast for simple heightmaps. Performance degrades if complex 3D noise or extensive post-processing (trees, caves) is added. Caching heightmaps can optimize repeated access [S4, unofficial].
*   **Strengths:** Deterministic, easy to implement, low memory overhead for simple blocks, infinite world support via chunking.
*   **Limitations:** Blocky appearance without smoothing; limited geological realism compared to fractal-based or simulation-based methods; sharp transitions between block types.
*   **When to Use:** Voxel games, Minecraft-like engines, procedural level generation where blocky aesthetics are acceptable.
*   **When Not to Use:** High-fidelity terrain rendering requiring smooth meshes, or when complex geological layering is required.

## Background
The method originates from the use of Perlin noise, introduced by Ken Perlin in 1985, for procedural texture and terrain generation [S10, unofficial]. The permutation table of 512 numbers (0-255 repeated) allows for tiling without visible seams [S10, unofficial]. Improved noise functions using quintic interpolation ($6x^5 - 15x^4 + 10x^3$) replaced the original cubic interpolation for smoother gradients [S10, unofficial]. The application to voxel terrain heightmaps became standard in voxel engine development, leveraging the pseudo-random property to preserve terrain data across frames [S10, unofficial].

## Sources
[S1] Ray-Marched Voxel Simulation, https://www.jorisar.nl/blog/VoxelPlayground/, fetched 2026-10-25 (unofficial)
[S2] millerc3/Marching-Cubes, https://github.com/millerc3/Marching-Cubes, fetched 2026-10-26 (unofficial)
[S3] StanislavPetrovV/Minecraft, https://github.com/StanislavPetrovV/Minecraft, fetched 2026-10-26 (unofficial)
[S4] tonadr1022/vulkan_voxels, https://github.com/tonadr1022/vulkan_voxels, fetched 2026-10-27 (unofficial)
[S5] scarlyle0/voxel-compare, https://github.com/scarlyle0/voxel-compare, fetched 2026-10-27 (unofficial)
[S6] trimesh33/sdf, https://github.com/trimesh33/sdf, fetched 2026-10-27 (unofficial)
[S7] philip2012/voxel-forge, https://github.com/philip2012/voxel-forge, fetched 2026-10-27 (unofficial)
[S8] Creating a Voxel Engine with OpenGL | Walker McGilvary, https://theslabby.github.io/2023/12/24/start-voxelengine.html, fetched 2026-10-28 (unofficial)
[S9] GitHub Repo for VoxelEngine, https://github.com/TheSlabby/VoxelEngine, fetched 2026-10-29 (unofficial)
[S10] Voxel-Based Procedural Terrain Generation | PDF | Voxel | Rendering ..., https://www.scribd.com/document/840779393/Procedural-Terrain-Generation-Using-Voxels, fetched 2026-10-29 (unofficial)
[S11] GitHub - ernerdgn/voxel-engine: A voxel engine written from scratch in ..., https://github.com/ernerdgn/voxel-engine, fetched 2026-10-29 (unofficial)
