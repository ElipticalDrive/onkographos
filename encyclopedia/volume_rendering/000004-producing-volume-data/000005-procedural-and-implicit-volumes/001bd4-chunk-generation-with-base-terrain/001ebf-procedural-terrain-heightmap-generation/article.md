# Procedural Terrain Heightmap Generation

Also called: FBM Heightmap Chunk Generation, Procedural World Generation

## Overview
Procedural Terrain Heightmap Generation creates a 3D voxel terrain chunk by sampling a 2D noise function to determine column heights, then filling solid blocks from the base up to the computed height for each $(x,z)$ position within the chunk [S2, S4, 5, unofficial]. It solves the problem of creating deterministic, pseudo-random terrain geometry by leveraging the property that a fixed input yields the same output, allowing terrain data to be preserved across frames without change . It sets itself apart from [Procedural Terrain Feature Generation](2572-procedural-terrain-feature-generation.md) by focusing on continuous height variation rather than discrete feature placement, and from [Heightmap generation via circular displacements](5186-heightmap-generation-via-circular-displacements.md) by using gradient-based noise functions rather than radial displacement patterns [G].

## How it works
**Inputs and Outputs:**
*   **Input:** Chunk coordinates $(cx, cz)$, chunk dimensions $(CX, CZ)$, noise generator state (seed, frequency, type), and optional parameters for height scaling and layer definitions.
*   **Output:** A 3D voxel array `block_data` of size $[CX][CY][CZ]$ (or similar layout) where each cell contains an integer material ID (e.g., 0 for air, 1 for grass, etc.).

**Data Structures:**
*   `block_data`: A 3D array of integers. In the standard implementation, the layout is $[16][64][16]$ (x, y, z) [S2, unofficial]. Index 0 represents air, index 1 represents solid terrain block, and higher indices represent specific materials (dirt, stone, water) [S2, unofficial].
*   `noise_state`: Configuration for the noise function, including `seed`, `frequency`, and `noise_type` (e.g., OpenSimplex2) [S2, unofficial].

**Numbered Steps:**
1.  **Initialize Data:** Initialize all `block_data[x][y][z]` to 0 (air) [S2, unofficial].
2.  **Configure Noise:** Create the noise state with a fixed seed and set parameters such as `frequency` (e.g., 0.025) and `noise_type` (e.g., OPENSIMPLEX2) [S2, unofficial].
3.  **Iterate Chunk Grid:** Loop through each column $(x, z)$ within the chunk boundaries ($0 \le x < CX$, $0 \le z < CZ$) [S2, unofficial].
4.  **Calculate World Coordinates:** Compute the world coordinates for the current column:
    $$ wx = cx \cdot CX + x $$
    $$ wz = cz \cdot CZ + z $$
    [S1, S2, unofficial].
5.  **Sample Noise for Height:** Sample the 2D noise function at $(wx, wz)$ (optionally scaled by frequency) [S2, unofficial].
    *   *Variant (FBM):* Use a Fractional Brownian Motion (FBM) function. Calculate height as:
        $$ h = \lfloor \text{fbm}(wx / 80, wz / 80) \cdot 60 \rfloor + 60 $$
        [S1, unofficial].
    *   *Variant (Simple Noise):* Sample noise, scale by a factor (e.g., 4), add an offset (e.g., 5), and cast to integer:
        $$ h = \lfloor \text{noise}(wx, wz) \cdot 4 + 5 \rfloor $$
        [S2, unofficial].
6.  **Fill Terrain Layers:** For each $y$ from 0 to $h$:
    *   If $y == h$: Set `block_data[x][y][z]` to Grass (1) [S1, S2, unofficial].
    *   If $y > h - 4$ (and $y < h$): Set `block_data[x][y][z]` to Dirt (2) [S1, unofficial].
    *   Otherwise: Set `block_data[x][y][z]` to Stone (3) [S1, unofficial].
    *   *Note:* The depth of the dirt layer (4 blocks) is a typical parameter [S1, unofficial].
7.  **Fill Water/Background:** For $y$ from $h + 1$ to a maximum height (e.g., 70 or chunk top):
    *   Set `block_data[x][y][z]` to Water (4) or Air, depending on the desired sea level logic [S1, unofficial].
    *   *Note:* If the noise-based height $h$ exceeds the chunk's vertical bounds, clamp or handle overflow appropriately [G].

**Parameters with Typical Values:**
*   `chunk_maxsize_x`, `chunk_maxsize_z`: 16 [S2, unofficial].
*   `chunk_maxsize_y`: 64 [S2, unofficial].
*   `noise.frequency`: 0.025 [S2, unofficial].
*   `noise.noise_type`: FNL_NOISE_OPENSIMPLEX2 [S2, unofficial].
*   FBM Scale: 80 [S1, unofficial].
*   FBM Height Multiplier: 60 [S1, unofficial].
*   FBM Height Offset: 60 [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Casting noise output to integer can cause "stepping" artifacts if the noise scale is too high relative to chunk size [S1, S2, unofficial].
*   **Boundaries:** Ensure world coordinate calculation correctly maps chunk-local indices to global space to maintain continuity between adjacent chunks [S1, S2, unofficial].
*   **Degenerate Input:** If noise returns values that result in negative heights, clamp $h$ to 0 [G].
*   **CORS:** When using image sampling variants, local file access may fail due to CORS restrictions, requiring a fallback to mathematical generation [S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_chunk(cx, cz, chunk_size_x=16, chunk_size_z=16, chunk_size_y=64):
    # Initialize block data with air (0)
    block_data = [[[0 for z in range(chunk_size_z)] for y in range(chunk_size_y)] for x in range(chunk_size_x)]
    
    # Configure noise (simplified representation)
    noise_state = create_noise_state(seed=12345, frequency=0.025, noise_type="OPENSIMPLEX2")
    
    for x in range(chunk_size_x):
        for z in range(chunk_size_z):
            # Calculate world coordinates
            wx = cx * chunk_size_x + x
            wz = cz * chunk_size_z + z
            
            # Sample noise for height
            # Using FBM variant from [S1, unofficial]
            noise_val = fbm(wx / 80.0, wz / 80.0)
            h = floor(noise_val * 60) + 60
            
            # Clamp height to chunk bounds if necessary
            if h >= chunk_size_y:
                h = chunk_size_y - 1
            if h < 0:
                h = 0
                
            # Fill terrain layers
            for y in range(h + 1):
                if y == h:
                    block_data[x][y][z] = 1  # Grass
                elif y > h - 4:
                    block_data[x][y][z] = 2  # Dirt
                else:
                    block_data[x][y][z] = 3  # Stone
            
            # Fill water below height (assuming sea level at 70 or chunk top)
            # Note: In [S1, unofficial], water is filled up to y=70. 
            # If chunk_size_y < 70, adjust logic. Here we assume chunk covers sea level.
            for y in range(h + 1, chunk_size_y):
                if y <= 70: # Assuming 70 is sea level or max water depth
                    block_data[x][y][z] = 4  # Water
                else:
                    block_data[x][y][z] = 0  # Air above water
                    
    return block_data
```

## Subtopics

### [Terrain Generation via Noise](2412-terrain-generation-via-noise.md)
Terrain Generation via Noise generates a 3D voxel volume from a 2D noise heightmap, extruding terrain-like structures into voxel space. It solves the problem of creating deterministic, pseudo-random terrain geometry by leveraging the property that a fixed input yields the same output, allowing terrain data to be preserved across frames without change. It sets itself apart from [Procedural Terrain Feature Generation](2572-procedural-terrain-feature-generation.md) by focusing on continuous height variation rather than discrete feature placement, and from [Heightmap generation via circular displacements](5186-heightmap-generation-via-circular-displacements.md) by using gradient-based noise functions rather than radial displacement patterns [G].

### [Procedural Terrain Feature Generation](2572-procedural-terrain-feature-generation.md)
Procedural Terrain Feature Generation is a method for creating specific volumetric structures, primarily cliffs and overhangs, by approximating their topology as a bicubic Bézier surface combined with a parabolic plateau function. It produces local delta values (additive blending) applied directly to volumetric data, allowing for nondestructive editing where parameters can be adjusted and transformations applied before surface extraction. It differs from general noise-based terrain generation by focusing on explicit geometric primitives for sharp features rather than stochastic displacement.

### [Smooth Terrain Grid](5026-smooth-terrain-grid.md)
Smooth Terrain Grid is a procedural geometry generation technique that converts discrete voxel height data into a continuous, smoothed terrain mesh by computing vertex heights at grid corners via averaging. It solves the problem of jagged, stair-stepped voxel surfaces by replacing flat voxel faces with interpolated planes, allowing for tilted and elevated terrain shapes. It differs from [Terrain Generation via Noise](2412-terrain-generation-via-noise.md) by relying on spatial averaging of existing discrete values rather than continuous noise functions, and from [Procedural Terrain Feature Generation](2572-procedural-terrain-feature-generation.md) by focusing on global surface smoothing rather than local feature placement.

### [Heightmap generation via circular displacements](5186-heightmap-generation-via-circular-displacements.md)
This method generates a procedural terrain heightmap by iteratively applying overlapping circular displacement functions to a flat vertex grid. It constructs height by accumulating discrete, localized perturbations defined by random centers, radii, and signs.

### [Terrain Skinning via Perturbation Function](13510-terrain-skinning-via-perturbation-function.md)
Terrain Skinning via Perturbation Function represents complex terrain as a simple base surface (e.g., a plane) perturbed by a scalar altitude map, eliminating the need for polygonal triangulation. Unlike voxel-based methods that require mesh extraction (e.g., Marching Cubes) or traditional heightmap rendering that suffers from LOD cracking and high depth complexity, this method uses recursive voxel subdivision and direct texture-sampling of the altitude map to modulate the base surface equation, resulting in low depth complexity and seamless rendering.

### [Chunk-based Terrain Generation and Culling](15193-chunk-based-terrain-generation-and-culling.md)
This method generates terrain geometry using a hierarchical octree structure where leaf nodes correspond to terrain chunks. It decouples heightmap generation from mesh creation, using a culling system based on camera distance and Level of Detail (LOD) thresholds to determine which chunks require processing. It differs from flat grid approaches by adapting resolution dynamically based on viewer proximity, ensuring performance scales with scene complexity rather than world size.

### [Chunk Filling with Height Map](15196-chunk-filling-with-height-map.md)
Not given in the sources.

### [Heightmap Combination & Seamless Terrain Generation](15472-heightmap-combination-seamless-terrain-generation.md)
This method generates macroscopic terrain by stitching together multiple pre-existing heightmaps into a single seamless composite, using simplex noise to guide the selection and blending of regions. It differs from pure procedural noise generation by leveraging real-world or high-fidelity source data, optimizing memory bandwidth by packing multiple heightmaps into a single RGB/RGBA texture for simultaneous retrieval.

### [Far-Terrain Grid Generation](15837-far-terrain-grid-generation.md)
Generates a coarse grid of terrain vertices and linear-space RGBA colors for far-terrain rendering, using analytic heightfield sampling.

#### Terrain Generation via Image Sampling
A method to dictate voxel Y-axis height and vertex colors by extracting raw pixel arrays from 2D .png files (grayscale heightmap and RGB colormap) using `getImageData()` [S3, unofficial].
**Steps:**
1. Load `heightmap.png` and `colormap.png` images [S3, unofficial].
2. Extract pixel data using `ctx.getImageData(0, 0, gridSize, gridSize).data` [S3, unofficial].
3. Iterate $x$ from 0 to $gridSize-1$ and $z$ from 0 to $gridSize-1$ [S3, unofficial].
4. Calculate world position: $posX = (x \cdot spacing) - offset$, $posZ = (z \cdot spacing) - offset$, where $offset = (gridSize \cdot spacing) / 2$ [S3, unofficial].
5. Calculate pixel index: $pixelIndex = (z \cdot gridSize + x) \cdot 4$ [S3, unofficial].
6. Calculate Y scale from heightmap: $yScale = 1 + (heightData[pixelIndex] / 255.0) \cdot 100$ [S3, unofficial].
7. Calculate color from colormap: $r = colorData[pixelIndex] / 255.0$, $g = colorData[pixelIndex + 1] / 255.0$, $b = colorData[pixelIndex + 2] / 255.0$ [S3, unofficial].
8. Store instance data: `instancePositions[i3] = posX`, `instancePositions[i3+1] = yScale - 1`, `instancePositions[i3+2] = posZ`; `instanceScales[i1] = yScale`; `instanceColors[i3] = r`, `instanceColors[i3+1] = g`, `instanceColors[i3+2] = b` [S3, unofficial].
**Pitfall:** If local image sampling fails due to CORS restrictions, the engine falls back to a mathematical terrain generation using `Math.sin` and `Math.cos` [S3, unofficial].

#### Cliffs Layer
A terrain layer that creates escarpments by stepping into terraces of a specific height, joined by cliffs over the last portion of each step [S4, unofficial].
**Steps:**
1. Implementation uses integer terracing in 32 bits, identical on CPU (`landform::terrace`) and GPU (`landform_terrace`) [S4, unofficial].
2. The riser share rides in the stack layer's kind bits [S4, unofficial].
**Pitfall:** Boundary: A terrace moves a height by less than a step and is not Lipschitz, so each level's margin grows by two steps to contain it [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(CX \cdot CZ \cdot CY)$ per chunk, where $CX, CZ$ are horizontal chunk dimensions and $CY$ is vertical height [S2, unofficial]. The noise sampling is $O(1)$ per column, making the dominant factor the voxel filling loop [S1, S2, unofficial].
*   **Memory Complexity:** $O(CX \cdot CZ \cdot CY)$ for the voxel array [S2, unofficial].
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Performance scales linearly with chunk size. Larger chunks require more memory and generation time but reduce overhead per unit area [S2, unofficial].
*   **Strengths:** Deterministic output, simple implementation, low memory overhead for simple terrain, easy to integrate with streaming systems [S1, S2, unofficial].
*   **Limitations:** Limited to vertical extrusion (no overhangs/caves without additional logic), noise artifacts at chunk boundaries if not carefully managed, fixed layer structure [S1, S2, unofficial].
*   **When to use:** Simple voxel games, terrain that does not require complex internal structures or overhangs.
*   **When not to use:** When caves, overhangs, or complex geological features are required (use [Procedural Cave Generation](2579-procedural-cave-generation.md) or [Procedural Terrain Feature Generation](2572-procedural-terrain-feature-generation.md)).

## Background
Procedural terrain generation using noise functions is a foundational technique in computer graphics and game development, popularized by games like *Minecraft* [S1, S2, unofficial]. The use of FBM (Fractional Brownian Motion) and simplex noise allows for efficient, infinite terrain generation with natural-looking variations [S1, S2, unofficial]. The specific implementation of layering (grass, dirt, stone) is a common heuristic to add visual variety to otherwise uniform noise-based heights [S1, unofficial].

## Sources
[S1] Build a 3D Voxel Engine in WebGL | Tutorials, https://www.mysimulator.uk/content/tutorials/voxel-engine.html, fetched 2026-10-27 (unofficial)
[S2] Kaefaell/VoxelDemo, https://github.com/Kaefaell/VoxelDemo, fetched 2026-10-27 (unofficial)
[S3] evjapont-hash/CubeWorld-Canyon, https://github.com/evjapont-hash/CubeWorld-Canyon, fetched 2026-10-26 (unofficial)
[S4] Voxel planet: chunk trees, a fully destructible planet, and the lighting to show it, https://github.com/Far-Beyond-Pulsar/Helio/pull/327, fetched 2026-10-28 (unofficial)
