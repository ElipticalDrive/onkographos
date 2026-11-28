# Procedural Chunk Generation with Density Falloff

Also called: Floating Rock Generation, Voxel World Generation

## Overview
This method generates a voxel chunk by computing a density field using normalized coordinates, plateau falloff, cave noise, and center falloff, then assigns block types based on density thresholds and random distributions [S1, unofficial]. It differs from its parent [Procedural Voxel World Generation](3891-procedural-voxel-world-generation.md) by explicitly incorporating a radial center falloff and a specific plateau falloff mechanism to create isolated floating structures, while using a distinct cave generation threshold [S1, unofficial].

## How it works
**Inputs:**
- `world_x`, `world_y`, `world_z`: Integer coordinates of the chunk's origin in the global world space [S1, unofficial].
- `chunk_width`, `chunk_height`, `chunk_depth`: Integer dimensions of the chunk [S1, unofficial].
- `e`: A random number generator state or seed instance [S1, unofficial].

**Outputs:**
- A 3D array of block types (`block_type`) with dimensions `chunk_width` × `chunk_height` × `chunk_depth` [S1, unofficial].

**Data Structures:**
- `block_type`: An enumeration representing block states (`air`, `stone`, `dirt`, `grass`) [S1, unofficial].
- `density`: A floating-point value representing voxel solidity [S1, unofficial].

**Parameters:**
- `caves_threshold`: 0.5. Cave density below this value forces the block to air [S1, unofficial].
- `density_threshold`: 3.1. Blocks with final density below this value are set to `air` [S1, unofficial].
- `grass_dist`: Uniform real distribution over $[0.8, 0.9]$. Determines the height threshold for grass [S1, unofficial].
- `dirt_dist`: Uniform real distribution over $[0.4, 0.7]$. Determines the height threshold for dirt [S1, unofficial].
- `simplex_noise(seed, x, y, z)`: A simplex noise function returning a value in $[-1, 1]$ or $[0, 1]$ depending on implementation. The source uses `simplex_noise(1, ...)` for caves and `simplex_noise(5, ...)` for base terrain [S1, unofficial].
- `noise(x, y, z)`: A secondary noise function used for high-frequency detail. The source does not specify the seed, implying it may be stateless or derived from coordinates [S1, unofficial].

**Steps:**
1. **Normalize Coordinates:** Compute normalized world coordinates $x_f, y_f, z_f$ for the current voxel at local coordinates $(x, y, z)$:
   $$x_f = \frac{\text{world\_x} + x}{\text{world\_width} \times \text{chunk\_width}}$$
   $$y_f = \frac{\text{world\_y} + y}{\text{world\_height} \times \text{chunk\_height}}$$
   $$z_f = \frac{\text{world\_z} + z}{\text{world\_depth} \times \text{chunk\_depth}}$$
   [S1, unofficial]
2. **Compute Plateau Falloff:** Determine `plateau_falloff` based on $y_f$:
   - If $y_f \le 0.8$, `plateau_falloff` = 1.0 [S1, unofficial].
   - If $0.8 < y_f < 0.9$, `plateau_falloff` = $1.0 - (y_f - 0.8) \times 10.0$ [S1, unofficial].
   - If $y_f \ge 0.9$, `plateau_falloff` = 0.0 [S1, unofficial].
3. **Compute Cave Density:** Calculate cave noise:
   $$\text{caves} = \text{simplex\_noise}(1, x_f \times 5, y_f \times 5, z_f \times 5)^3$$
   If `caves` < 0.5, set `density` = 0 and skip to Step 7 [S1, unofficial]. Otherwise, proceed.
4. **Compute Center Falloff:** Calculate radial distance from the chunk center $(0.5, 1.0, 0.5)$:
   $$\text{center\_falloff} = \frac{0.1}{\text{pow}((x_f - 0.5) \times 1.5, 2) + \text{pow}((y_f - 1.0) \times 0.8, 2) + \text{pow}((z_f - 0.5) \times 1.5, 2)}$$
   [S1, unofficial]
5. **Compute Base Density:**
   $$\text{density} = \text{simplex\_noise}(5, x_f, y_f \times 0.5, z_f) \times \text{center\_falloff} \times \text{plateau\_falloff}$$
   [S1, unofficial]
6. **Apply High-Frequency Detail:**
   $$\text{density} = \text{density} \times \text{pow}(\text{noise}((x_f + 1) \times 3.0, (y_f + 1) \times 3.0, (z_f + 1) \times 3.0) + 0.4, 1.8)$$
   [S1, unofficial]
7. **Assign Block Type:**
   - If `density` < 3.1, `type` = `air` [S1, unofficial].
   - Else if $y_f > \text{grass\_dist}(e)$, `type` = `grass` [S1, unofficial].
   - Else if $y_f > \text{dirt\_dist}(e)$, `type` = `dirt` [S1, unofficial].
   - Else, `type` = `stone` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Define block type enumeration
class block_type:
    air = 0
    stone = 1
    dirt = 2
    grass = 3

# Define noise helpers as standard mathematical noise functions [G]
def simplex_noise(seed, x, y, z):
    # Standard simplex noise implementation returning float in [-1, 1]
    return 0.0 # Placeholder for actual simplex noise algorithm [G]

def noise(x, y, z):
    # Standard secondary noise implementation returning float in [-1, 1]
    return 0.0 # Placeholder for actual noise algorithm [G]

def generate_chunk(world_x, world_y, world_z, world_width, world_height, world_depth, chunk_width, chunk_height, chunk_depth, rng):
    """
    Generates a voxel chunk using procedural density falloff.
    """
    # Initialize chunk with air
    chunk = [[[block_type.air for _ in range(chunk_depth)] 
              for _ in range(chunk_height)] 
              for _ in range(chunk_width)]
    
    for z in range(chunk_depth):
        for y in range(chunk_height):
            for x in range(chunk_width):
                # Step 1: Normalize coordinates
                xf = (world_x + x) / (world_width * chunk_width)
                yf = (world_y + y) / (world_height * chunk_height)
                zf = (world_z + z) / (world_depth * chunk_depth)
                
                # Step 2: Plateau falloff
                if yf <= 0.8:
                    plateau_falloff = 1.0
                elif yf < 0.9:
                    plateau_falloff = 1.0 - (yf - 0.8) * 10.0
                else:
                    plateau_falloff = 0.0
                
                # Step 3: Cave density
                caves = simplex_noise(1, xf * 5, yf * 5, zf * 5) ** 3
                if caves < 0.5:
                    density = 0.0
                else:
                    # Step 4: Center falloff
                    dx = (xf - 0.5) * 1.5
                    dy = (yf - 1.0) * 0.8
                    dz = (zf - 0.5) * 1.5
                    center_falloff = 0.1 / (dx**2 + dy**2 + dz**2)
                    
                    # Step 5: Base density
                    base_noise = simplex_noise(5, xf, yf * 0.5, zf)
                    density = base_noise * center_falloff * plateau_falloff
                    
                    # Step 6: High-frequency detail
                    detail_noise = noise((xf + 1) * 3.0, (yf + 1) * 3.0, (zf + 1) * 3.0)
                    density *= (detail_noise + 0.4) ** 1.8
                
                # Step 7: Assign block type
                if density < 3.1:
                    block = block_type.air
                else:
                    grass_threshold = rng.uniform(0.8, 0.9)
                    dirt_threshold = rng.uniform(0.4, 0.7)
                    
                    if yf > grass_threshold:
                        block = block_type.grass
                    elif yf > dirt_threshold:
                        block = block_type.dirt
                    else:
                        block = block_type.stone
                
                chunk[x][y][z] = block
                
    return chunk
```
[G] The pseudocode defines `block_type` as an enumeration, passes `world_width`, `world_height`, `world_depth` as parameters to match the evidence's denominator, and implements `simplex_noise` and `noise` as standard mathematical noise functions. Random distribution calls `grass_dist(e)` and `dirt_dist(e)` are implemented as `rng.uniform(0.8, 0.9)` and `rng.uniform(0.4, 0.7)` respectively [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Division by Zero:** In Step 4, if $x_f=0.5, y_f=1.0, z_f=0.5$, the denominator for `center_falloff` becomes 0 [S1, unofficial]. This results in infinity or NaN. In practice, this occurs only at the exact center of the chunk. A guard clause or epsilon addition is required to prevent NaN propagation [G].
- **Negative Density:** `simplex_noise` can return negative values. If `base_noise` is negative, `density` becomes negative, which will likely result in `air` blocks [S1, unofficial].
- **Precision:** The use of `pow` and multiple noise evaluations can accumulate floating-point errors. Not given in the sources.

## Subtopics

### [Deterministic Fixture Worlds](3941-deterministic-fixture-worlds.md)
Generates fixed, non-procedural structures. Choose when static, hand-crafted layouts are needed.

### [Biome and Voxel Layer Determination](4167-biome-and-voxel-layer-determination.md)
Assigns block types based on biome data. Choose when environmental variation is required.

### [Procedural Tree Generation](4826-procedural-tree-generation.md)
Creates tree structures. Choose when vegetation is needed.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Performance:** Not given in the sources.
- **Strengths:** Not given in the sources.
- **Limitations:** Not given in the sources.
- **When to Use:** Not given in the sources.
- **When Not to Use:** Not given in the sources.

## Background
Not given in the sources.

## Sources
[S1] jshrake/lexov, https://github.com/jshrake/lexov, fetched 2026-10-27 (unofficial)
