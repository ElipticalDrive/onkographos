# Material Sampling
Also called: faceUV

## Overview
Material Sampling maps block identifiers to specific texture samplers and computes UV coordinates based on the face normal for atlased textures [S1, unofficial]. It differs from its parent [Texture Atlas UV Mapping](4238-texture-atlas-uv-mapping.md) by adding block-type-specific logic, such as grass top/side mixing and leaf tinting, rather than relying solely on geometric UV projection [S1, unofficial].

## How it works
**Inputs:**
- `blockID`: Integer identifier for the block type (e.g., 1 for grass, 5 for leaves) [S1, unofficial].
- `localPos`: `vec3` position of the vertex within the voxel unit cube $[0, 1]^3$ [S1, unofficial].
- `faceNormal`: `vec3` unit normal of the face being shaded [S1, unofficial].
- `cell`: `ivec3` integer coordinates of the voxel cell used for procedural variation [S1, unofficial].
- `tileScale`: Scalar multiplier for UV coordinates, typically derived from atlas dimensions [S1, unofficial].
- `atlasLod`: Fixed to `0.0` to disable mip-map selection [S1, unofficial].

**Outputs:**
- `albedo`: `vec3` color value sampled from the appropriate texture atlas region [S1, unofficial].

**Data Structures:**
- Texture samplers: `samplerGrass`, `samplerGrassSide`, `samplerDirt`, `samplerStone`, `samplerWood`, `samplerWoodSide`, `samplerLeaves` [S1, unofficial].
- Helper function `hash12(vec2 p)`: Returns a pseudo-random float in $[0, 1]$ based on input coordinates [S1, unofficial].

**Steps:**

1. **Compute UV Coordinates:** Determine the 2D UV coordinates based on the dominant axis of the `faceNormal` [S1, unofficial].
   $$
   \text{uv} = \begin{cases} 
   (z, y) \times \text{tileScale} & \text{if } |n_x| > 0.5 \\
   (x, z) \times \text{tileScale} & \text{if } |n_y| > 0.5 \\
   (x, y) \times \text{tileScale} & \text{otherwise}
   \end{cases}
   $$
   Where $\text{localPos} = (x, y, z)$ and $\text{faceNormal} = (n_x, n_y, n_z)$ [S1, unofficial].

2. **Sample Based on Block ID:** Apply block-specific sampling logic [S1, unofficial].
   - **Grass (ID 1):** If $n_y > 0$, sample `samplerGrass` (top) and tint it by mixing with a fixed grass-side green color [S1, unofficial]. If $n_y < 0$, sample `samplerDirt` [S1, unofficial]. Otherwise (side faces), sample `samplerGrassSide` [S1, unofficial].
   - **Leaves (ID 5):** Sample `samplerLeaves` using the computed UV [S1, unofficial]. Apply brightness variation using a hash of the cell coordinates: $v = \text{hash12}(\text{cell}.x \times 0.17, \text{cell}.y \times 0.29)$ [S1, unofficial]. Modulate base color: $\text{base} \times (0.82 + 0.28 \times v)$ [S1, unofficial]. If $n_y < 0$ (bottom face), darken the result by multiplying by $0.70$ [S1, unofficial]. Mix the result with a fixed grass-side green color with a weight of $0.72$ [S1, unofficial].
   - **Wood (ID 4):** If $|n_y| > 0$ (top or bottom), sample `samplerWood` (log top) [S1, unofficial]. Otherwise (sides), sample `samplerWoodSide` (log side) [S1, unofficial].
   - **Other Blocks:** Dirt (ID 2), Stone (ID 3), and others are sampled directly from their respective atlas regions using the UV computed in Step 1 [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Standard graphics operations assumed available: texture(sampler, uv, lod), mix, clamp, length, abs, fract, sin, dot, vec2, vec3

def material_sampling(block_id, local_pos, face_normal, cell, tile_scale, atlas_lod):
    x, y, z = local_pos
    nx, ny, nz = face_normal
    
    # Step 1: Compute UV based on dominant normal axis
    if abs(nx) > 0.5:
        uv = vec2(z, y) * tile_scale
    elif abs(ny) > 0.5:
        uv = vec2(x, z) * tile_scale
    else:
        uv = vec2(x, y) * tile_scale
        
    # Step 2: Sample based on block ID
    if block_id == 1: # Grass
        if ny > 0.0:
            top_color = texture(samplerGrass, uv, atlas_lod).rgb
            grass_green = fetch_grass_side_green()
            tint_strength = 0.92
            result = mix(top_color, top_color * grass_green, tint_strength)
            return clamp(result, 0.0, 1.0)
        elif ny < 0.0:
            return texture(samplerDirt, uv, atlas_lod).rgb
        else:
            return texture(samplerGrassSide, uv, atlas_lod).rgb
            
    elif block_id == 5: # Leaves
        base = texture(samplerLeaves, uv, atlas_lod).rgb
        v = hash12(vec2(float(cell.x) * 0.17, float(cell.y) * 0.29))
        base = base * (0.82 + 0.28 * v)
        if ny < 0.0:
            base = base * 0.70
        grass_green = fetch_grass_side_green()
        base = mix(base, grass_green * length(base), 0.72)
        return base
        
    elif block_id == 4: # Wood
        if abs(ny) > 0.0:
            return texture(samplerWood, uv, atlas_lod).rgb
        else:
            return texture(samplerWoodSide, uv, atlas_lod).rgb
            
    elif block_id == 2: # Dirt
        return texture(samplerDirt, uv, atlas_lod).rgb
        
    elif block_id == 3: # Stone
        return texture(samplerStone, uv, atlas_lod).rgb
        
    else:
        return texture(samplerDefault, uv, atlas_lod).rgb

def fetch_grass_side_green():
    uv = vec2(110.0 / 256.0, 222.0 / 256.0)
    return texture(samplerGrassSide, uv).rgb

def hash12(p):
    return fract(sin(dot(p, vec2(12.9898, 78.233))) * 43758.5453)
```

**Parameters:**
- `atlasLod`: Fixed to `0.0` [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Precision:** The condition `abs(normal.x) > 0.5` assumes normalized normals [S1, unofficial].
- **Boundaries:** UV coordinates are not clamped in the `faceUV` function, relying on the texture sampler's wrapping mode or the atlas layout to handle boundaries [S1, unofficial].
- **Hardcoded Values:** The grass green tint uses a hardcoded UV `(110.0/256.0, 222.0/256.0)` which is fragile if the atlas layout changes [S1, unofficial].

## Subtopics

### Texture Atlas with Shader Wrapping
This variant handles the case where multiple block types share a single 2D atlas texture but require distinct UV regions [S2, unofficial]. It involves splitting vertices shared by different block types and assigning distinct texture coordinates to each vertex copy, representing the top-left corner of the texture in the atlas or its index in an array texture [S2, unofficial]. The shader must perform texture wrapping math manually because hardware sampling cannot handle arbitrary atlas regions directly [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ per fragment, involving a few comparisons, a hash lookup, and a texture sample [G].
- **Memory Complexity:** $O(1)$ additional memory, requiring only the texture samplers and a small amount of uniform data [G].
- **Strengths:** Efficient single-pass texturing for blocky worlds; reduces draw calls by using a single atlas [S2, unofficial].
- **Limitations:** Fixed LOD disables mip-mapping, potentially causing aliasing at distance; hardcoded UVs for specific colors (grass green) are brittle; hash-based variation is visible up close but may appear noisy [S1, unofficial].
- **When to use:** Real-time voxel rendering where performance is critical and block types are limited [G].
- **When not to use:** When high-quality mip-mapping is required or when the atlas layout is dynamic and cannot be hardcoded [G].

## Background
Originating from GLSL voxel world shaders, this method is used in real-time voxel engines to apply textures to blocky meshes efficiently [S1, unofficial]. It builds on [Texture Atlas UV Mapping](4238-texture-atlas-uv-mapping.md) by adding semantic block-type handling [S1, unofficial]. Documented in a GLSL voxel shader repository and a game development forum [S1, S2, unofficial].

## Sources
[S1] GitHub - razzle-reinizu/Voxel-World-Shader: A GLSL voxel-world shader ..., https://github.com/razzle-reinizu/Voxel-World-Shader, fetched 2026-10-25 (unofficial)
[S2] Texturing different block types on an optimized voxel mesh, https://gamedev.stackexchange.com/questions/185036, fetched 2026-10-30 (unofficial)
