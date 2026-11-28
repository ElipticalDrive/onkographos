# Height Curve Activation & Voxel Column Generation

Also called: Heightmap to Voxel Mesh Conversion

## Overview
This method transforms a normalized 2D noise heightmap into a 3D voxel column structure by applying a hyperbolic tangent activation function, scaling the result to world units, and stacking voxels from a defined ground offset to the calculated height [S1, unofficial]. It differs from linear extrusion by compressing extreme noise values through a non-linear curve and explicitly mapping discrete terrain bands to color indices [S1, unofficial].

## How it works
**Inputs:**
- `noise_map`: A 2D grid of normalized noise values [S1, unofficial].
- `color_map`: A structure containing a list of `Terrain` bands, each with a `start_level`, `end_level`, and `color_index` [S1, unofficial].
- `zscale`: A floating-point scaling factor converting normalized height to voxel depth [S1, unofficial].
- `ground`: A `u8` base height offset for the terrain floor [S1, unofficial].

**Outputs:**
- A 3D voxel mesh populated with stacked voxels representing the terrain [S1, unofficial].

**Data Structures:**
The processing state is held in a `MeshMap` struct [S1, unofficial]:
- `zscale`: `f64`, height scaling factor.
- `ground`: `u8`, base height offset.
- `map`: `Vec<Vec<u8>>`, 2D grid storing computed voxel heights.
- `color_map`: `Option<&'a ColorMap>`, reference to terrain color definitions.
- `noise_map`: `Option<&'a NoiseMap>`, reference to input noise data.

Each `Terrain` band defines a height range and palette index [S1, unofficial]:
- `start_level`: `u8`, inclusive lower bound.
- `end_level`: `u8`, inclusive upper bound.
- `color_index`: `u8`, index into the voxel palette.

**Steps:**
1. **Compute Heights**: Iterate over every coordinate $(x, y)$ in the `noise_map` [S1, unofficial]. Retrieve the raw noise value $n$, apply the activation function, scale it, and store the resulting voxel height $z$ in `map[x][y]` [S1, unofficial].
2. **Generate Voxels**: Iterate over the computed `map` grid [S1, unofficial]. For each $(x, y)$, retrieve $z$, calculate the relative height $h_{rel} = z - ground$, and look up the corresponding color index [S1, unofficial]. Create a voxel at $(x, y, z)$ with that color and add it to the model [S1, unofficial].
3. **Fill Columns**: If $z > ground$, iterate downwards from $z-1$ to $ground$ inclusive [S1, unofficial]. For each level $zp$, calculate $h_{rel\_col} = zp - ground$, look up its color index, create a voxel at $(x, y, zp)$, and add it to the model [S1, unofficial].

**Formulas:**
- Activation: $h_{norm} = \tanh(n \times 1.5)$ [S1, unofficial].
- Height Index: $z = \lfloor h_{norm} \times zscale \rfloor + ground$ [S1, unofficial].
- The source code casts the scaled result to `u8`, which truncates fractional parts [S1, unofficial].

**Parameters:**
- `zscale`: Not given in the sources. Typical values depend on desired terrain resolution [S1, unofficial].
- `ground`: Not given in the sources. Typically $0$ or a small positive integer [S1, unofficial].
- Activation Factor: $1.5$ [S1, unofficial].
- Water Line: Defined as $6.0 / 50.0$ in the notes, but the provided code implements water as a terrain band spanning levels $0$ to $6$ rather than a hard clamp [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Color Mapping Failure**: If $z - ground$ falls outside all defined `Terrain` bands, the color lookup returns `None` and generation fails with "Unable to map height" [S1, unofficial].
- **Voxel Bounds**: If $z$ exceeds the target model's depth, `add_voxel` fails with "Unable to add voxel, does it fit inside the model?" [S1, unofficial].
- **Precision**: Casting to `u8` truncates floating-point values; ensure `zscale` and `ground` prevent overflow beyond $255$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def generate_terrain(noise_map, color_map, zscale, ground, model):
    """
    Generates a voxel terrain from a noise map using tanh activation.
    
    Args:
        noise_map: 2D array of floats (normalized noise).
        color_map: Object with method get_terrain_color(relative_height) -> int.
        zscale: float, scaling factor for height.
        ground: int, base height.
        model: Object with method add_voxel(x, y, z, color_index) -> bool.
        
    Returns:
        True if successful, raises ValueError on failure.
    """
    height = len(noise_map)
    width = len(noise_map[0])
    
    # Phase 1: Compute heights from noise
    map_heights = [[0] * width for _ in range(height)]
    
    for y in range(height):
        for x in range(width):
            n = noise_map[y][x]
            activated = math.tanh(n * 1.5)
            z = int(activated * zscale) + ground
            map_heights[y][x] = z

    # Phase 2: Generate voxels and fill columns
    for y in range(height):
        for x in range(width):
            z = map_heights[y][x]
            
            # Get color for top voxel
            rel_height = z - ground
            color_index = color_map.get_terrain_color(rel_height)
            
            if color_index is None:
                raise ValueError(f"Unable to map height: {rel_height}")
            
            if not model.add_voxel(x, y, z, color_index):
                raise ValueError("Unable to add voxel, does it fit inside the model?")
            
            # Fill column downwards from z-1 to ground inclusive
            if z > ground:
                for zp in range(z - 1, ground - 1, -1):
                    rel_height_col = zp - ground
                    color_index_col = color_map.get_terrain_color(rel_height_col)
                    
                    if color_index_col is None:
                        raise ValueError(f"Unable to map height: {rel_height_col}")
                    
                    if not model.add_voxel(x, y, zp, color_index_col):
                        raise ValueError("Unable to add voxel, does it fit inside the model?")

    return True
```

## Subtopics

### Height Field Surface Approximation
This variant treats a height field as a fluid surface boundary rather than a solid extrusion [S2, unofficial]. Cells intersecting the height field are labeled as Surface cells, cells positioned above the field are labeled Empty, and cells below are labeled Full [S2, unofficial]. It is chosen when simulating calm fluid flows or approximating water surfaces without modeling dramatic waves or violent splashes [S2, unofficial].

## Operating characteristics
- **Time Complexity**: $O(W \times H \times D)$, where $W$ and $H$ are map dimensions and $D$ is average column depth [G].
- **Memory Complexity**: $O(W \times H)$ for the intermediate height grid, plus storage for the voxel model [G].
- **Strengths**: Computationally simple; the tanh activation naturally smooths terrain transitions without requiring separate filtering passes [S1, unofficial].
- **Limitations**: Discrete color bands can cause visual layering; the method cannot generate overhangs, caves, or non-heightmap geometry [S1, unofficial].
- **When to use**: Rapid generation of 2D-height-based terrain for voxel games or visualizations where performance is prioritized [S1, unofficial].
- **When not to use**: Scenes requiring complex 3D topology, fluid dynamics, or non-extruded geometry [S1, unofficial].

## Background
The method originates from the `tellus` project by dsunshi, which draws inspiration from procedural terrain generation techniques popularized by Sebastian Lague and tools like MagicaVoxel and WorldPainter [S1, unofficial]. It applies a hyperbolic tangent activation curve to raw noise, a less common approach in basic voxel generators but effective for compressing noise extremes [S1, unofficial]. The project is distributed under the Apache 2.0 or MIT license [S1, unofficial].

## Sources
[S1] dsunshi/tellus, https://github.com/dsunshi/tellus, fetched 2026-10-27 (unofficial)
[S2] Efficient computation and rendering of smoke animation, http://hdl.handle.net/2346/16985, fetched 2026-10-26 (unofficial)
