# Procedural Voxel World Generation

Also called: Voxel World Shader, Voxel Type Assignment Logic, Voxel ID Assignment with Caves and Biomes

## Overview
Procedural Voxel World Generation determines the material type of every voxel in a 3D coordinate space by evaluating a deterministic function per cell, synthesizing terrain height, caves, trees, and special blocks (clouds, emissives) from noise and hash functions [S1, S3, unofficial]. It differs from heightmap-based methods by generating volumetric data directly in 3D space rather than projecting a 2D surface, and from octree methods by using a fixed-resolution chunk structure where every cell is evaluated [S1, S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** 3D integer coordinates $(x, y, z)$ representing the voxel cell in world space. Configuration parameters: `QUALITY` (integer, typically 0–3), `CLOUD_HEIGHT` (integer), `ENABLE_TREES`, `ENABLE_CAVES`, `ENABLE_CLOUDS`, `BLOCK_LIGHT` (boolean), `ENDLESS_LOWER_HEIGHT` (boolean).
*   **Output:** An integer block ID. Standard IDs: 1=Grass, 2=Dirt, 3=Stone, 4=Wood, 5=Leaves, 6=Emissive, 7=Cloud. 0 represents Air [S1, unofficial].

**Data Structures:**
*   **Block ID:** Single integer per voxel.
*   **Noise/Hash Functions:**
    *   `hash12(vec2 p)`: Returns `fract(sin(dot(p, vec2(127.1, 311.7))) * 43758.5453123)` [S1, unofficial].
    *   `hash13(vec3 p)`: Returns `fract(sin(dot(p, vec3(127.1, 311.7, 74.7))) * 43758.5453123)` [S1, unofficial].
    *   `fbm3(vec3 p)`: Fractional Brownian Motion 3D noise. Implementation details vary, but typically sums octave noise values [S1, unofficial].
    *   `noise3`: Value noise on a unit lattice with Hermite smoothing and trilinear interpolation [S1, unofficial].

**Algorithm Steps:**

1.  **Global Bounds Check:**
    If `ENDLESS_LOWER_HEIGHT` is false and $y \le -50$, return 3 (Stone) [S1, unofficial].

2.  **Quality 0 (Stochastic/Low Quality):**
    *   Compute random value $r = \text{hash13}(x \cdot 0.12, y \cdot 0.08, z \cdot 0.15)$ [S1, unofficial].
    *   Compute density: $d = 0.30 + 0.360 \cdot r + 0.15$ [S1, unofficial].
    *   Determine threshold $T$:
        *   If $y \ge 15$, $T = 0.8069$.
        *   If $y < 15$, $T = 0.6$ [S1, unofficial].
    *   If $d > T$:
        *   If $r > 0.999$, return 6 (Emissive) [S1, unofficial].
        *   Compute pick value $p = \text{hash12}(x \cdot 0.19, z \cdot 0.23)$ [S1, unofficial].
        *   If $p < 0.33$, return 1 (Grass).
        *   Else if $p < 0.4$, return 4 (Wood).
        *   Else if $p < 0.66$, return 2 (Dirt).
        *   Else return 3 (Stone) [S1, unofficial].
    *   If $y \ge 15$, return 0 (Air) [S1, unofficial].
    *   Return 0 (Air) [S1, unofficial].

3.  **Quality > 0 (High Quality):**
    *   **Clouds:** If `ENABLE_CLOUDS` is true and $y == \text{CLOUD_HEIGHT}$:
        *   Compute $ch = \text{hash12}(\lfloor(x \cdot 0.21, z \cdot 0.13) \cdot 2.0\rfloor)$ [S1, unofficial].
        *   If $ch > 0.74$, return 7 (Cloud).
        *   Else return 0 (Air) [S1, unofficial].
    *   **Sky Cutoff:** If $y \ge 19$, return 0 (Air) [S1, unofficial].
    *   **Terrain Height:**
        *   Compute float height $h_f = \text{proceduralTerrainHeight}(x, z, \text{QUALITY})$ [S1, unofficial].
        *   Formula for `proceduralTerrainHeight`:
            *   Scale $s$: $0.045$ if $q=1$, $0.055$ if $q=2$, $0.06$ otherwise [S1, unofficial].
            *   Vertical Scale $V$: $10.0$ if $q=1$, $11.0$ if $q=2$, $12.0$ otherwise [S1, unofficial].
            *   Base $B$: $3.5$ if $q=1$, $4.0$ otherwise [S1, unofficial].
            *   $h_f = \text{fbm3}(x \cdot s, 0.0, z \cdot s) \cdot V + B$ [S1, unofficial].
        *   $h = \lfloor h_f \rfloor$ [S1, unofficial].
    *   **Caves:**
        *   If `ENABLE_CAVES` is true:
            *   Compute cave noise $c_n = 2.0 \cdot \text{fbm3}(x \cdot 0.10, y \cdot 0.10, z \cdot 0.10)$ [S1, unofficial].
            *   Threshold $C_{th}$: $0.68$ if $q=2$, $0.66$ otherwise [S1, unofficial].
            *   If $c_n > C_{th}$, mark `isCave = true` [S1, unofficial].
            *   Else `isCave = false` [S1, unofficial].
        *   Else `isCave = false` [S1, unofficial].
    *   **Emissive Blocks (Lighting):**
        *   If `BLOCK_LIGHT` is true:
            *   Compute $rr = \text{hash13}(x \cdot 0.23, y \cdot 0.17, z \cdot 0.19)$ [S1, unofficial].
            *   If `isCave`:
                *   Threshold $E_{cave}$: $0.9975$ if $q=1$, $0.9965$ if $q=2$, $0.995$ otherwise [S1, unofficial].
                *   If $rr > E_{cave}$, return 6 (Emissive) [S1, unofficial].
            *   If $y == h$:
                *   Threshold $E_{surf}$: $0.9995$ if $q=1$, $0.999$ if $q=2$, $0.998$ otherwise [S1, unofficial].
                *   If $rr > E_{surf}$, return 6 (Emissive) [S1, unofficial].
    *   **Terrain Body:**
        *   If $y \le h$ AND `isCave` is false:
            *   If $y == h$, return 1 (Grass) [S1, unofficial].
            *   Else if $y \ge h - 3$, return 2 (Dirt) [S1, unofficial].
            *   Else return 3 (Stone) [S1, unofficial].
    *   **Trees:**
        *   If `ENABLE_TREES` is true:
            *   Check `hasTreeAtColumn(x, z, QUALITY)` [S1, unofficial].
            *   Formula for `hasTreeAtColumn`:
                *   $r_t = \text{hash12}(x \cdot 0.374, z \cdot 0.912)$ [S1, unofficial].
                *   Return true if $r_t > 0.995$ ($q=1$), $0.985$ ($q=2$), or $0.97$ (other) [S1, unofficial].
            *   If tree present:
                *   Trunk height $t_h = 3 + \lfloor \text{hash12}(x \cdot 0.11, z \cdot 0.08) \cdot 3.0 \rfloor$ [S1, unofficial].
                *   Base $y_b = h + 1$ [S1, unofficial].
                *   Top $y_t = y_b + t_h$ [S1, unofficial].
                *   If $y_b \le y < y_t$, return 4 (Wood) [S1, unofficial].
    *   **Leaves:**
        *   Define search bounds:
            *   `SEARCH_MAX`: $1$ if $q=1$, $3$ if $q=2$ [S1, unofficial].
            *   `maxOffset`: $1$ if $q=1$, $2$ if $q=2$ [S1, unofficial].
        *   Iterate $ox$ from $-\text{SEARCH\_MAX}$ to $\text{SEARCH\_MAX}$, $oz$ from $-\text{SEARCH\_MAX}$ to $\text{SEARCH\_MAX}$ [S1, unofficial].
        *   Skip if $|ox| > \text{maxOffset}$ or $|oz| > \text{maxOffset}$ [S1, unofficial].
        *   Check if neighbor column $(x+ox, z+oz)$ has a tree using `hasTreeAtColumn` [S1, unofficial].
        *   If neighbor has tree:
            *   Calculate local coordinates relative to neighbor tree top: $dx = x - (x+ox)$, $dz = z - (z+oz)$, $dy = y - y_t$ [G] (Note: $y_t$ is calculated per neighbor).
            *   Return 5 (Leaves) if:
                *   $dy = 0$ or $1$ AND $dx \in [-1, 2]$ AND $dz \in [-1, 2]$ OR ($dx = -2$ AND $dz \in [-1, 2]$) [S1, unofficial].
                *   OR $dy = 2$ AND ($dx=0, dz=0$) OR ($|dx| \le 1, dz=0$) OR ($|dz| \le 1, dx=0$) [S1, unofficial].
    *   Return 0 (Air) [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hash12(p):
    return fract(sin(dot(p, vec2(127.1, 311.7))) * 43758.5453123)

def hash13(p):
    return fract(sin(dot(p, vec3(127.1, 311.7, 74.7))) * 43758.5453123)

def proceduralTerrainHeight(x, z, q):
    if q == 1:
        s = 0.045
        v = 10.0
        b = 3.5
    elif q == 2:
        s = 0.055
        v = 11.0
        b = 4.0
    else:
        s = 0.06
        v = 12.0
        b = 4.0
    
    # fbm3 is assumed to be defined in the environment
    h = fbm3(vec3(x * s, 0.0, z * s))
    return h * v + b

def hasTreeAtColumn(x, z, q):
    r = hash12(vec2(float(x) * 0.374, float(z) * 0.912))
    if q == 1:
        return r > 0.995
    elif q == 2:
        return r > 0.985
    else:
        return r > 0.97

def getBlockType(x, y, z, quality, cloudY, enableTrees, enableCaves, enableClouds, blockLight, endlessLowerHeight):
    # Global Bounds
    if not endlessLowerHeight and y <= -50:
        return 3 # Stone

    # Quality 0: Stochastic
    if quality == 0:
        r = hash13(vec3(float(x) * 0.12, float(y) * 0.08, float(z) * 0.15))
        density = 0.30 + 0.360 * r + 0.15
        
        threshold = 0.8069 if y >= 15 else 0.6
        
        if density > threshold:
            if r > 0.999:
                return 6 # Emissive
            pick = hash12(vec2(float(x) * 0.19, float(z) * 0.23))
            if pick < 0.33:
                return 1 # Grass
            elif pick < 0.4:
                return 4 # Wood
            elif pick < 0.66:
                return 2 # Dirt
            else:
                return 3 # Stone
        
        if y >= 15:
            return 0 # Air
        return 0 # Air

    # Quality > 0
    # Clouds
    if enableClouds:
        if y == cloudY:
            ch = hash12(floor(vec2(float(x) * 0.21, float(z) * 0.13) * 2.0))
            if ch > 0.74:
                return 7 # Cloud
            return 0 # Air
    else:
        if y == cloudY:
            return 0 # Air

    if y >= 19:
        return 0 # Air

    fx = float(x)
    fz = float(z)
    terrainH_f = proceduralTerrainHeight(fx, fz, quality)
    terrainH = floor(terrainH_f)

    # Caves
    isCave = False
    if enableCaves:
        caveNoise = 2.0 * fbm3(vec3(float(x) * 0.10, float(y) * 0.10, float(z) * 0.10))
        caveThresh = 0.68 if quality == 2 else 0.66
        if caveNoise > caveThresh:
            isCave = True

    # Emissive
    if blockLight:
        rr = hash13(vec3(float(x) * 0.23, float(y) * 0.17, float(z) * 0.19))
        
        if isCave:
            caveET = 0.9975 if quality == 1 else (0.9965 if quality == 2 else 0.995)
            if rr > caveET:
                return 6 # Emissive
        
        if y == terrainH:
            surfET = 0.9995 if quality == 1 else (0.999 if quality == 2 else 0.998)
            if rr > surfET:
                return 6 # Emissive

    # Terrain Body
    if y <= terrainH and not isCave:
        if y == terrainH:
            return 1 # Grass
        if y >= terrainH - 3:
            return 2 # Dirt
        return 3 # Stone

    # Trees
    if enableTrees:
        if hasTreeAtColumn(x, z, quality):
            trunkH = 3 + floor(hash12(vec2(float(x) * 0.11, float(z) * 0.08)) * 3.0)
            baseY = terrainH + 1
            topY = baseY + trunkH
            if baseY <= y < topY:
                return 4 # Wood

    # Leaves
    searchMax = 1 if quality == 1 else 3
    maxOffset = 1 if quality == 1 else 2
    
    for ox in range(-searchMax, searchMax + 1):
        for oz in range(-searchMax, searchMax + 1):
            if abs(ox) > maxOffset or abs(oz) > maxOffset:
                continue
            
            # Check neighbor column for tree
            if hasTreeAtColumn(x + ox, z + oz, quality):
                # Calculate neighbor tree top
                n_trunkH = 3 + floor(hash12(vec2(float(x + ox) * 0.11, float(z + oz) * 0.08)) * 3.0)
                n_baseY = terrainH + 1 # Note: terrainH is local to x,z, not neighbor. 
                                       # In shader, terrainH is computed per cell. 
                                       # For neighbor, we need neighbor's terrain height.
                # Correction: The shader code iterates neighbors to find IF they have a tree,
                # but uses the CURRENT cell's y. The leaf logic in the source code 
                # calculates dx, dz, dy relative to the neighbor's tree top.
                # We must compute neighbor's terrain height to get its topY.
                
                n_terrainH_f = proceduralTerrainHeight(float(x + ox), float(z + oz), quality)
                n_terrainH = floor(n_terrainH_f)
                n_topY = n_terrainH + 1 + n_trunkH
                
                dx = x - (x + ox)
                dz = z - (z + oz)
                dy = y - n_topY
                
                # Leaf placement rules
                if (dy == 0 or dy == 1) and (-1 <= dx <= 2) and (-1 <= dz <= 2):
                    return 5 # Leaves
                if (dx == -2) and (-1 <= dz <= 2):
                    return 5 # Leaves
                
                if dy == 2:
                    if (dx == 0 and dz == 0):
                        return 5 # Leaves
                    if (abs(dx) <= 1 and dz == 0):
                        return 5 # Leaves
                    if (abs(dz) <= 1 and dx == 0):
                        return 5 # Leaves

    return 0 # Air
```

**Parameters:**
*   `QUALITY`: Integer. 0: Stochastic sparse blocks, no trees/caves/clouds. 1: Simple terrain, basic leaves (3x3), caves active. 2: Detailed terrain, detailed leaves (4x4), caves active. 3: Dense trees (implied by source comments, though logic matches q=2 thresholds) [S1, unofficial].
*   `CLOUD_HEIGHT`: Integer Y-coordinate for cloud layer [S1, unofficial].
*   `ENABLE_TREES`, `ENABLE_CAVES`, `ENABLE_CLOUDS`, `BLOCK_LIGHT`: Boolean flags [S1, unofficial].
*   `ENDLESS_LOWER_HEIGHT`: Boolean. If false, returns Stone below Y=-50 [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Division by Zero:** In touch input handling for UV calculation, resolution is clamped to `max(1.0)` to prevent division by zero [S1, unofficial].
*   **Precision:** Hash functions rely on floating-point precision. `fract(sin(dot(...)))` is fast but may have statistical shortcomings compared to integer bitwise hashes [S1, unofficial].
*   **Boundary Conditions:** The leaf search loop assumes neighbor trees exist within the search radius. If a tree is at the chunk boundary, the neighbor check might fail if the neighbor's chunk hasn't been generated or if the hash logic doesn't account for chunk offsets correctly (though the shader uses global coordinates, so this is less of an issue for pure generation) [S1, unofficial].

## Subtopics

### [Deterministic Fixture Worlds](3941-deterministic-fixture-worlds.md)
Generates specific, small-scale geometric structures (slabs, towers) using pure functions of `VolumeId` without RNG, serving as regression tests.

### [Biome and Voxel Layer Determination](4167-biome-and-voxel-layer-determination.md)
Assigns voxel types based on vertical position relative to terrain surface and biome parameters (moisture, temperature) derived from noise.

### [Procedural Tree Generation](4826-procedural-tree-generation.md)
Places voxel-based trees on terrain surfaces by identifying valid ground positions and filling volumes with trunk and leaf blocks.

### [Procedural Chunk Generation with Density Falloff](15703-procedural-chunk-generation-with-density-falloff.md)
Generates chunks with radial center falloff and plateau mechanisms to create isolated floating structures.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per voxel evaluation. The cost is dominated by noise function evaluations (FBM) and hash computations. Tree leaf placement adds a constant factor loop ($O(1)$ since search radius is fixed per quality) [S1, unofficial].
*   **Memory Complexity:** $O(1)$ auxiliary memory per voxel evaluation. No large data structures are required during generation if the function is stateless [S1, S3, unofficial].
*   **Performance:** Quality 0 is extremely fast due to simple hash checks. Quality 2 is more expensive due to FBM noise and leaf search loops [S1, unofficial].
*   **Strengths:** Deterministic, reproducible, low memory footprint, supports complex features (caves, trees) without pre-computed meshes [S1, S3, unofficial].
*   **Limitations:** Fixed resolution per chunk. High-quality generation can be CPU/GPU intensive if evaluated per-voxel without culling [S1, unofficial].
*   **When to use:** When deterministic, infinite-world generation is required with minimal storage overhead.
*   **When not to use:** When highly optimized meshing is required without per-voxel evaluation overhead, or when non-deterministic, organic variation beyond noise is needed.

## Background
Procedural voxel world generation is a core technique in voxel-based games and simulations. It evolved from early heightmap-based terrain generation to full volumetric evaluation. The specific implementation described here, often referred to as "Voxel World Shader," originates from GLSL-based voxel engines where generation is performed on the GPU or via a JavaScript bridge to WebGL [S1, S3, unofficial]. The three-phase approach (constraints, biomes, terrain) was formalized in the DigBuild engine documentation [S4, unofficial]. The method relies heavily on hash-based pseudo-random number generation for efficiency in shader environments [S1, unofficial].

## Sources
[S1] GitHub - razzle-reinizu/Voxel-World-Shader: A GLSL voxel-world shader ..., https://github.com/razzle-reinizu/Voxel-World-Shader, fetched 2026-10-25 (unofficial)
[S2] StanislavPetrovV/Minecraft, https://github.com/StanislavPetrovV/Minecraft, fetched 2026-10-26 (unofficial)
[S3] voxel/voxel-engine-stackgl, https://github.com/voxel/voxel-engine-stackgl, fetched 2026-10-27 (unofficial)
[S4] DigBuild: Sandbox Voxel Game and Engine Using Vulkan, https://gredos.usal.es/handle/10366/149914, fetched 2026-10-31 (unofficial)
