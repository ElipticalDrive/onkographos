# Chunk-based Voxel World Generation

Also called: Density-based voxel placement, Simplex noise terrain generation, Terrain Generation

## Overview
This method generates a 3D voxel world by sampling 3D simplex noise to determine surface height and internal density, filtering blocks against water level and density thresholds. It differs from sibling methods by using a separate noise function for vertical density rather than a single heightmap, allowing for overhangs and caves, while maintaining a simple threshold-based placement logic. It builds on [Chunk-Based Voxel Terrain Generation](4140-chunk-based-voxel-terrain-generation.md) by defining the specific procedural noise sampling and block placement rules used within each chunk.

## How it works
**Inputs:**
- `worldSize`: Total world dimensions in blocks (integer).
- `chunkSize`: Blocks per chunk dimension (integer).
- `blockSize`: World units per block (float).
- `waterlevel`: Y-coordinate threshold below which blocks are not generated (float).
- `density_threshold`: Minimum simplex noise value to place a block (float).
- `noise_scale_x`, `noise_scale_y`: Divisors for noise coordinate scaling (float).
- `seed`: Random seed for noise initialization.

**Outputs:**
- A set of active chunks, each containing a 3D array of voxel IDs (block types).

**Data Structures:**
- `Chunk`: Contains a 3D array `voxels` of size `[chunkSize, chunkSize, chunkSize]` of type `uint8` (or equivalent block ID type).
- `World`: Contains a map or array of `Chunk` objects indexed by chunk coordinates.
- `Block`: Represents a single voxel with position `(x, y, z)` and color/type.

**Parameters (Typical Values from [S1, unofficial]):**
- `worldSize = 192`
- `chunkSize = 16`
- `blockSize = 0.1`
- `waterlevel = 0`
- `density_threshold = 0.5`
- `noise_scale_x = 100`
- `noise_scale_y = 100`

**Steps:**
1. Initialize the noise generator with a random seed.
2. Initialize the world data structure (e.g., a map of chunk IDs to Chunk objects).
3. Iterate through every block coordinate `(x, z)` in the horizontal plane from `0` to `worldSize - 1`.
4. Calculate the surface height `H` for column `(x, z)` using a 3D simplex noise function:
   $$ H = \text{simplex3}\left(\frac{x}{\text{noise\_scale\_x}}, \frac{x}{2 \cdot \text{noise\_scale\_x}}, \frac{z}{\text{noise\_scale\_x}}\right) \cdot \text{worldSize} $$
   Note: The second coordinate uses `x/200` in the source, which is `x/(2*100)` [S1, unofficial].
5. Iterate through every block coordinate `y` from `0` to `H - 1`.
6. Check if `y >= waterlevel`. If false, skip this block.
7. Calculate the internal density `D` for block `(x, y, z)` using a separate 3D simplex noise function:
   $$ D = \text{simplex3}\left(\frac{x}{\text{noise\_scale\_y}}, \frac{y}{\text{noise\_scale\_y}}, \frac{z}{\text{noise\_scale\_y}}\right) $$
   Note: The source uses `noise_scale_y` (100) for all axes in the density calculation [S1, unofficial].
8. If `D > density_threshold`, mark block `(x, y, z)` as solid.
9. After iterating all `x, z, y`, group the solid blocks into chunks based on their coordinates.
10. For each chunk containing solid blocks, rebuild its mesh or finalize its data structure.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_world(world_size, chunk_size, block_size, water_level, density_threshold, noise_scale):
    # Initialize noise generator
    noise.seed(random())
    
    # Data structures
    chunks = {}  # Map from chunk_id to Chunk object
    # Chunk object structure: voxels[x][y][z] = block_id
    
    # Helper to get chunk ID
    def get_chunk_id(x, y, z):
        cx = x // chunk_size
        cy = y // chunk_size
        cz = z // chunk_size
        return (cx, cy, cz)
    
    # Helper to get local voxel index
    def get_local_index(x, y, z):
        lx = x % chunk_size
        ly = y % chunk_size
        lz = z % chunk_size
        return (lx, ly, lz)

    # Iterate over horizontal plane
    for x in range(world_size):
        for z in range(world_size):
            # Calculate height using simplex3 with specific scaling
            # Note: Source uses x/100, x/200, z/100 for height
            height_noise_x = x / noise_scale
            height_noise_y = x / (2 * noise_scale)
            height_noise_z = z / noise_scale
            
            height = simplex3(height_noise_x, height_noise_y, height_noise_z) * world_size
            
            # Iterate vertical column
            for y in range(int(height)):
                # Water level check
                if y < water_level:
                    continue
                
                # Density calculation
                # Source uses x/100, y/100, z/100 for density
                density_noise_x = x / noise_scale
                density_noise_y = y / noise_scale
                density_noise_z = z / noise_scale
                
                density = simplex3(density_noise_x, density_noise_y, density_noise_z)
                
                # Block placement check
                if density > density_threshold:
                    chunk_id = get_chunk_id(x, y, z)
                    if chunk_id not in chunks:
                        chunks[chunk_id] = Chunk(chunk_size)
                    
                    local_idx = get_local_index(x, y, z)
                    chunks[chunk_id].set_voxel(local_idx, BLOCK_SOLID)

    # Rebuild meshes for all active chunks
    for chunk_id, chunk in chunks.items():
        chunk.rebuild_mesh()
```

**Edge Cases and Pitfalls:**
- **Precision:** Simplex noise values are typically in `[-1, 1]`. The height calculation multiplies by `worldSize`, so the noise amplitude directly scales the world height. Ensure the noise function returns normalized values [S1, unofficial].
- **Boundaries:** The loops run from `0` to `worldSize`. Ensure chunk indexing handles the last chunk correctly if `worldSize` is not a multiple of `chunkSize` [S1, unofficial].
- **Water Level:** Blocks below `waterlevel` are skipped entirely. This creates a flat bottom or empty space below sea level, depending on implementation [S1, unofficial].
- **Density Threshold:** A threshold of `0.5` is used in the source. Since simplex noise is centered around 0, this creates sparse terrain. Adjusting this threshold changes the density of the world significantly [S1, unofficial].

## Subtopics

### Heightmap-based Generation
Uses a 2D noise function to determine height, then fills columns. Simpler but cannot generate overhangs or caves. Choose when performance is critical and terrain is flat [S1, unofficial].

### Fractal Brownian Motion (FBM)
Uses octaved noise for more detailed terrain. Chooses when higher visual fidelity is needed. Adds complexity to noise evaluation [S1, unofficial].

### Marching Cubes
Converts voxel data to smooth meshes. Chooses when smooth surfaces are required instead of blocky voxels [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(W^2 \cdot H)$, where $W$ is world size and $H$ is average height. Each block is evaluated once.
- **Memory Complexity:** $O(W^2 \cdot H)$ for storing voxel data, but optimized by chunking and sparse storage (only storing active chunks) [S1, unofficial].
- **Performance:** The JavaScript implementation in [S1, unofficial] is a proof-of-concept and may be slow for large worlds due to lack of optimization. The Python implementation in [S2, unofficial] uses more structured data but still iterates all blocks.
- **Strengths:** Simple to implement; allows for caves and overhangs; easy to modify terrain by adjusting noise parameters.
- **Limitations:** Noisy terrain at high frequencies; requires careful tuning of noise scales and thresholds; memory usage can be high for large worlds if not chunked efficiently.
- **When to Use:** Small to medium worlds; educational purposes; games with blocky aesthetics.
- **When Not to Use:** Large-scale worlds requiring high performance; smooth terrain requirements; complex geological features.

## Background
This method originates from early WebGL voxel engines, specifically the "vox2" project by Nergal (2014) [S1, unofficial]. It was designed as a proof-of-concept for using bit-operations to reduce memory usage in JavaScript. The technique uses 3D simplex noise for both height and density, a common approach in procedural generation to create natural-looking terrain with internal structures. The method has been adapted in various open-source projects, such as the Minecraft clone by StanislavPetrovV [S2, unofficial].

## Sources
[S1] Show HN: Basic voxel engine in WebGL/JavaScript with full source, https://github.com/Lallassu/vox2, fetched 2026-10-26 (unofficial)
[S2] StanislavPetrovV/Minecraft, https://github.com/StanislavPetrovV/Minecraft, fetched 2026-10-26 (unofficial)
