# Chunk-Based Voxel Terrain Generation

Also called: Procedural voxel placement, Voxel Terrain Generation, Voxel Terrain Generation via Chunked Noise Sampling, VoxelChunk generation

## Overview
Chunk-Based Voxel Terrain Generation is a procedural method that divides a 3D world into fixed-size cubic chunks, populating each with voxel data via 3D noise sampling and biome logic [S1, S2, unofficial]. It differs from heightmap-based approaches by explicitly generating a dense 3D voxel grid per chunk before mesh extraction, allowing for overhangs and caves, while differing from raymarching methods by computing the discrete volume explicitly [S2, unofficial]. This method builds on [Chunk Generation with Base Terrain](1110-chunk-generation-with-base-terrain.md) by defining the specific procedural noise sampling and block placement rules used within each chunk.

## How it works
**Inputs:**
- World configuration: `seed`, `world_size` (X/Z extent), `max_elevation`, `water_level`, `equator_temp`, `temp_decay_coeff` [S1, unofficial].
- Chunk parameters: `CHUNK_SIZE` (8 or 32 voxels per side), `VOXEL_SIZE` (1 meter) [S1, S2, unofficial].
- Noise parameters: Amplitudes and frequencies for height, moisture, and wackiness noise [S1, unofficial].
- Player position: Determines which chunks to generate and load [S2, unofficial].

**Outputs:**
- A set of `VoxelChunk` objects, each containing a voxel array and a generated mesh [S1, S2, unofficial].
- Mesh geometry consisting of visible faces after culling internal and water-adjacent faces [S1, unofficial].

**Data Structures:**
1. **Voxel Array:** A 1D `Uint8Array` of size `CHUNK_LENGTH = CHUNK_SIZE^3` [S1, unofficial]. Index for voxel `(x,y,z)` is `x + y * CHUNK_SIZE + z * CHUNK_SIZE * CHUNK_SIZE` [S1, unofficial].
2. **Chunk Cache Item:** Contains `enc` (RLE encoded data), `faces` (Map of VoxelID to packed face data), `n` (face count), `is_empty` (boolean) [S1, unofficial].
3. **Packed Face Data:** An array `[x, y, z, voxel_id, direction_index]` for each visible face [S1, unofficial].
4. **Vertex Buffers:** `position_buffer` (Float32, `num_vertices * 3`), `vert_id_buffer` (Float32, `num_vertices`), `packed_buffer` (Float32, `num_vertices`), `index_buffer` (Uint16, `num_faces * 6`) [S1, unofficial].

**Steps:**
1. Initialize noise functions with configured amplitudes and frequencies [S1, unofficial].
2. Initialize a loading threshold sphere to trigger chunk unloading when the player exits [S2, unofficial].
3. For each chunk at coordinates $(C_x, C_y, C_z)$, calculate a chunk-level wackiness exponent [S1, unofficial].
4. Populate the voxel array by evaluating height noise, applying the wackiness exponent, and determining voxel types based on biome logic [S1, unofficial].
5. Run-length encode the voxel array and flag empty chunks [S1, unofficial].
6. Identify visible faces by checking all 6 neighbors for out-of-bounds, AIR, or WATER conditions [S1, unofficial].
7. Generate mesh geometry from visible faces and finalize the chunk [S1, S2, unofficial].

**Formulas:**
- Temperature: $T(z) = \text{clamp}(E - |z - Z_{eq}| \cdot D, 0, 1)$, where $E$ is `equator_temp`, $Z_{eq}$ is `world_size/2`, and $D$ is `temp_decay_coeff` [S1, unofficial].
- Generation Height: $H = \text{noise}_{\text{height}}(X_w, Z_w)^{E_{\text{chunk}}} \cdot \text{max\_elevation}$ [S1, unofficial].
- Chunk Wackiness Exponent: $E_{\text{chunk}}$ maps wackiness noise $W \in [0, 1]$ as follows: $W < 0.2 \rightarrow 16$, $0.2 \le W < 0.4 \rightarrow 8$, $0.4 \le W < 0.6 \rightarrow 2$, $W > 0.8 \rightarrow 0.5$, else $E_{\text{chunk}} = W$ [S1, unofficial].

**Parameters:**
- `CHUNK_SIZE`: 8 [S1, unofficial] or 32 [S2, unofficial].
- `VOXEL_SIZE`: 1 meter [S1, unofficial].
- `render_distance`: 256 meters [S2, unofficial].
- `cut_off`: -0.75 (noise threshold for voxel presence) [S2, unofficial].
- `terrain_scope`: 1.0 (noise zoom factor) [S2, unofficial].
- `smoothIterations`: 2 (blur passes, range 0-4) [S3, unofficial].
- `caveNoiseScale`: 0.07 (cave cavity scale) [S3, unofficial].
- `spawnRoomRadius`: 9 meters [S3, unofficial].
- `borderThickness`: 2 voxels [S3, unofficial].

**Edge Cases and Pitfalls:**
- Neighbor boundary check: Neighbors outside chunk bounds are treated as visible faces (exposed to void or adjacent chunk) [S1, unofficial].
- Water visibility: Faces adjacent to WATER voxels are rendered, treating water as transparent/empty for culling [S1, unofficial].
- Thread safety: `last_loading_position` must not be replaced by `loading_threshold.global_position` because it is read by worker threads; doing so causes runtime errors [S2, unofficial].
- Stale chunk processing: If the player moves during generation, the thread must immediately return to avoid generating mismatched chunks [S2, unofficial].
- Precision: Noise evaluation uses floating point; height calculation involves exponentiation which can amplify small noise differences [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_chunk(chunk_coords, config, noises):
    CHUNK_SIZE = config.CHUNK_SIZE
    VOXEL_SIZE = config.VOXEL_SIZE
    CHUNK_LENGTH = CHUNK_SIZE ** 3
    WATER_LEVEL = config.water_level
    MAX_ELEVATION = config.max_elevation
    EQ_TEMP = config.equator_temp
    TEMP_DECAY = config.temp_decay_coeff
    WORLD_SIZE = config.world_size
    Z_EQ = WORLD_SIZE / 2

    # 1. Calculate chunk wackiness exponent
    wackiness_val = noises.wackiness.eval(chunk_coords.x / 100, chunk_coords.z / 100)
    if wackiness_val < 0.2:
        chunk_exp = 16
    elif wackiness_val < 0.4:
        chunk_exp = 8
    elif wackiness_val < 0.6:
        chunk_exp = 2
    elif wackiness_val > 0.8:
        chunk_exp = 0.5
    else:
        chunk_exp = wackiness_val

    voxel_data = [0] * CHUNK_LENGTH
    FACE_DIRECTIONS = [[0,1,0], [0,-1,0], [1,0,0], [-1,0,0], [0,0,1], [0,0,-1]]

    # 2. Populate voxel array
    for x in range(CHUNK_SIZE):
        for z in range(CHUNK_SIZE):
            world_x = chunk_coords.x * CHUNK_SIZE * VOXEL_SIZE + x * VOXEL_SIZE
            world_z = chunk_coords.z * CHUNK_SIZE * VOXEL_SIZE + z * VOXEL_SIZE
            height_noise_val = noises.height.eval(world_x, world_z)
            gen_height = (height_noise_val ** chunk_exp) * MAX_ELEVATION

            for y in range(CHUNK_SIZE):
                world_y = chunk_coords.y * CHUNK_SIZE * VOXEL_SIZE + y * VOXEL_SIZE
                voxel_type = determine_voxel(gen_height, wackiness_val, world_x, world_y, world_z, noises, config)
                idx = x + y * CHUNK_SIZE + z * CHUNK_SIZE * CHUNK_SIZE
                voxel_data[idx] = voxel_type

    # 3. RLE encode and check empty
    rle_data = run_length_encode(voxel_data)
    is_empty = len(rle_data) == 2 and rle_data[1] == VOXEL_TYPES.AIR

    # 4. Generate mesh (visible faces)
    faces = []
    for x in range(CHUNK_SIZE):
        for y in range(CHUNK_SIZE):
            for z in range(CHUNK_SIZE):
                idx = x + y * CHUNK_SIZE + z * CHUNK_SIZE * CHUNK_SIZE
                v_type = voxel_data[idx]
                if v_type == VOXEL_TYPES.AIR or v_type == VOXEL_TYPES.WATER:
                    continue
                for d_idx, dx, dy, dz in enumerate(FACE_DIRECTIONS):
                    nx, ny, nz = x + dx, y + dy, z + dz
                    if not (0 <= nx < CHUNK_SIZE and 0 <= ny < CHUNK_SIZE and 0 <= nz < CHUNK_SIZE):
                        faces.append([x, y, z, v_type, d_idx])
                        continue
                    n_idx = nx + ny * CHUNK_SIZE + nz * CHUNK_SIZE * CHUNK_SIZE
                    n_type = voxel_data[n_idx]
                    if n_type == VOXEL_TYPES.AIR or n_type == VOXEL_TYPES.WATER:
                        faces.append([x, y, z, v_type, d_idx])

    return voxel_data, faces, is_empty

def determine_voxel(gen_height, wackiness, wx, wy, wz, noises, config):
    if wy < config.water_level:
        if wy >= gen_height: return VOXEL_TYPES.WATER
        if wy + 2 >= gen_height: return VOXEL_TYPES.SAND
        if wy + 8 >= gen_height: return VOXEL_TYPES.MUD
        if wy + 14 >= gen_height: return VOXEL_TYPES.SANDSTONE
        return VOXEL_TYPES.STONE
    if wy > gen_height: return VOXEL_TYPES.AIR

    moisture = noises.moisture.eval(wx, wz)
    eq_dist = abs(wz - config.world_size / 2)
    temperature = clamp(config.equator_temp - eq_dist * config.temp_decay_coeff, 0, 1)

    if moisture <= 0.3 and temperature >= 0.7:
        if wy + 3 >= gen_height: return VOXEL_TYPES.SAND
        if wy + 7 >= gen_height: return VOXEL_TYPES.SANDSTONE
        return VOXEL_TYPES.STONE
    if moisture <= 0.4 and temperature <= 0.3:
        if wy + 1 >= gen_height: return VOXEL_TYPES.SNOW
        if wy + 6 >= gen_height: return VOXEL_TYPES.PACKED_DIRT
        return VOXEL_TYPES.STONE
    if moisture > 0.7:
        if wackiness > 0.7 and temperature > 0.7:
            if wy + 1 >= gen_height: return VOXEL_TYPES.MYCELIUM
            if wy + 7 >= gen_height: return VOXEL_TYPES.DIRT
            return VOXEL_TYPES.STONE
        else:
            if wy + 1 >= gen_height: return VOXEL_TYPES.GRASS
            return VOXEL_TYPES.DIRT [G]
    return VOXEL_TYPES.DIRT [G]
```

## Subtopics

### [Chunk-based Voxel World Generation](4832-chunk-based-voxel-world-generation.md)
Generates a 3D voxel world by sampling 3D simplex noise to determine surface height and internal density, filtering blocks against water level and density thresholds. It differs from sibling methods by using a separate noise function for vertical density rather than a single heightmap, allowing for overhangs and caves, while maintaining a simple threshold-based placement logic [G]. It builds on [Chunk-Based Voxel Terrain Generation](4140-chunk-based-voxel-terrain-generation.md) by defining the specific procedural noise sampling and block placement rules used within each chunk [G].

## Operating characteristics
- **Time Complexity:** $O(C \cdot S^3)$ per chunk, where $C$ is the number of chunks and $S$ is chunk size [S1, S2, unofficial]. Mesh generation scales with the exposed surface area [S1, unofficial].
- **Memory Complexity:** $O(C \cdot S^3)$ for voxel data, plus $O(F)$ for mesh geometry, where $F$ is the number of visible faces [S1, S2, unofficial].
- **Performance:** Real-time generation is achieved via background threads and distance-sorted loading [S2, unofficial]. A chunk size of 8x8x8 generates quickly but increases per-chunk overhead, while 32x32x32 reduces overhead but increases per-chunk generation time [S1, S2, unofficial].
- **Strengths:** Simple implementation, supports complex biomes and verticality via 3D noise, efficient culling of internal faces [S1, S2, unofficial].
- **Limitations:** Dense voxel grid consumes significant memory for large worlds, noise artifacts may appear at chunk boundaries without smoothing, and thread safety requires careful state management [S1, S2, unofficial].
- **When to Use:** When a fully volumetric, explorable terrain with caves and overhangs is required, and performance constraints allow for dense voxel storage [S1, S2, unofficial].
- **When Not to Use:** When memory is extremely limited, or when terrain is strictly 2D heightmap-based [S1, S2, unofficial].

## Background
Originating from early voxel games and popularized by titles like *Minecraft*, chunk-based generation allows for infinite, procedurally generated worlds [S1, S2, S3, unofficial]. The specific implementation details cited here are derived from community projects: [S1, unofficial] provides a JavaScript/WebGL implementation with octaved noise and biome logic, [S2, unofficial] provides a Godot 4 C++ extension with threaded loading, and [S3, unofficial] provides a Unity implementation with density fields and deformation [S1, S2, S3, unofficial].

## Sources
[S1] web_voxel_terrain, https://github.com/andrewbloese-00/web_voxel_terrain, fetched 2026-10-27 (unofficial)
[S2] voxel-terrain-plugin, https://github.com/Ry645/voxel-terrain-plugin, fetched 2026-10-27 (unofficial)
[S3] UnityVoxelTerrain, https://github.com/Rabidus4k/UnityVoxelTerrain, fetched 2026-10-28 (unofficial)
