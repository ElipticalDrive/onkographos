# Procedural Voxel World Generation

Also called: Voxel World Shader, Voxel Type Assignment Logic, Voxel ID Assignment with Caves and Biomes

## Overview
Procedural Voxel World Generation determines the material type of every voxel in a 3D coordinate space using deterministic functions, typically evaluated per-chunk or per-voxel [S1, S3, S4, unofficial]. It solves the problem of creating infinite, varied terrain without storing explicit mesh data by mapping coordinates to block IDs [S3, unofficial]. It differs from [Procedural Terrain Heightmap Generation](1850-procedural-terrain-heightmap-generation.md) by explicitly handling 3D volume features like caves and internal layering rather than just surface height, and from [Procedural Cave Generation](2579-procedural-cave-generation.md) by integrating cave logic into the primary block-type decision pipeline rather than treating it as a separate post-processing step [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** 3D integer coordinates $(x, y, z)$ representing the voxel position in world space; a `QUALITY` integer parameter controlling generation complexity; boolean flags for features (e.g., `ENABLE_CAVES`, `ENABLE_TREES`) [S1, S3, unofficial].
*   **Output:** An integer `voxel_id` representing the block type, or 0 for air [S1, S3, unofficial].

**Data Structures:**
*   **Block ID Map:**
    *   `0`: Air
    *   `1`: Grass
    *   `2`: Dirt
    *   `3`: Stone
    *   `4`: Wood
    *   `5`: Leaves
    *   `6`: Emissive Block
    *   `7`: Cloud [S1, unofficial]

**Hash and Noise Functions:**
The method relies on deterministic pseudo-random functions.
*   `hash12(vec2 p)`: Returns a float in $[0, 1)$. Implementation: `fract(sin(dot(p, vec2(127.1, 311.7))) * 43758.5453123)` [S1, unofficial].
*   `hash13(vec3 p)`: Returns a float in $[0, 1)$. Implementation: `fract(sin(dot(p, vec3(127.1, 311.7, 74.7))) * 43758.5453123)` [S1, unofficial].
*   `proceduralTerrainHeight(fx, fz, q)`: Calculates surface height.
    *   Scale $s$: $0.045$ if $q=1$, $0.055$ if $q=2$, $0.06$ otherwise.
    *   Vertical Scale $v$: $10.0$ if $q=1$, $11.0$ if $q=2$, $12.0$ otherwise.
    *   Base $b$: $3.5$ if $q=1$, $4.0$ otherwise.
    *   Formula: $h = \text{fbm3}(vec3(fx \cdot s, 0.0, fz \cdot s))$. Result: $h \cdot v + b$ [S1, unofficial].
*   `fbm3`: Fractal Brownian Motion 3D noise (value noise with Hermite smoothing and trilinear interpolation) [S1, unofficial].

**Algorithm Steps:**
The generation logic branches based on the `QUALITY` setting.

1.  **Low Quality (QUALITY == 0):** Stochastic sparse generation.
    *   Calculate density: $r = \text{hash13}(vec3(x \cdot 0.12, y \cdot 0.08, z \cdot 0.15))$.
    *   Density value: $D = 0.30 + 0.360 \cdot r + 0.15$.
    *   Threshold $T$: $0.8069$ if $y \ge 15$, else $0.6$.
    *   If $D > T$:
        *   If $r > 0.999$, return `BLOCK_EMISSIVE` (6) [S1, unofficial].
        *   Calculate pick: $p = \text{hash12}(vec2(x \cdot 0.19, z \cdot 0.23))$.
        *   If $p < 0.33$, return `1` (Grass).
        *   Else if $p < 0.4$, return `4` (Wood).
        *   Else if $p < 0.66$, return `2` (Dirt).
        *   Else, return `3` (Stone) [S1, unofficial].
    *   If $y \ge 15$, return `0` (Air).
    *   Return `0` (Air) [S1, unofficial].

2.  **High Quality (QUALITY > 0):** Structured terrain.
    *   **Clouds:** If `ENABLE_CLOUDS` is true and $y == \text{CLOUD\_HEIGHT}$:
        *   $c = \text{hash12}(floor(vec2(x \cdot 0.21, z \cdot 0.13) \cdot 2.0))$.
        *   If $c > 0.74$, return `BLOCK_CLOUD` (7).
        *   Else return `0` (Air) [S1, unofficial].
    *   **Sky:** If $y \ge 19$, return `0` (Air) [S1, unofficial].
    *   **Terrain Height:** Calculate $terrainH = \lfloor \text{proceduralTerrainHeight}(x, z, q) \rfloor$ [S1, unofficial].
    *   **Caves:** If `ENABLE_CAVES` is true:
        *   $cn = 2.0 \cdot \text{fbm3}(vec3(x \cdot 0.10, y \cdot 0.10, z \cdot 0.10))$.
        *   Threshold $C_{th}$: $0.68$ if $q=2$, else $0.66$.
        *   If $cn > C_{th}$, mark `isCave = true` [S1, unofficial].
    *   **Emissive Blocks:** If `BLOCK_LIGHT` is enabled:
        *   $rr = \text{hash13}(vec3(x \cdot 0.23, y \cdot 0.17, z \cdot 0.19))$.
        *   If `isCave`: Threshold $E_{cave}$ is $0.9975$ ($q=1$), $0.9965$ ($q=2$), $0.995$ ($q=0$). If $rr > E_{cave}$, return `BLOCK_EMISSIVE` (6) [S1, unofficial].
        *   If $y == terrainH$: Threshold $E_{surf}$ is $0.9995$ ($q=1$), $0.999$ ($q=2$), $0.998$ ($q=0$). If $rr > E_{surf}$, return `BLOCK_EMISSIVE` (6) [S1, unofficial].
    *   **Terrain Body:** If $y \le terrainH$ AND NOT `isCave`:
        *   If $y == terrainH$, return `1` (Grass).
        *   Else if $y \ge terrainH - 3$, return `2` (Dirt).
        *   Else, return `3` (Stone) [S1, unofficial].
    *   **Trees:** If `ENABLE_TREES` is true:
        *   Check `hasTreeAtColumn(x, z, q)`.
        *   If true:
            *   $trunkH = 3 + \lfloor \text{hash12}(vec2(x \cdot 0.11, z \cdot 0.08)) \cdot 3.0 \rfloor$.
            *   $baseY = terrainH + 1$.
            *   $topY = baseY + trunkH$.
            *   If $baseY \le y < topY$, return `4` (Wood) [S1, unofficial].
            *   **Leaves:** Determine search radius: $SEARCH\_MAX = 1$ ($q=1$), $3$ ($q=2$). $maxOffset = 1$ ($q=1$), $2$ ($q=2$).
            *   Iterate $ox$ from $-SEARCH\_MAX$ to $SEARCH\_MAX$, $oz$ from $-SEARCH\_MAX$ to $SEARCH\_MAX$.
            *   If $|ox| > maxOffset$ or $|oz| > maxOffset$, continue.
            *   Check if neighbor column $(x+ox, z+oz)$ has a tree. If not, continue.
            *   Calculate local offsets relative to the neighbor tree's top.
            *   Return `5` (Leaves) if:
                *   $dy \in \{0, 1\}$ AND $dx \in [-1, 2]$ AND $dz \in [-1, 2]$ OR ($dx=-2$ AND $dz \in [-1, 2]$).
                *   $dy = 2$ AND ($dx=0, dz=0$) OR ($|dx| \le 1, dz=0$) OR ($|dz| \le 1, dx=0$) [S1, unofficial].
    *   **Endless Lower Height:** If `ENDLESS_LOWER_HEIGHT` is false and $y \le -50$, return `3` (Stone) [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def get_block_type(x: int, y: int, z: int, quality: int, settings: dict) -> int:
    # Constants
    BLOCK_AIR = 0
    BLOCK_GRASS = 1
    BLOCK_DIRT = 2
    BLOCK_STONE = 3
    BLOCK_WOOD = 4
    BLOCK_LEAVES = 5
    BLOCK_EMISSIVE = 6
    BLOCK_CLOUD = 7
    
    # Helper Hashes
    def hash12(px, pz):
        return fract(sin(px * 127.1 + pz * 311.7) * 43758.5453123)
        
    def hash13(px, py, pz):
        return fract(sin(px * 127.1 + py * 311.7 + pz * 74.7) * 43758.5453123)

    # Low Quality Path
    if quality == 0:
        r = hash13(x * 0.12, y * 0.08, z * 0.15)
        density = 0.30 + 0.360 * r + 0.15
        threshold = 0.8069 if y >= 15 else 0.6
        
        if density > threshold:
            if r > 0.999:
                return BLOCK_EMISSIVE
            pick = hash12(x * 0.19, z * 0.23)
            if pick < 0.33: return BLOCK_GRASS
            if pick < 0.4: return BLOCK_WOOD
            if pick < 0.66: return BLOCK_DIRT
            return BLOCK_STONE
            
        if y >= 15:
            return BLOCK_AIR
        return BLOCK_AIR

    # High Quality Path
    # Clouds
    if settings.get('ENABLE_CLOUDS', False) and y == settings.get('CLOUD_HEIGHT', 10):
        ch = hash12(floor(x * 0.21) * 2.0, floor(z * 0.13) * 2.0) # Note: Source uses vec2 multiplication then floor, approximated here
        # Correct interpretation from source: hash12(floor(vec2(x*0.21, z*0.13) * 2.0))
        ch = hash12(floor(x * 0.42), floor(z * 0.26))
        if ch > 0.74:
            return BLOCK_CLOUD
        return BLOCK_AIR

    if y >= 19:
        return BLOCK_AIR

    # Terrain Height
    terrain_h = floor(procedural_terrain_height(x, z, quality))

    # Caves
    is_cave = False
    if settings.get('ENABLE_CAVES', False):
        cave_noise = 2.0 * fbm3(x * 0.10, y * 0.10, z * 0.10)
        cave_thresh = 0.68 if quality == 2 else 0.66
        if cave_noise > cave_thresh:
            is_cave = True

    # Emissive
    if settings.get('BLOCK_LIGHT', False):
        rr = hash13(x * 0.23, y * 0.17, z * 0.19)
        if is_cave:
            e_thresh = 0.9975 if quality == 1 else (0.9965 if quality == 2 else 0.995)
            if rr > e_thresh:
                return BLOCK_EMISSIVE
        if y == terrain_h:
            s_thresh = 0.9995 if quality == 1 else (0.999 if quality == 2 else 0.998)
            if rr > s_thresh:
                return BLOCK_EMISSIVE

    # Terrain Body
    if y <= terrain_h and not is_cave:
        if y == terrain_h:
            return BLOCK_GRASS
        if y >= terrain_h - 3:
            return BLOCK_DIRT
        return BLOCK_STONE

    # Trees
    if settings.get('ENABLE_TREES', False):
        if has_tree_at_column(x, z, quality):
            trunk_h = 3 + floor(hash12(x * 0.11, z * 0.08) * 3.0)
            base_y = terrain_h + 1
            top_y = base_y + trunk_h
            
            if base_y <= y < top_y:
                return BLOCK_WOOD
            
            # Leaves
            search_max = 1 if quality == 1 else 3
            max_offset = 1 if quality == 1 else 2
            
            for ox in range(-search_max, search_max + 1):
                for oz in range(-search_max, search_max + 1):
                    if abs(ox) > max_offset or abs(oz) > max_offset:
                        continue
                    
                    # Check neighbor tree
                    if not has_tree_at_column(x + ox, z + oz, quality):
                        continue
                    
                    # Calculate relative position to neighbor tree top
                    # Neighbor tree top is at (terrain_h_neighbor + 1 + trunk_h_neighbor)
                    # We approximate neighbor terrain height as current terrain_h for simplicity in this dense check,
                    # or strictly follow source logic which implies checking against the specific neighbor's structure.
                    # Source logic: "Calculate local dx, dz, dy relative to neighbor tree top."
                    # Since we don't have the neighbor's exact height in this loop without recursion/caching,
                    # we assume the source implies a simplified check or that the loop is for the CURRENT tree's leaves
                    # if the source logic "neighbor column has tree" is a proxy for "is this cell under a tree canopy".
                    # However, the source explicitly says "Check if neighbor column has tree".
                    # To implement strictly: we must evaluate the neighbor's tree.
                    
                    # Simplified Leaf Logic based on source heuristics:
                    # dy=0 or 1 and dx in [-1,2], dz in [-1,2] or (dx=-2, dz in [-1,2])
                    # dy=2 and (dx=0,dz=0) or (abs(dx)<=1,dz=0) or (abs(dz)<=1,dx=0)
                    
                    # Note: The source pseudocode for leaves is complex and relies on `neighbor tree top`.
                    # In a single-voxel evaluation function, this usually requires passing the local terrain height
                    # or evaluating the neighbor's height.
                    
                    # For the purpose of this article, we note the heuristic conditions.
                    # A full implementation would require `get_terrain_height(x+ox, z+oz)` and `has_tree_at_column(x+ox, z+oz)`.
                    
                    return BLOCK_LEAVES # Placeholder for complex leaf logic defined in source [S1, unofficial]

    # Endless Lower Height
    if not settings.get('ENDLESS_LOWER_HEIGHT', True) and y <= -50:
        return BLOCK_STONE

    return BLOCK_AIR

def procedural_terrain_height(fx, fz, q):
    s = 0.045 if q == 1 else (0.055 if q == 2 else 0.06)
    h = fbm3(fx * s, 0.0, fz * s)
    v = 10.0 if q == 1 else (11.0 if q == 2 else 12.0)
    b = 3.5 if q == 1 else 4.0
    return h * v + b

def has_tree_at_column(cx, cz, quality):
    if not settings.get('ENABLE_TREES', False):
        return False
    r = hash12(cx * 0.374, cz * 0.912)
    if quality == 1: return r > 0.995
    if quality == 2: return r > 0.985
    return r > 0.97
```

## Subtopics

### [Deterministic Fixture Worlds](3941-deterministic-fixture-worlds.md)
Deterministic Fixture Worlds are pure functions that generate specific, small-scale voxel structures based solely on a `VolumeId`, ensuring no reliance on random number generators (RNG), system clocks, or I/O. They serve as regression tests and acceptance criteria for voxel engines, providing predictable geometric primitives like slabs, towers, and bridges. Unlike general procedural generation which uses noise for terrain, fixture worlds are hand-crafted logical shapes defined by exact brick coordinates. They build on [Procedural Voxel World Generation](3891-procedural-voxel-world-generation.md) and sit alongside [Biome and Voxel Layer Determination](4167-biome-and-voxel-layer-determination.md) and [Procedural Tree Generation](4826-procedural-tree-generation.md).

### [Biome and Voxel Layer Determination](4167-biome-and-voxel-layer-determination.md)
This method assigns the type of each voxel in a procedural world based on its vertical position relative to the generated terrain surface (`genHeight`) and biome parameters derived from noise functions (moisture, temperature, wackiness). It solves the problem of creating varied, biologically plausible terrain structures (such as underwater layers, deserts, and jungles) without hard-coding specific coordinates. It differs from its sibling [Procedural Tree Generation](4826-procedural-tree-generation.md) by focusing on static ground composition and layering rather than organic structure placement.

### [Procedural Tree Generation](4826-procedural-tree-generation.md)
Procedural tree generation populates a voxel world with vegetation by algorithmically placing trunk and leaf blocks at valid surface coordinates. It differs from [Biome and Voxel Layer Determination](4167-biome-and-voxel-layer-determination.md) by operating on top of the static ground layer to inject vertical structures [G]. It differs from [Procedural Voxel World Generation](3891-procedural-voxel-world-generation.md) by focusing on the geometric construction of individual entities rather than global terrain noise [G].

## Operating characteristics
**Time and Memory Complexity:**
*   Time: $O(1)$ per voxel, as it involves a fixed number of hash and noise evaluations [S1, S3, unofficial].
*   Memory: $O(1)$ auxiliary memory per evaluation, as no global world state is stored in the function itself [S1, unofficial].

**Measured Performance:**
*   Quality 0 is significantly faster due to the absence of FBM noise and tree/leaf calculations, relying only on simple hashes [S1, unofficial].
*   Quality 2 is the most expensive due to FBM evaluation, cave noise, and the leaf search loop [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Deterministic, infinite world generation with minimal storage requirements [S1, S3, unofficial]. Supports multiple complexity levels [S1, unofficial].
*   **Limitations:** Stochastic low-quality mode can result in sparse, disconnected terrain [S1, unofficial]. Leaf placement logic is computationally heavy and requires checking neighboring columns [S1, unofficial].

**When to Use:**
*   Use for infinite exploration games where memory is constrained [S3, unofficial].
*   Use Quality 0 for mobile or low-end hardware [S1, unofficial].

## Background
Originating from GLSL shader implementations for voxel engines, this method was popularized by the "Voxel-World-Shader" project [S1, unofficial]. It is also implemented in standalone voxel engines like `voxel-engine-stackgl` which pass a generation function to the game loop [S3, unofficial]. Academic works like "DigBuild" describe similar phased approaches (Constraints, Biomes, Terrain) for Vulkan-based voxel games [S4, unofficial].

## Sources
[S1] GitHub - razzle-reinizu/Voxel-World-Shader: A GLSL voxel-world shader ..., https://github.com/razzle-reinizu/Voxel-World-Shader, fetched 2026-10-25 (unofficial)
[S2] StanislavPetrovV/Minecraft, https://github.com/StanislavPetrovV/Minecraft, fetched 2026-10-26 (unofficial)
[S3] voxel/voxel-engine-stackgl, https://github.com/voxel/voxel-engine-stackgl, fetched 2026-10-27 (unofficial)
[S4] DigBuild: Sandbox Voxel Game and Engine Using Vulkan, https://gredos.usal.es/handle/10366/149914, fetched 2026-10-31 (unofficial)
