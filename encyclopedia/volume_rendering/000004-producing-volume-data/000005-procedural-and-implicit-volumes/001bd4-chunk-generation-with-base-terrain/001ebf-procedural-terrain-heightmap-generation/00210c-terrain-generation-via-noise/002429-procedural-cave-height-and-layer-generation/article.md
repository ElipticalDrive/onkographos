# Procedural Cave Height and Layer Generation

Also called: Cave Height and Layer Generation, Cave layer generation, FunctionMakeCaveLayer, Procedural Cave Layer Generation

## Overview
Modifies voxel density to create a horizontal cave layer with variable height, stalactites, and Perlin noise displacement [S1, unofficial]. It builds on [Terrain Generation via Noise](2412-terrain-generation-via-noise.md) by applying a Z-axis dependent noise profile to carve a specific elevation band [G].

## How it works
**Inputs:**
- `Density`: Initial voxel density (float) [S1, unofficial].
- `WorldPos`: 3D world position $(X, Y, Z)$ of the voxel center [S1, unofficial].

**Outputs:**
- `Result`: Modified density value (float, clamped to $[0, 1]$) [S1, unofficial].

**Parameters:**
- `BaseCaveLevel`: 7000.0f. The central Z-coordinate offset for the cave layer [S1, unofficial].
- `BaseCaveHeight`: 400.0f. Fixed vertical thickness of the cave layer [S1, unofficial].
- `ExtraCaveHeightMultiplier`: 490.0f. Amplitude for height variation noise [S1, unofficial].
- `ExtraCaveLevelMultiplier`: 1000.0f. Amplitude for Z-level displacement noise [S1, unofficial].
- `ScaleStalactite`: 0.005f. Spatial frequency for stalactite noise [S1, unofficial].
- `ScaleGeneral`: 0.001f. Spatial frequency for general cave noise [S1, unofficial].

**Data Structures:**
- No persistent data structures. The method is stateless and evaluates per-voxel [S1, unofficial].

**Steps:**

1.  **Initialize Result:**
    $$ \text{Result} = \text{Density} $$ [S1, unofficial]

2.  **Calculate Base Cave Height Variation:**
    Compute a 3D Perlin noise value using scaled X and Y coordinates, ignoring Z for the base height variation:
    $$ v_3 = (X \cdot 0.001, Y \cdot 0.001, 0) $$
    $$ \text{Noise3} = \text{PerlinNoise}(v_3) $$
    $$ \text{ExtraCaveHeight} = 490 \cdot \text{Noise3} $$
    $$ \text{CaveHeight} = \max(0, 400 + \text{ExtraCaveHeight}) $$ [S1, unofficial]

3.  **Calculate Cave Level Displacement:**
    Compute a 3D Perlin noise value using scaled X and Y coordinates with a fixed Z offset:
    $$ v_4 = (X \cdot 0.00025, Y \cdot 0.00025, 10) $$
    $$ \text{Noise4} = \text{PerlinNoise}(v_4) $$
    $$ \text{ExtraCaveLevel} = 1000 \cdot \text{Noise4} $$
    $$ \text{CaveLevel} = 7000 + \text{ExtraCaveLevel} $$ [S1, unofficial]

4.  **Calculate Cave Layer Falloff:**
    Compute a Gaussian-like falloff based on the distance from the voxel's Z position to the calculated `CaveLevel`, normalized by `CaveHeight`:
    $$ \text{CaveHeightF} = \text{CaveHeight} \cdot 160 $$
    $$ \text{CaveLayer} = 1 - \exp\left( -\frac{(Z + \text{CaveLevel})^2}{\text{CaveHeightF}} \right) $$ [S1, unofficial]

5.  **Calculate Stalactite Noise:**
    $$ v_0 = (X \cdot 0.005, Y \cdot 0.005, Z \cdot 0.0002) $$
    $$ \text{Noise0} = \text{PerlinNoise}(v_0) $$ [S1, unofficial]

6.  **Calculate General Noise:**
    $$ v_1 = (X \cdot 0.001, Y \cdot 0.001, Z \cdot 0.001) $$
    $$ \text{Noise1} = \text{PerlinNoise}(v_1) $$ [S1, unofficial]

7.  **Blend Density in Cave Zone:**
    If the voxel's Z coordinate is within the range $[-(7000 - 1200), -(7000 + 1200)]$, i.e., $[-8200, -5800]$, blend the result:
    $$ \text{CaveLayerN} = 1 - \text{CaveLayer} $$
    $$ \text{Result} = \text{Result} \cdot (\text{CaveLayer} + \text{CaveLayerN} \cdot \text{Noise0} \cdot 0.5 + \text{Noise1} \cdot 0.75) $$ [S1, unofficial]

8.  **Clamp Result:**
    $$ \text{Result} = \max(0, \min(1, \text{Result})) $$ [S1, unofficial]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def perlin_noise(x: float, y: float, z: float) -> float:
    """Evaluates standard 3D Perlin noise at the given coordinates [G]."""
    # Implementation depends on the chosen Perlin noise variant.
    # Returns a value typically in [-1, 1] or [0, 1].
    pass

def make_cave_layer(density: float, world_pos: tuple) -> float:
    x, y, z = world_pos
    
    # Parameters
    base_cave_level = 7000.0
    base_cave_height = 400.0
    extra_height_mult = 490.0
    extra_level_mult = 1000.0
    
    # Step 2: Base Cave Height Variation
    v3 = (x * 0.001, y * 0.001, 0.0)
    noise3 = perlin_noise(*v3)
    extra_cave_height = extra_height_mult * noise3
    cave_height = max(0.0, base_cave_height + extra_cave_height)
    
    # Step 3: Cave Level Displacement
    v4 = (x * 0.00025, y * 0.00025, 10.0)
    noise4 = perlin_noise(*v4)
    extra_cave_level = extra_level_mult * noise4
    cave_level = base_cave_level + extra_cave_level
    
    # Step 4: Cave Layer Falloff
    cave_height_f = cave_height * 160.0
    # Guard against division by zero if cave_height is exactly 0 [G]
    if cave_height_f < 1e-6:
        cave_height_f = 1e-6
    dist_sq = (z + cave_level) ** 2
    cave_layer = 1.0 - math.exp(-dist_sq / cave_height_f)
    
    # Step 5: Stalactite Noise
    v0 = (x * 0.005, y * 0.005, z * 0.0002)
    noise0 = perlin_noise(*v0)
    
    # Step 6: General Noise
    v1 = (x * 0.001, y * 0.001, z * 0.001)
    noise1 = perlin_noise(*v1)
    
    # Step 7: Blend Density in Cave Zone
    result = density
    # Range is [-8200, -5800]
    if -8200.0 <= z <= -5800.0:
        cave_layer_n = 1.0 - cave_layer
        result = result * (cave_layer + cave_layer_n * noise0 * 0.5 + noise1 * 0.75)
        
    # Step 8: Clamp
    result = max(0.0, min(1.0, result))
    
    return result
```

**Edge Cases and Pitfalls:**
- **Precision:** The Z-coordinates involved (around -7000 to -8200) are large. Ensure floating-point precision is sufficient for the noise sampling at these scales [G].
- **Degenerate Input:** If `CaveHeight` is 0, `CaveHeightF` becomes 0, leading to division by zero in the exponential term. The `max(0, ...)` in Step 2 ensures `CaveHeight` is non-negative, but if it is exactly 0, the division by zero must be handled (e.g., by capping `CaveHeightF` to a small epsilon or skipping the blend) [G].

## Subtopics

### [Terrain Chunk Generation](3653-terrain-chunk-generation.md)
Generates a voxel chunk using two layers of fractal Brownian motion (fBm) noise: one for surface height and one for cave depth. It assigns materials based on vertical distance from the surface and absolute Y position. This method builds on [Procedural Cave Height and Layer Generation](3202-procedural-cave-height-and-layer-generation.md) for procedural terrain implementation [G].

### Procedural Volumetric Terrain Generation
A real-time, customizable method to generate volumetric terrains with layered materials, mineral veins, underground caves, and material mixtures [S2]. It uses probabilistic generation to ensure unpredictability and allows designers to specify terrain characteristics via intuitive parameters [S2]. The process divides into three steps: read input data and index materials, generate layers/veins/caves, and construct the final volumetric data using a stacked column representation [S2]. Sharp limits between materials are avoided by generating underground layers formed with material mixtures, connecting materials with realistic progressive changes [S2].

## Operating characteristics
- **Time Complexity:** O(1) per voxel. The method involves a constant number of noise evaluations (3) and basic arithmetic operations [G].
- **Memory Complexity:** O(1). No additional memory is required beyond local variables [G].
- **Performance:** The method is computationally lightweight, suitable for real-time voxel generation. The primary cost is the Perlin noise evaluation [S1, unofficial].
- **Limitations:** The cave layer is strictly horizontal and centered around a fixed Z-coordinate band, which may not suit all terrain types [S1, unofficial].
- **When to use:** When a distinct, horizontal cave layer is desired in a voxel terrain [S1, unofficial].
- **When not to use:** When the terrain uses a different coordinate convention or requires complex, non-horizontal cave structures [S1, unofficial].

## Background
Originating from the UE4VoxelTerrain project, this method is part of a procedural terrain generation pipeline [S1, unofficial]. It is designed to create underground cave systems by modifying the density field based on noise-driven height and level variations [S1, unofficial].

## Sources
[S1] bw2012/UE4VoxelTerrain, https://github.com/bw2012/UE4VoxelTerrain, fetched 2026-10-26 (unofficial)
[S2] Procedural approach to volumetric terrain generation, https://zenodo.org/records/18429902, fetched 2026-10-27
