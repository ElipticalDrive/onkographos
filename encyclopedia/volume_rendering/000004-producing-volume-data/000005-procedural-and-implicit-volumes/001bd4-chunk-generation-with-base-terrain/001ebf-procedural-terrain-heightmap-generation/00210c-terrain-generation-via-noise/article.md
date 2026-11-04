# Terrain Generation via Noise

Also called: Perlin Noise Terrain Generation, Perlin Noise for Terrain, Noise Terrain, Heightmap Terrain

## Overview
Terrain Generation via Noise generates a 3D voxel volume from a 2D noise heightmap, extruding terrain-like structures into voxel space [S2, S4, S5, unofficial]. It solves the problem of creating deterministic, pseudo-random terrain geometry by leveraging the property that a fixed input yields the same output, allowing terrain data to be preserved across frames without change [S6, unofficial]. It sets itself apart from [Procedural Terrain Feature Generation](2572-procedural-terrain-feature-generation.md) by focusing on continuous height variation rather than discrete feature placement, and from [Heightmap generation via circular displacements](5186-heightmap-generation-via-circular-displacements.md) by using gradient-based noise functions rather than radial displacement patterns.

## How it works
**Inputs:**
- Chunk dimensions: $xSize, ySize, zSize$ (integers).
- Noise parameters: Perlin noise implementation, fractal octaves, lacunarity, gain, amplitude, frequency, sea level, island mask center, and open middle radius [S2, S4, S7, unofficial].
- World coordinates: Chunk offset $(cx, cz)$ to convert local chunk coordinates to world coordinates [S4, unofficial].

**Outputs:**
- A 3D density array `density[x][y][z]` of size $xSize \times ySize \times zSize$ [S2, unofficial].
- Optional block type assignment based on height relative to sea level [S7, unofficial].

**Data Structures:**
- **Permutation Table:** An array of 512 integers. Indices 0–255 contain a random permutation of 0–255, repeated to fill the table to ensure tiling without recognition [S6, unofficial].
- **Noise State:** Accumulated density value, min/max density trackers for normalization [S2, unofficial].

**Steps:**

1.  **Initialize Noise Parameters:**
    Define amplitude and frequency layers for fractal Brownian motion (FBm). Typical values:
    - Amplitudes: $a_1 = \text{CENTER\_Y}$, $a_2 = a_1 \times 0.5$, $a_4 = a_1 \times 0.25$, $a_8 = a_1 \times 0.125$ [S3, unofficial].
    - Frequencies: $f_1 = 0.005$, $f_2 = f_1 \times 2$, $f_4 = f_1 \times 4$, $f_8 = f_1 \times 8$ [S3, unofficial].
    - Lacunarity: 2.0 [S4, unofficial].
    - Gain (Persistence): 0.5 [S4, unofficial].
    - Octaves: 4 [S4, unofficial].

2.  **Generate 2D Heightmap:**
    For each local coordinate $(x, z)$ in the chunk:
    a. Convert to world coordinates: $wx = x + cx \times xSize$, $wz = z + cz \times zSize$ [S4, unofficial].
    b. Initialize `noiseDensity = 0`, `amplitude = initial_amplitude`, `frequency = initial_frequency` [S2, unofficial].
    c. Iterate through octaves $i = 0$ to $octaves-1$:
       i. Calculate sample coordinates:
          $$sampleX = \frac{wx + \text{octaveOffsets}[i].x}{\text{scale}} \times \text{frequency}$$
          $$sampleZ = \frac{wz + \text{octaveOffsets}[i].y}{\text{scale}} \times \text{frequency}$$
          [S2, unofficial]
       ii. Evaluate 2D Perlin noise:
           $$perlinValue = \text{Perlin}(sampleX, sampleZ) \times 2 - 1$$
           [S2, unofficial]
       iii. Accumulate:
            $$noiseDensity += perlinValue \times amplitude$$
            [S2, unofficial]
       iv. Update parameters:
           $$amplitude *= \text{gain}$$
           $$frequency *= \text{lacunarity}$$
           [S2, unofficial]
    d. Track min/max density for normalization [S2, unofficial].
    e. Normalize to [0, 1]:
       $$noiseDensity = \frac{noiseDensity - \text{minNoiseDensity}}{\text{maxNoiseDensity} - \text{minNoiseDensity}}$$
       [S2, unofficial]
    f. Scale by factor (e.g., 0.4) [S2, unofficial].

3.  **Apply Island Mask:**
    Calculate distance from center $(CENTER\_XZ, CENTER\_XZ)$:
    $$dist = \sqrt{(wx - CENTER\_XZ)^2 + (wz - CENTER\_XZ)^2}$$
    [S3, unofficial]
    Compute mask:
    $$island = \frac{1}{\max(0.0001, \text{pow}(0.0025 \times dist, 20))}$$
    Clamp `island` to max 1.0 [S3, unofficial].
    Multiply heightmap value by `island` [S3, unofficial].

4.  **Apply Height Constraints:**
    a. Minimum height constraint using high-frequency noise:
       $$height = \max(height, \text{noise2}(x \times f_8, z \times f_8) + 2)$$
       [S3, unofficial]
    b. Adjust amplitude if noise is negative (optional variation):
       If $\text{noise2}(0.1 \times x, 0.1 \times z) < 0$, reduce $a_1$ by dividing by 1.07 [S3, unofficial].

5.  **Calculate Final Height:**
    Normalize noise value $n \in [-1, 1]$ to [0, 1]:
    $$n_{norm} = (n + 1.0) \times 0.5$$
    [S7, unofficial]
    Calculate height in voxels:
    $$h = \text{clamp}\left( (n_{norm} \times \text{CHUNK\_HEIGHT} \times 0.6) + (\text{CHUNK\_HEIGHT} \times 0.15), 0, \text{CHUNK\_HEIGHT} - 1 \right)$$
    [S4, unofficial]
    Alternatively, using sea level and amplitude:
    $$h = \text{RoundToInt}((\text{noise\_value} + 1.0) \times 0.5 \times \text{amplitude} + \text{sea\_level})$$
    [S7, unofficial]
    Clamp $h$ to $[0, ySize - 1]$ [S7, unofficial].

6.  **Extrude to 3D Volume:**
    For each $(x, z)$ and $y$ from 0 to $ySize - 1$:
    a. If $y < h$: `density[x][y][z] = 0` (Air) [S2, unofficial].
    b. If $y == h$: `density[x][y][z] = 0.51` (Surface) [S2, unofficial].
    c. If $y > h$: `density[x][y][z] = 1` (Solid) [S2, unofficial].
    d. **Bottom Clamping:** If $y \le ySize / 8$, set `density[x][y][z] = 0` [S2, unofficial].

7.  **Assign Block Types (Optional):**
    For each voxel at height $h$:
    a. If $y == h$: Block Type 1 (Grass) [S7, unofficial].
    b. If $y > h - 3$: Block Type 2 (Dirt) [S7, unofficial].
    c. Else: Block Type 3 (Stone) [S7, unofficial].
    d. **Surface Rule Variation:** If there is air two blocks above the voxel, set to Grass; otherwise Rock [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_terrain(x_size, y_size, z_size, chunk_offset_x, chunk_offset_z, 
                     center_xz, open_middle_radius, sea_level, amplitude, frequency, 
                     lacunarity, gain, octaves):
    """
    Generates a 3D voxel density array using 2D Perlin noise.
    """
    # Initialize permutation table if not present
    if not permutation_table_initialized:
        init_permutation_table()

    density = [[[0.0 for _ in range(z_size)] for _ in range(y_size)] for _ in range(x_size)]
    
    min_noise = float('inf')
    max_noise = float('-inf')
    heightmap = [[0.0 for _ in range(z_size)] for _ in range(x_size)]

    # Step 1 & 2: Generate 2D Heightmap
    for x in range(x_size):
        for z in range(z_size):
            # Convert to world coordinates
            wx = x + chunk_offset_x * x_size
            wz = z + chunk_offset_z * z_size
            
            noise_density = 0.0
            current_amplitude = amplitude
            current_frequency = frequency
            
            for i in range(octaves):
                # Calculate sample coordinates
                sample_x = (wx + octave_offsets[i].x) / scale * current_frequency
                sample_z = (wz + octave_offsets[i].y) / scale * current_frequency
                
                # Evaluate Perlin Noise
                perlin_val = perlin_noise_2d(sample_x, sample_z)
                # Normalize to [-1, 1]
                perlin_val = perlin_val * 2 - 1
                
                # Accumulate
                noise_density += perlin_val * current_amplitude
                
                # Update parameters
                current_amplitude *= gain
                current_frequency *= lacunarity
            
            # Track min/max
            if noise_density < min_noise: min_noise = noise_density
            if noise_density > max_noise: max_noise = noise_density
            
            heightmap[x][z] = noise_density

    # Normalize heightmap to [0, 1]
    range_noise = max_noise - min_noise
    if range_noise == 0: range_noise = 1
    
    for x in range(x_size):
        for z in range(z_size):
            normalized = (heightmap[x][z] - min_noise) / range_noise
            normalized = normalized * 0.4 # Scale factor
            
            # Step 3: Island Mask
            dist = math.hypot(wx - center_xz, wz - center_xz)
            island = 1.0 / (pow(0.0025 * dist, 20) + 0.0001)
            if island > 1.0: island = 1.0
            
            normalized *= island
            
            # Step 4 & 5: Height Calculation
            # Clamp noise to [-1, 1] before height calc if needed
            n = max(-1.0, min(1.0, normalized * 2 - 1)) # Assuming normalized is [0,1] -> [-1,1]
            # Formula from [S4, unofficial]
            h = clamp(((n + 1.0) * 0.5 * y_size * 0.6 + y_size * 0.15), 0, y_size - 1)
            
            # Step 6: Extrude to 3D
            for y in range(y_size):
                if y < h:
                    density[x][y][z] = 0.0
                elif y == h:
                    density[x][y][z] = 0.51
                else:
                    density[x][y][z] = 1.0
                
                # Bottom clamping
                if y <= y_size / 8:
                    density[x][y][z] = 0.0

    return density
```

**Edge Cases and Pitfalls:**
- **Precision:** Perlin noise implementation must handle floating-point precision carefully to avoid seams at chunk boundaries [S6, unofficial].
- **Boundaries:** The island mask can produce extreme values if the center is far from the chunk; clamping is essential [S3, unofficial].
- **Degenerate Input:** If `max_noise == min_noise`, division by zero occurs during normalization; guard against this [S2, unofficial].

## Subtopics

### [Procedural Cave Height and Layer Generation](3202-procedural-cave-height-and-layer-generation.md)
Modifies voxel density to create a horizontal cave layer with variable height, stalactites, and Perlin noise displacement. It builds on [Terrain Generation via Noise](2412-terrain-generation-via-noise.md) by introducing a Z-axis dependent noise profile that carves out a specific elevation band, distinct from the surface height generation [G].

## Operating characteristics
- **Time Complexity:** $O(xSize \times ySize \times zSize)$ for extrusion, $O(xSize \times zSize \times \text{octaves})$ for heightmap generation. Total is dominated by volume filling.
- **Memory Complexity:** $O(xSize \times ySize \times zSize)$ for the density array.
- **Performance:** Generating a $64 \times 64 \times 64$ chunk takes milliseconds on modern CPUs. GPU acceleration is possible for the noise evaluation step [S4, unofficial].
- **Strengths:** Deterministic, seamless tiling with proper permutation tables, easy to control via amplitude/frequency [S6, unofficial].
- **Limitations:** Can appear repetitive if octaves are insufficient; requires careful tuning of island masks for non-uniform terrain [S3, unofficial].
- **When to Use:** Ideal for voxel-based games requiring consistent, procedural terrain [S6, unofficial].
- **When Not to Use:** When highly varied, non-noise-based features (e.g., specific mountain ranges) are required without additional post-processing [S2, unofficial].

## Background
Originating from Ken Perlin's work on gradient noise, this method leverages the pseudo-random property of Perlin noise to generate deterministic terrain [S6, unofficial]. The permutation table, containing 512 numbers (0-255 repeated), ensures tiling without recognition [S6, unofficial]. Improved noise uses a smoother interpolation function $f(x) = 6x^5 - 15x^4 + 10x^3$ [S6, unofficial]. It is widely used in voxel engines and procedural generation tools for games [S1, S7, unofficial].

## Sources
[S1] Ray-Marched Voxel Simulation, https://www.jorisar.nl/blog/VoxelPlayground/, fetched 2026-10-25 (unofficial)
[S2] millerc3/Marching-Cubes, https://github.com/millerc3/Marching-Cubes, fetched 2026-10-26 (unofficial)
[S3] StanislavPetrovV/Minecraft, https://github.com/StanislavPetrovV/Minecraft, fetched 2026-10-26 (unofficial)
[S4] scarlyle0/voxel-compare, https://github.com/scarlyle0/voxel-compare, fetched 2026-10-27 (unofficial)
[S5] trimesh33/sdf, https://github.com/trimesh33/sdf, fetched 2026-10-27 (unofficial)
[S6] Voxel-Based Procedural Terrain Generation, https://www.scribd.com/document/840779393/Procedural-Terrain-Generation-Using-Voxels, fetched 2026-10-29 (unofficial)
[S7] ernerdgn/voxel-engine, https://github.com/ernerdgn/voxel-engine, fetched 2026-10-29 (unofficial)
