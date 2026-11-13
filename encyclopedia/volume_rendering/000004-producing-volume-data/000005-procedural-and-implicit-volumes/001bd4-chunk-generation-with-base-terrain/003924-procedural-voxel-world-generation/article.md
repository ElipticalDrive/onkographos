# Procedural Voxel World Generation

Also called: Voxel World Shader, Voxel Type Assignment Logic, Voxel ID Assignment with Caves and Biomes

## Overview
Procedural Voxel World Generation is a method that deterministically assigns a block type (material ID) to every voxel in a 3D coordinate space based on its position, height, and noise-derived parameters [S1, S3, unofficial]. It solves the problem of creating complex, varied terrain features—including surface layers, underground caves, trees, and clouds—without hard-coding specific coordinates [S1, S4, unofficial]. Unlike sibling methods that may rely on heightmaps or separate chunk-based meshing, this approach evaluates a single function per voxel to return a material index, often implemented as a GPU shader for real-time generation [S1, S3, unofficial]. It builds on [Chunk Generation with Base Terrain](1110-chunk-generation-with-terrain.md) by providing the specific logic for what fills those chunks.

## How it works
### Inputs and Outputs
- **Inputs**:
  - `cell`: A 3D integer coordinate `(x, y, z)` representing the voxel position in game units [S1, S3, unofficial].
  - `QUALITY`: An integer parameter controlling generation complexity and detail (typically 0, 1, or 2) [S1, unofficial].
  - Configuration flags: `ENABLE_TREES`, `ENABLE_CAVES`, `ENABLE_CLOUDS`, `BLOCK_LIGHT`, `ENDLESS_LOWER_HEIGHT` [S1, unofficial].
  - Constants: `CLOUD_HEIGHT`, `world_height` (implicit or explicit boundary) [S1, S2, unofficial].
- **Outputs**:
  - An integer `block_id` representing the material type. `0` indicates air/empty space [S1, S3, unofficial].

### Data Structures
The method relies on pseudo-random hash functions and noise functions.
- **Hash Functions**:
  - `hash12(vec2 p)`: Returns a float in `[0, 1)`. Implementation: `fract(sin(dot(p, vec2(127.1, 311.7))) * 43758.5453123)` [S1, unofficial].
  - `hash13(vec3 p)`: Returns a float in `[0, 1)`. Implementation: `fract(sin(dot(p, vec3(127.1, 311.7, 74.7))) * 43758.5453123)` [S1, unofficial].
- **Noise Functions**:
  - `fbm3(vec3 p)`: Fractional Brownian Motion in 3D, typically summing octaves of value noise with Hermite smoothing and trilinear interpolation [S1, unofficial].
  - `noise3`, `noise2`: Single-octave wrappers or specific dimensional variants used for cave and biome noise [S1, S2, unofficial].

### Block ID Mapping
The following integer IDs are used for material assignment [S1, unofficial]:
- `0`: Air
- `1`: Grass
- `2`: Dirt
- `3`: Stone
- `4`: Wood (Trunk)
- `5`: Leaves
- `6`: Emissive Block
- `7`: Cloud

### Algorithm Steps
The generation logic branches based on the `QUALITY` setting.

#### 1. Low Quality Mode (`QUALITY == 0`)
This mode uses stochastic placement without complex terrain height calculation [S1, unofficial].
1.  **Density Calculation**: Compute `r = hash13(vec3(x*0.12, y*0.08, z*0.15))` [S1, unofficial].
2.  **Threshold Check**: Calculate `density = 0.30 + 0.360 * r + 0.15` [S1, unofficial].
    - If `y >= 15`, the threshold is `0.8069`.
    - If `y < 15`, the threshold is `0.6`.
    - If `density <= threshold`, return `0` (Air) [S1, unofficial].
3.  **Emissive Check**: If `r > 0.999`, return `6` (Emissive) [S1, unofficial].
4.  **Block Selection**: Compute `pick = hash12(vec2(x*0.19, z*0.23))` [S1, unofficial].
    - If `pick < 0.33`, return `1` (Grass).
    - Else if `pick < 0.4`, return `4` (Wood).
    - Else if `pick < 0.66`, return `2` (Dirt).
    - Else, return `3` (Stone) [S1, unofficial].
5.  **Boundary**: If `y >= 15` and density was insufficient, return `0` [S1, unofficial].

#### 2. High Quality Mode (`QUALITY > 0`)
This mode generates structured terrain with height, caves, and features [S1, unofficial].

**Step 1: Clouds**
- If `ENABLE_CLOUDS` is true and `cell.y == CLOUD_HEIGHT`:
  - Compute `ch = hash12(floor(vec2(x*0.21, z*0.13) * 2.0))` [S1, unofficial].
  - If `ch > 0.74`, return `7` (Cloud) [S1, unofficial].
  - Otherwise, return `0` (Air) [S1, unofficial].
- If `ENABLE_CLOUDS` is false and `cell.y == CLOUD_HEIGHT`, return `0` [S1, unofficial].

**Step 2: Sky and Lower Bounds**
- If `cell.y >= 19`, return `0` (Air) [S1, unofficial].
- If `ENDLESS_LOWER_HEIGHT` is false and `cell.y <= -50`, return `3` (Stone) [S1, unofficial].

**Step 3: Terrain Height**
- Calculate floating-point height: `terrainH_f = proceduralTerrainHeight(x, z, QUALITY)` [S1, unofficial].
  - `proceduralTerrainHeight` uses `fbm3` with scale `s` and vertical scaling:
    - If `QUALITY == 1`: `s = 0.045`, `verticalScale = 10.0`, `base = 3.5`.
    - If `QUALITY == 2`: `s = 0.055`, `verticalScale = 11.0`, `base = 4.0`.
    - Default (e.g., 3): `s = 0.06`, `verticalScale = 12.0`, `base = 4.0` [S1, unofficial].
- Convert to integer: `terrainH = floor(terrainH_f)` [S1, unofficial].

**Step 4: Cave Generation**
- If `ENABLE_CAVES` is true:
  - Compute `caveNoise = 2.0 * fbm3(vec3(x*0.10, y*0.10, z*0.10))` [S1, unofficial].
  - If `caveNoise > 0.68` (for `QUALITY==2`) or `0.66` (for `QUALITY==1`), mark `isCave = true` [S1, unofficial].
- Else, `isCave = false` [S1, unofficial].

**Step 5: Emissive Blocks (Lighting)**
- If `BLOCK_LIGHT` is enabled:
  - Compute `rr = hash13(vec3(x*0.23, y*0.17, z*0.19))` [S1, unofficial].
  - If `isCave`:
    - Threshold `caveThresh`: `0.9975` (q=1), `0.9965` (q=2), `0.995` (q=0).
    - If `rr > caveThresh`, return `6` (Emissive) [S1, unofficial].
  - If `cell.y == terrainH`:
    - Threshold `surfThresh`: `0.9995` (q=1), `0.999` (q=2), `0.998` (q=0).
    - If `rr > surfThresh`, return `6` (Emissive) [S1, unofficial].

**Step 6: Terrain Body**
- If `cell.y <= terrainH` AND `!isCave`:
  - If `cell.y == terrainH`, return `1` (Grass) [S1, unofficial].
  - Else if `cell.y >= terrainH - 3`, return `2` (Dirt) [S1, unofficial].
  - Else, return `3` (Stone) [S1, unofficial].
- If `isCave` or `cell.y > terrainH`, proceed to feature generation (trees) or return `0` [S1, unofficial].

**Step 7: Tree Generation**
- If `ENABLE_TREES` is true:
  - Check `hasTreeAtColumn(x, z, QUALITY)` [S1, unofficial].
    - Compute `r = hash12(vec2(x*0.374, z*0.912))` [S1, unofficial].
    - Return `true` if `r > 0.995` (q=1), `0.985` (q=2), or `0.97` (q=0) [S1, unofficial].
  - If a tree exists:
    - Calculate trunk height: `trunkH = 3 + floor(hash12(vec2(x*0.11, z*0.08)) * 3.0)` [S1, unofficial].
    - `baseY = terrainH + 1` [S1, unofficial].
    - `topY = baseY + trunkH` [S1, unofficial].
    - If `baseY <= cell.y < topY`, return `4` (Wood) [S1, unofficial].

**Step 8: Leaf Placement**
- Determine search parameters:
  - `SEARCH_MAX`: `1` (q=1), `3` (q=2) [S1, unofficial].
  - `maxOffset`: `1` (q=1), `2` (q=2) [S1, unofficial].
- Iterate `ox` from `-SEARCH_MAX` to `SEARCH_MAX` and `oz` from `-SEARCH_MAX` to `SEARCH_MAX` [S1, unofficial].
- For each neighbor `(nx, nz) = (x + ox, z + oz)`:
  - If `abs(ox) > maxOffset` or `abs(oz) > maxOffset`, continue [S1, unofficial].
  - Check if `hasTreeAtColumn(nx, nz, QUALITY)` is true [S1, unofficial].
  - If true, calculate local coordinates relative to the neighbor tree's top:
    - `dx = x - nx`, `dz = z - nz`, `dy = y - topY_neighbor` [S1, unofficial].
    - Return `5` (Leaves) if:
      - `dy == 0` or `dy == 1` AND `dx` in `[-1, 2]` AND `dz` in `[-1, 2]` OR (`dx == -2` AND `dz` in `[-1, 2]`) [S1, unofficial].
      - OR `dy == 2` AND (`dx == 0` AND `dz == 0`) OR (`abs(dx) <= 1` AND `dz == 0`) OR (`abs(dz) <= 1` AND `dx == 0`) [S1, unofficial].

**Step 9: Default**
- If no block type was returned, return `0` (Air) [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def get_block_type(cell_x, cell_y, cell_z, quality, config):
    """
    Determines the block type at a given 3D coordinate.
    
    Args:
        cell_x, cell_y, cell_z: Integer world coordinates.
        quality: Integer (0, 1, 2).
        config: Dict with keys ENABLE_TREES, ENABLE_CAVES, ENABLE_CLOUDS, BLOCK_LIGHT, ENDLESS_LOWER_HEIGHT, CLOUD_HEIGHT.
        
    Returns:
        Integer block ID. 0 is air.
    """
    # Constants
    BLOCK_AIR = 0
    BLOCK_GRASS = 1
    BLOCK_DIRT = 2
    BLOCK_STONE = 3
    BLOCK_WOOD = 4
    BLOCK_LEAVES = 5
    BLOCK_EMISSIVE = 6
    BLOCK_CLOUD = 7
    
    # Helper Hash Functions
    def hash12(v):
        return fract(sin(dot(v, vec2(127.1, 311.7))) * 43758.5453123)
    
    def hash13(v):
        return fract(sin(dot(v, vec3(127.1, 311.7, 74.7))) * 43758.5453123)
        
    # Helper Noise Functions (Simplified for pseudocode)
    def fbm3(v):
        # Placeholder for actual FBM implementation
        return noise3(v) 
        
    def procedural_terrain_height(fx, fz, q):
        if q == 1:
            s, vs, base = 0.045, 10.0, 3.5
        elif q == 2:
            s, vs, base = 0.055, 11.0, 4.0
        else:
            s, vs, base = 0.06, 12.0, 4.0
            
        h = fbm3(vec3(fx * s, 0.0, fz * s))
        return h * vs + base

    def has_tree_at_column(cx, cz, q):
        if not config['ENABLE_TREES']:
            return False
        r = hash12(vec2(float(cx) * 0.374, float(cz) * 0.912))
        if q == 1: return r > 0.995
        if q == 2: return r > 0.985
        return r > 0.97

    # 1. Low Quality Mode
    if quality == 0:
        r = hash13(vec3(float(cell_x) * 0.12, float(cell_y) * 0.08, float(cell_z) * 0.15))
        density = 0.30 + 0.360 * r + 0.15
        
        threshold = 0.8069 if cell_y >= 15 else 0.6
        
        if density <= threshold:
            if cell_y >= 15:
                return BLOCK_AIR
            return BLOCK_AIR
            
        if r > 0.999:
            return BLOCK_EMISSIVE
            
        pick = hash12(vec2(float(cell_x) * 0.19, float(cell_z) * 0.23))
        if pick < 0.33: return BLOCK_GRASS
        if pick < 0.4: return BLOCK_WOOD
        if pick < 0.66: return BLOCK_DIRT
        return BLOCK_STONE

    # 2. High Quality Mode
    cloud_y = config['CLOUD_HEIGHT']
    
    # Clouds
    if config['ENABLE_CLOUDS']:
        if cell_y == cloud_y:
            ch = hash12(floor(vec2(float(cell_x) * 0.21, float(cell_z) * 0.13) * 2.0))
            if ch > 0.74:
                return BLOCK_CLOUD
            return BLOCK_AIR
    else:
        if cell_y == cloud_y:
            return BLOCK_AIR
            
    # Sky
    if cell_y >= 19:
        return BLOCK_AIR
        
    # Lower Bounds
    if not config['ENDLESS_LOWER_HEIGHT'] and cell_y <= -50:
        return BLOCK_STONE
        
    fx = float(cell_x)
    fz = float(cell_z)
    terrain_h_f = procedural_terrain_height(fx, fz, quality)
    terrain_h = floor(terrain_h_f)
    
    # Caves
    is_cave = False
    if config['ENABLE_CAVES']:
        cave_noise = 2.0 * fbm3(vec3(float(cell_x) * 0.10, float(cell_y) * 0.10, float(cell_z) * 0.10))
        cave_thresh = 0.68 if quality == 2 else 0.66
        if cave_noise > cave_thresh:
            is_cave = True
            
    # Emissive
    rr = hash13(vec3(float(cell_x) * 0.23, float(cell_y) * 0.17, float(cell_z) * 0.19))
    if config['BLOCK_LIGHT']:
        if is_cave:
            cave_light_thresh = 0.9975 if quality == 1 else (0.9965 if quality == 2 else 0.995)
            if rr > cave_light_thresh:
                return BLOCK_EMISSIVE
                
        if cell_y == terrain_h:
            surf_light_thresh = 0.9995 if quality == 1 else (0.999 if quality == 2 else 0.998)
            if rr > surf_light_thresh:
                return BLOCK_EMISSIVE
                
    # Terrain Body
    if cell_y <= terrain_h and not is_cave:
        if cell_y == terrain_h:
            return BLOCK_GRASS
        if cell_y >= terrain_h - 3:
            return BLOCK_DIRT
        return BLOCK_STONE
        
    # Trees
    if config['ENABLE_TREES'] and has_tree_at_column(cell_x, cell_z, quality):
        trunk_h = 3 + floor(hash12(vec2(float(cell_x) * 0.11, float(cell_z) * 0.08)) * 3.0)
        base_y = terrain_h + 1
        top_y = base_y + trunk_h
        
        if base_y <= cell_y < top_y:
            return BLOCK_WOOD
            
        # Leaves
        search_max = 1 if quality == 1 else 3
        max_offset = 1 if quality == 1 else 2
        
        for ox in range(-search_max, search_max + 1):
            for oz in range(-search_max, search_max + 1):
                if abs(ox) > max_offset or abs(oz) > max_offset:
                    continue
                    
                nx = cell_x + ox
                nz = cell_z + oz
                
                if has_tree_at_column(nx, nz, quality):
                    # Calculate neighbor tree top
                    n_trunk_h = 3 + floor(hash12(vec2(float(nx) * 0.11, float(nz) * 0.08)) * 3.0)
                    n_base_y = terrain_h + 1 # Note: terrain height might vary per column, but simplified here to global or neighbor's local height
                    # In strict implementation, neighbor's terrain height must be calculated
                    n_terrain_h_f = procedural_terrain_height(float(nx), float(nz), quality)
                    n_terrain_h = floor(n_terrain_h_f)
                    n_top_y = n_terrain_h + 1 + n_trunk_h
                    
                    dx = cell_x - nx
                    dz = cell_z - nz
                    dy = cell_y - n_top_y
                    
                    # Leaf logic
                    if (dy == 0 or dy == 1) and (-1 <= dx <= 2) and (-1 <= dz <= 2):
                        if not (dx == -2 and not (-1 <= dz <= 2)):
                             return BLOCK_LEAVES
                    if dy == 2:
                        if (dx == 0 and dz == 0):
                            return BLOCK_LEAVES
                        if (abs(dx) <= 1 and dz == 0):
                            return BLOCK_LEAVES
                        if (abs(dz) <= 1 and dx == 0):
                            return BLOCK_LEAVES
                            
    return BLOCK_AIR
```

### Parameters
- `QUALITY`: Integer. `0` = stochastic sparse blocks, no caves/trees/clouds. `1` = medium detail, simple trees, basic caves. `2` = high detail, complex caves, larger leaf search radius [S1, unofficial].
- `CLOUD_HEIGHT`: Integer Y-coordinate for cloud layer [S1, unofficial].
- `ENABLE_*`: Boolean flags for features [S1, unofficial].

### Edge Cases and Pitfalls
- **Division by Zero**: In touch input handling for UV calculation, resolution must be clamped to `max(1.0)` to prevent division by zero [S1, unofficial].
- **Boundary Conditions**: The algorithm assumes a specific vertical range (e.g., `y < 19` for sky, `y > -50` for endless stone). Outside these, behavior is defined by `ENDLESS_LOWER_HEIGHT` [S1, unofficial].
- **Tree Height Consistency**: Leaf placement requires checking the terrain height of *neighboring* columns, not just the current column, to correctly position leaves relative to the neighbor's tree top [S1, unofficial].

## Subtopics

### [Deterministic Fixture Worlds](3941-deterministic-fixture-worlds.md)
Generates small-scale, static geometric structures (slabs, towers) using pure functions of `VolumeId` for regression testing, avoiding RNG.

### [Biome and Voxel Layer Determination](4167-biome-and-voxel-layer-determination.md)
Assigns voxel types based on vertical position relative to `genHeight` and biome noise (moisture, temperature), creating varied surface compositions like deserts or jungles.

### [Procedural Tree Generation](4826-procedural-tree-generation.md)
Focuses specifically on placing organic tree structures (trunks and leaves) on valid ground positions, handling boundary checks and intersection avoidance.

## Operating characteristics
- **Time Complexity**: $O(1)$ per voxel evaluation. The cost is dominated by hash and noise function evaluations, which are constant time relative to world size but depend on `QUALITY` (FBM octaves).
- **Memory Complexity**: $O(1)$ auxiliary memory per voxel if evaluated on-the-fly (shader). If pre-computed, $O(N)$ where $N$ is the number of voxels in the chunk/world [S3, unofficial].
- **Performance**: Low quality (`QUALITY=0`) is extremely fast, suitable for mobile or low-end hardware. High quality (`QUALITY=2`) with caves and trees requires more noise evaluations and leaf neighbor checks, increasing CPU/GPU load [S1, unofficial].
- **Strengths**: Deterministic, parameter-driven, supports real-time generation on GPU via shaders [S1, S3, unofficial].
- **Limitations**: Stochastic low-quality mode lacks coherent terrain structure. High-quality mode can be computationally expensive due to neighbor lookups for leaves and FBM noise [S1, unofficial].

## Background
Procedural voxel world generation is a staple in voxel-based games (e.g., Minecraft). The specific implementation detailed here originates from a GLSL shader project (`Voxel-World-Shader`) designed for efficient in-shader world generation [S1, unofficial]. Other implementations, such as `voxel-engine-stackgl`, use JavaScript to pass a `generate` function to the engine, which is called per voxel [S3, unofficial]. Academic and engine projects like DigBuild structure generation into phases: constraints, biome picking, and terrain population [S4, unofficial].

## Sources
[S1] GitHub - razzle-reinizu/Voxel-World-Shader: A GLSL voxel-world shader ..., https://github.com/razzle-reinizu/Voxel-World-Shader, fetched 2026-10-25 (unofficial)
[S2] StanislavPetrovV/Minecraft, https://github.com/StanislavPetrovV/Minecraft, fetched 2026-10-26 (unofficial)
[S3] voxel/voxel-engine-stackgl, https://github.com/voxel/voxel-engine-stackgl, fetched 2026-10-27 (unofficial)
[S4] DigBuild: Sandbox Voxel Game and Engine Using Vulkan, https://gredos.usal.es/handle/10366/149914, fetched 2026-10-31 (unofficial)
