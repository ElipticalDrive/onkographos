# Chunk-based Voxel World Generation

Also called: Density-based voxel placement, Simplex noise terrain generation, Terrain Generation

## Overview
This method generates a 3D voxel world by sampling 3D simplex noise to determine surface height and internal density, filtering blocks against a water level threshold and a density cutoff [S1, unofficial]. It builds on [Chunk-Based Voxel Terrain Generation](4140-chunk-based-voxel-terrain-generation.md) by defining the specific procedural noise sampling and block placement rules used within each chunk [G].

## How it works
**Inputs:**
- `worldSize`: Total world dimensions in blocks (integer).
- `chunkSize`: Blocks per chunk dimension (integer).
- `blockSize`: World units per block (float).
- `waterlevel`: Y-coordinate threshold below which blocks are not generated (float).
- `density_threshold`: Minimum simplex noise value to place a block (float).
- `noise_scale_x`: X coordinate divisor for height noise (float).
- `noise_scale_y`: Y coordinate divisor for density noise (float).
- `seed`: Random seed for noise initialization.

**Outputs:**
- A collection of active chunks, each containing a 3D array of voxel IDs (block types).

**Data Structures:**
- `Chunk`: Contains a 3D array `voxels` of size `[chunkSize, chunkSize, chunkSize]` of type `uint8` (or equivalent block ID type) [S2, unofficial].
- `World`: Contains a map or array of `Chunk` objects indexed by chunk coordinates, plus a `chunkIdMap` tracking active chunks [S1, unofficial].
- `Block`: Represents a single voxel with position `(x, y, z)` and color/type [S1, unofficial].

**Parameters (Typical Values from [S1, unofficial]):**
- `worldSize = 192`
- `chunkSize = 16`
- `blockSize = 0.1`
- `waterlevel = 0`
- `density_threshold = 0.5`
- `noise_scale_x = 100`
- `noise_scale_y = 100`

**Steps:**
1. Initialize the noise generator with a random seed [S1, unofficial].
2. Initialize the world data structure, including a map of chunk IDs to `Chunk` objects and an active chunk list [S1, S2, unofficial].
3. Iterate through every block coordinate `(x, z)` in the horizontal plane from `0` to `worldSize - 1` [S1, unofficial].
4. Calculate the surface height `H` for column `(x, z)` using a 3D simplex noise function:
   $$ H = \text{simplex3}\left(\frac{x}{\text{noise\_scale\_x}}, \frac{x}{2 \cdot \text{noise\_scale\_x}}, \frac{z}{\text{noise\_scale\_x}}\right) \cdot \text{worldSize} $$
   Note: The source code uses `x/200` for the second coordinate, which corresponds to `x/(2*noise_scale_x)` [S1, unofficial].
5. Iterate through every block coordinate `y` from `0` to `H - 1` [S1, unofficial].
6. Check if `y >= waterlevel`. If false, skip this block [S1, unofficial].
7. Calculate the internal density `D` for block `(x, y, z)` using a separate 3D simplex noise function:
   $$ D = \text{simplex3}\left(\frac{x}{\text{noise\_scale\_y}}, \frac{y}{\text{noise\_scale\_y}}, \frac{z}{\text{noise\_scale\_y}}\right) $$
   The source uses `noise_scale_y` (100) for all axes in the density calculation [S1, unofficial].
8. If `D > density_threshold`, mark block `(x, y, z)` as solid [S1, unofficial].
9. After iterating all `x, z, y`, group the solid blocks into chunks based on their coordinates [S1, unofficial].
10. For each chunk containing solid blocks, rebuild its mesh or finalize its data structure [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_world(world_size, chunk_size, block_size, water_level, density_threshold, noise_scale):
    # Initialize noise generator with a random seed [S1, unofficial]
    noise_seed = random.random()
    
    # Data structures: map chunk_id to Chunk object, track active chunks [S1, S2, unofficial]
    chunks = {}
    active_chunks = []
    
    # Helper: standard 3D simplex noise evaluator [G]
    def simplex3(x, y, z):
        # Implementation of 3D simplex noise is assumed [G]
        pass
    
    # Helper: Chunk data structure [S2, unofficial]
    class Chunk:
        def __init__(self, size):
            self.voxels = [[0] * size for _ in range(size)]
            self.size = size
        def set_voxel(self, lx, ly, lz, value):
            self.voxels[lx][ly][lz] = value
        def rebuild_mesh(self):
            pass # Mesh generation logic [G]
    
    # Iterate over horizontal plane [S1, unofficial]
    for x in range(world_size):
        for z in range(world_size):
            # Calculate height using simplex3 with specific scaling [S1, unofficial]
            height_noise_x = x / noise_scale
            height_noise_y = x / (2 * noise_scale)
            height_noise_z = z / noise_scale
            
            height = simplex3(height_noise_x, height_noise_y, height_noise_z) * world_size
            
            # Iterate vertical column [S1, unofficial]
            for y in range(int(height)):
                # Water level check [S1, unofficial]
                if y < water_level:
                    continue
                
                # Density calculation [S1, unofficial]
                density_noise_x = x / noise_scale
                density_noise_y = y / noise_scale
                density_noise_z = z / noise_scale
                
                density = simplex3(density_noise_x, density_noise_y, density_noise_z)
                
                # Block placement check [S1, unofficial]
                if density > density_threshold:
                    # Calculate chunk coordinates and local index [S2, unofficial]
                    cx, cy, cz = x // chunk_size, y // chunk_size, z // chunk_size
                    chunk_id = (cx, cy, cz)
                    lx, ly, lz = x % chunk_size, y % chunk_size, z % chunk_size
                    
                    if chunk_id not in chunks:
                        chunks[chunk_id] = Chunk(chunk_size)
                        active_chunks.append(chunk_id)
                    
                    chunks[chunk_id].set_voxel(lx, ly, lz, 1) # 1 represents solid block

    # Rebuild meshes for all active chunks [S1, unofficial]
    for chunk_id in active_chunks:
        chunks[chunk_id].rebuild_mesh()
```

**Edge Cases and Pitfalls:**
- **Precision:** Simplex noise values are typically in `[-1, 1]`. The height calculation multiplies by `worldSize`, so the noise amplitude directly scales the world height [S1, unofficial].
- **Water Level:** Blocks below `waterlevel` are skipped entirely [S1, unofficial].
- **Density Threshold:** A threshold of `0.5` is used in the source. Since simplex noise is centered around 0, this creates sparse terrain [S1, unofficial].

## Subtopics

### Heightmap-based Generation
Uses a 2D noise function to determine height, then fills columns. Simpler but cannot generate overhangs or caves [G]. Choose when performance is critical and terrain is flat [G].

### Fractal Brownian Motion (FBM)
Uses octaved noise for more detailed terrain [G]. Choose when higher visual fidelity is needed [G]. Adds complexity to noise evaluation [G].

### Marching Cubes
Converts voxel data to smooth meshes [G]. Choose when smooth surfaces are required instead of blocky voxels [G].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Performance:** Not given in the sources.
- **Strengths:** Simple to implement; allows for caves and overhangs; easy to modify terrain by adjusting noise parameters [G].
- **Limitations:** Noisy terrain at high frequencies; requires careful tuning of noise scales and thresholds; memory usage can be high for large worlds if not chunked efficiently [G].
- **When to Use:** Small to medium worlds; educational purposes; games with blocky aesthetics [G].
- **When Not to Use:** Large-scale worlds requiring high performance; smooth terrain requirements; complex geological features [G].

## Background
This method originates from early WebGL voxel engines, specifically the "vox2" project by Nergal (2014) [S1, unofficial]. It was designed as a proof-of-concept for using bit-operations to reduce memory usage in JavaScript [S1, unofficial]. The technique uses 3D simplex noise for both height and density, a common approach in procedural generation to create natural-looking terrain with internal structures [G].

## Sources
[S1] Show HN: Basic voxel engine in WebGL/JavaScript with full source, https://github.com/Lallassu/vox2, fetched 2026-10-26 (unofficial)
[S2] StanislavPetrovV/Minecraft, https://github.com/StanislavPetrovV/Minecraft, fetched 2026-10-26 (unofficial)
