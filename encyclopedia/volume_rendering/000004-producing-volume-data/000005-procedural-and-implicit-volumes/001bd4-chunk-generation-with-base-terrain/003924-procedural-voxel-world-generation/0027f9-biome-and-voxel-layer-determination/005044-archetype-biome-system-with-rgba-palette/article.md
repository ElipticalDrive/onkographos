# Archetype Biome System with RGBA Palette

Also called: Archetype Biome System

## Overview
The Archetype Biome System with RGBA Palette is a procedural voxel generation method that assigns material properties to voxels based on a shared set of six archetypes (rock, regolith, earth, ice, lava, dune) mapped across multiple celestial bodies. It solves the problem of managing distinct biome aesthetics and physics for 16 different bodies without duplicating generation logic, by decoupling the structural archetype from the visual palette. It differs from sibling methods by using a unified archetype layering system with per-body color/atmosphere overrides and an emissive RGBA palette, rather than unique systems per body or simple distance-based biome selection.

## How it works
**Inputs:**
- `BiomeProfile`: A shared configuration object containing base colors, atmosphere parameters, and physics constants for each of the 6 archetypes.
- `BodyID`: Identifier for the specific celestial body (16 total).
- `genHeight`: The generated terrain surface height for each voxel coordinate.
- `BiomeParams`: Moisture, temperature, and wackiness values derived from the parent [Biome and Voxel Layer Determination](4167-biome-and-voxel-layer-determination.md) system.

**Outputs:**
- `voxelTypes`: A 3D grid of voxel IDs representing the material type.
- `palette`: An RGBA color map where each voxel ID maps to a color with an emissive channel.

**Data Structures:**
- `Archetype`: Enum or integer ID (0-5) representing one of: Rock, Regolith, Earth, Ice, Lava, Dune.
- `PaletteEntry`: Struct with fields `r`, `g`, `b`, `e` (emissive), all 8-bit unsigned integers.
- `VoxelData`: Struct or packed integer containing `type` (Archetype ID) and `paletteIndex`.

**Steps:**
1. **Map Archetype to Body:** For the given `BodyID`, select the primary archetype distribution. Each body has a specific mapping of the 6 archetypes to its terrain features.
2. **Determine Voxel Type:** For each voxel at `(x, y, z)`:
   - Calculate height relative to `genHeight`.
   - Apply biome parameters (moisture, temperature) to refine the archetype selection.
   - Assign `voxelTypes[x][y][z]` to one of the 6 archetypes.
3. **Generate Natural Features:** Apply archetype-specific layering rules:
   - **Earth:** Fill lowlands with water (Earth archetype variant).
   - **Titan:** Generate rippling dunes (Dune archetype).
   - **Moon/Mercury:** Generate impact craters (bowl + rim structure using Rock/Regolith).
   - **Europa:** Generate luminous ice veins and larger caverns (Ice archetype with high emissive).
   - **Io:** Flood deep caverns with lava (Lava archetype).
4. **Assign Palette:** For each `voxelType`, look up the color from the `BiomeProfile` adjusted for the `BodyID`.
   - `palette[voxelType] = BiomeProfile[BodyID][Archetype]`
   - The palette stride is RGBA, where the alpha channel is not used for transparency but the emissive channel (`e`) is used for self-illumination.
5. **Mesh Generation:** Pass the `voxelTypes` and `palette` to the mesh pipeline.
   - The greedy meshing algorithm must carry a per-vertex emissive channel.
   - The voxel material shader reads the emissive channel to self-illuminate lava and glowing ice in shadowed areas (caves), preventing washed-out lit surfaces.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_biome(body_id, gen_height_grid, biome_params):
    # 1. Get archetype mapping for this body
    archetype_map = get_body_archetype_mapping(body_id)
    
    # 2. Initialize voxel grid
    voxel_types = empty_grid(shape=gen_height_grid.shape, dtype=int)
    
    # 3. Determine voxel types based on height and biome params
    for x, y, z in all_voxels:
        height = gen_height_grid[x, y, z]
        moisture = biome_params.moisture[x, y, z]
        temperature = biome_params.temperature[x, y, z]
        
        # Determine archetype based on height and biome params
        # This logic is inherited from parent [Biome and Voxel Layer Determination]
        archetype = determine_archetype(height, moisture, temperature, archetype_map)
        voxel_types[x, y, z] = archetype
        
    # 4. Apply natural features (post-processing step)
    apply_natural_features(voxel_types, body_id)
    
    # 5. Generate palette
    palette = {}
    for archetype_id in range(6):
        # Get base color from shared BiomeProfile
        base_color = BiomeProfile[archetype_id]
        # Adjust for body-specific atmosphere/lighting
        body_color = adjust_color_for_body(base_color, body_id)
        # Set emissive channel
        emissive = get_emissive_for_archetype(archetype_id, body_id)
        palette[archetype_id] = RGBA(body_color.r, body_color.g, body_color.b, emissive)
        
    return voxel_types, palette

def apply_natural_features(voxel_types, body_id):
    if body_id == "Earth":
        fill_lowlands_with_water(voxel_types)
    elif body_id == "Titan":
        generate_dunes(voxel_types)
    elif body_id in ["Moon", "Mercury"]:
        generate_craters(voxel_types)
    elif body_id == "Europa":
        generate_ice_veins(voxel_types)
    elif body_id == "Io":
        flood_lava_lakes(voxel_types)
```

**Parameters:**
- `num_archetypes`: 6 (Rock, Regolith, Earth, Ice, Lava, Dune).
- `num_bodies`: 16.
- `palette_stride`: 4 bytes (RGBA).

**Edge Cases:**
- **Spawn Placement:** Ensure spawn point is above any sea/lava surface. This requires a post-generation check or constraint during generation.
- **Emissive Washout:** If emissive values are too high, lit surfaces may appear washed out. The shader must correctly blend emissive with ambient light.

## Subtopics

### [Voronoi Biome Blending](19530-voronoi-biome-blending.md)
Uses Voronoi cells for macro-biomes and smoothstep interpolation for seamless transitions, determining biome selection through macro noise rather than simple distance. Choose when contiguous macro-climates are needed.

## Operating characteristics
- **Time Complexity:** O(N) where N is the number of voxels, as each voxel is processed once for type assignment and palette lookup.
- **Memory Complexity:** O(N) for the voxel grid and palette. The palette is small (6 entries), so memory is dominated by the voxel grid.
- **Performance:** Efficient due to shared archetype logic across 16 bodies. No unique system per body reduces code complexity and cache misses.
- **Strengths:** Scalable to many bodies with distinct aesthetics. Emissive palette allows for dynamic lighting effects (glowing ice/lava) without complex shader logic.
- **Limitations:** Less flexible for highly unique biome features that don't fit the 6 archetypes. Natural features are hardcoded per body.

## Background
The Archetype Biome System with RGBA Palette was introduced in Phase 9 of the Solar System voxel exploration project. It was designed to allow 16 landable bodies to be explorable with matching voxel biomes, using 6 archetypes differentiated by per-body color, atmosphere, and physics. The system was implemented to avoid creating a unique system per body, reducing development overhead. The emissive RGBA palette was added to support self-illumination of lava and glowing ice, allowing Io and Europa to glow without washing out lit surfaces.

## Sources
[S1] Phase 9: Voxel Surface Exploration, https://github.com/LundstedtAdam/SolarSystem/pull/4, fetched 2026-10-28 (unofficial)
