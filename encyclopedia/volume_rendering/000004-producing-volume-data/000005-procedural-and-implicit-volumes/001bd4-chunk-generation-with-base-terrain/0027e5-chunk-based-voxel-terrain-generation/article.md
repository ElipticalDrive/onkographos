# Chunk-Based Voxel Terrain Generation

Also called: Procedural voxel placement, Voxel Terrain Generation, Voxel Terrain Generation via Chunked Noise Sampling, VoxelChunk generation

## Overview
Chunk-Based Voxel Terrain Generation is a procedural technique that divides a 3D world into fixed-size cubic chunks and populates each chunk with voxel data by sampling multi-octave noise functions for height, moisture, and temperature [S1, unofficial]. It differs from static generation by organizing output into fixed-size chunks for efficient memory management and localized updates, and from pure implicit volumes by explicitly discretizing space into a grid of typed voxels rather than a continuous field [S1, unofficial].

## How it works
**Inputs:**
- World seed (integer).
- Configuration parameters: `world_size` (X/Z extent), `max_elevation`, `water_level`, `equator_temp`, `temp_decay_coeff` [S1, unofficial].
- Chunk coordinates $(C_x, C_y, C_z)$ defining the chunk origin in world space [S1, unofficial].

**Outputs:**
- A chunk data structure containing:
  - `enc`: A `Uint16Array` of Run-Length Encoded (RLE) voxel type data [S1, unofficial].
  - `faces`: A `Map<VoxelID, number[]>` where each entry contains packed face data (x, y, z, voxel_id, direction) [S1, unofficial].
  - `n`: Total face count [S1, unofficial].
  - `is_empty`: Boolean flag [S1, unofficial].

**Data Structures:**
- **Voxel Types:** Enumerated integers (e.g., AIR=0, GRASS=1, STONE=6, WATER=5) [S1, unofficial].
- **Chunk Dimensions:** `CHUNK_SIZE` (default 8) [S1, unofficial]. `CHUNK_LENGTH` = $CHUNK\_SIZE^3$ (512) [S1, unofficial].
- **Face Directions:** Array of 6 unit vectors: $[0,1,0], [0,-1,0], [1,0,0], [-1,0,0], [0,0,1], [0,0,-1]$ [S1, unofficial].
- **Packed Face Data:** Array `[x, y, z, voxel_id, direction_index]` [S1, unofficial].

**Parameters:**
- `CHUNK_SIZE`: 8 [S1, unofficial].
- `VOXEL_SIZE`: 1 meter [S1, unofficial].
- `CHUNK_WORLD_SIZE`: $CHUNK\_SIZE \times VOXEL\_SIZE$ [S1, unofficial].
- `max_elevation`: 255 [S1, unofficial].
- `water_level`: 40 [S1, unofficial].
- `equator_temp`: 1.0 [S1, unofficial].
- `temp_decay_coeff`: 0.1 [S1, unofficial].
- `world_size`: 1000 [S1, unofficial].

**Algorithm Steps:**

1. **Initialize Noise Functions:**
   - Create height noise from `seed`.
   - Create moisture seed by shuffling `seed` by 10. Create moisture noise from this seed.
   - Create wackiness seed by shuffling moisture seed by 10. Create wackiness noise from this seed [S1, unofficial].
   - Initialize `OctavedNoise2d` instances:
     - Height: amplitude 1.2, frequency 400.
     - Moisture: amplitude 1.0, frequency 200.
     - Wackiness: amplitude 1.0, frequency 20 [S1, unofficial].

2. **Calculate Chunk-Level Wackiness Factor:**
   - Evaluate wackiness noise at $(C_x/100, C_z/100)$ [S1, unofficial].
   - Map result to exponent `chunk_exp`:
     - If $< 0.2$: `chunk_exp` = 16.
     - If $< 0.4$: `chunk_exp` = 8.
     - If $< 0.6$: `chunk_exp` = 2.
     - If $> 0.8$: `chunk_exp` = 0.5.
     - Else: `chunk_exp` = raw value [S1, unofficial].

3. **Generate Voxel Data:**
   - Allocate `Uint8Array` of size `CHUNK_LENGTH` [S1, unofficial].
   - Iterate $x$ from 0 to $CHUNK\_SIZE-1$, $z$ from 0 to $CHUNK\_SIZE-1$:
     - Calculate world X: $W_x = C_x \times CHUNK\_WORLD\_SIZE + x \times VOXEL\_SIZE$.
     - Calculate world Z: $W_z = C_z \times CHUNK\_WORLD\_SIZE + z \times VOXEL\_SIZE$.
     - Evaluate height noise: $H = \text{height\_noise}(W_x, W_z)$ [S1, unofficial].
     - Calculate terrain height: $genHeight = H^{\text{chunk\_exp}} \times max\_elevation$ [S1, unofficial].
     - Iterate $y$ from 0 to $CHUNK\_SIZE-1$:
       - Calculate world Y: $W_y = C_y \times CHUNK\_WORLD\_SIZE + y \times VOXEL\_SIZE$.
       - Determine voxel type using `determineGeneratedVoxel(genHeight, wackiness, (W_x, W_y, W_z))` [S1, unofficial].
       - Store type in array at index $x + y \times CHUNK\_SIZE + z \times CHUNK\_SIZE^2$ [S1, unofficial].

4. **Determine Voxel Type (`determineGeneratedVoxel`):**
   - **Underwater Check:** If $W_y < water\_level$:
     - If $W_y \ge genHeight$: return WATER.
     - If $W_y + 2 \ge genHeight$: return SAND.
     - If $W_y + 8 \ge genHeight$: return MUD.
     - If $W_y + 14 \ge genHeight$: return SANDSTONE.
     - Else: return STONE [S1, unofficial].
   - **Air Check:** If $W_y > genHeight$: return AIR [S1, unofficial].
   - **Biome Logic:**
     - Calculate moisture $M = \text{moisture\_noise}(W_x, W_z)$ [S1, unofficial].
     - Calculate temperature $T$:
       - $eq\_dist = |W_z - world\_size/2|$ [S1, unofficial].
       - $T = \text{clamp}(equator\_temp - eq\_dist \times temp\_decay\_coeff, 0, 1)$ [S1, unofficial].
     - **Desert:** If $M \le 0.3$ and $T \ge 0.7$:
       - If $W_y + 3 \ge genHeight$: return SAND.
       - If $W_y + 7 \ge genHeight$: return SANDSTONE.
       - Else: return STONE [S1, unofficial].
     - **Tundra:** If $M \le 0.4$ and $T \le 0.3$:
       - If $W_y + 1 \ge genHeight$: return SNOW.
       - If $W_y + 6 \ge genHeight$: return PACKED_DIRT.
       - Else: return STONE [S1, unofficial].
     - **Jungle/Mushroom:** If $M > 0.7$ and wackiness > 0.7 and $T > 0.7$:
       - If $W_y + 1 \ge genHeight$: return MYCELIUM.
       - If $W_y + 7 \ge genHeight$: return DIRT.
       - Else: return STONE [S1, unofficial].
     - **Default (Jungle/Forest):**
       - If $W_y + 1 \ge genHeight$: return GRASS.
       - If $W_y + 6 \ge genHeight$: return DIRT.
       - Else: return STONE [S1, unofficial].

5. **Encode Voxel Data:**
   - Run-length encode the `Uint8Array` of voxel types into `enc` [S1, unofficial].
   - If RLE length is 2 and second element is AIR, flag `is_empty` = true [S1, unofficial].

6. **Identify Visible Faces:**
   - Iterate all voxels in the chunk.
   - For each non-AIR/non-WATER voxel:
     - Check all 6 neighbors (using `FACE_DIRECTIONS`) [S1, unofficial].
     - If neighbor is out-of-bounds (OOB) or is AIR/WATER, the face is visible [S1, unofficial].
     - Pack face data $(x, y, z, \text{voxel\_id}, \text{direction\_index})$ and add to `faces` map keyed by voxel ID [S1, unofficial].

7. **Generate Mesh Geometry:**
   - For each visible face in `faces`:
     - Create 4 vertices (quad corners) [S1, unofficial].
     - Store position (placeholder 0,0,0), vertex ID (0-3), and packed face data in buffers [S1, unofficial].
     - Generate 2 triangles (6 indices) per face using vertex indices [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_chunk(chunk_coords, config, noise_functions):
    CHUNK_SIZE = 8
    CHUNK_LENGTH = CHUNK_SIZE ** 3
    VOXEL_SIZE = 1
    CHUNK_WORLD_SIZE = CHUNK_SIZE * VOXEL_SIZE
    
    # Step 2: Calculate Chunk-Level Wackiness
    wackiness_val = noise_functions['wackiness'].eval(chunk_coords.x / 100, chunk_coords.z / 100)
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

    # Step 3: Generate Voxel Data
    voxel_data = [0] * CHUNK_LENGTH
    for x in range(CHUNK_SIZE):
        world_x = chunk_coords.x * CHUNK_WORLD_SIZE + x * VOXEL_SIZE
        for z in range(CHUNK_SIZE):
            world_z = chunk_coords.z * CHUNK_WORLD_SIZE + z * VOXEL_SIZE
            height_noise_val = noise_functions['height'].eval(world_x, world_z)
            gen_height = (height_noise_val ** chunk_exp) * config['max_elevation']
            
            for y in range(CHUNK_SIZE):
                world_y = chunk_coords.y * CHUNK_WORLD_SIZE + y * VOXEL_SIZE
                voxel_type = determine_voxel_type(gen_height, wackiness_val, world_x, world_y, world_z, config, noise_functions)
                index = x + y * CHUNK_SIZE + z * CHUNK_SIZE ** 2
                voxel_data[index] = voxel_type

    # Step 5: Encode
    enc = run_length_encode(voxel_data)
    is_empty = len(enc) == 2 and enc[1] == VOXEL_TYPES['AIR']

    # Step 6: Identify Visible Faces
    faces = {}
    for x in range(CHUNK_SIZE):
        for y in range(CHUNK_SIZE):
            for z in range(CHUNK_SIZE):
                voxel_type = voxel_data[x + y * CHUNK_SIZE + z * CHUNK_SIZE ** 2]
                if voxel_type in [VOXEL_TYPES['AIR'], VOXEL_TYPES['WATER']]:
                    continue
                
                for dir_idx, (dx, dy, dz) in enumerate(FACE_DIRECTIONS):
                    nx, ny, nz = x + dx, y + dy, z + dz
                    is_visible = False
                    if 0 <= nx < CHUNK_SIZE and 0 <= ny < CHUNK_SIZE and 0 <= nz < CHUNK_SIZE:
                        neighbor_type = voxel_data[nx + ny * CHUNK_SIZE + nz * CHUNK_SIZE ** 2]
                        if neighbor_type in [VOXEL_TYPES['AIR'], VOXEL_TYPES['WATER']]:
                            is_visible = True
                    else:
                        is_visible = True # OOB is visible
                    
                    if is_visible:
                        voxel_id = x + y * CHUNK_SIZE + z * CHUNK_SIZE ** 2
                        packed_data = [x, y, z, voxel_type, dir_idx]
                        if voxel_id not in faces:
                            faces[voxel_id] = []
                        faces[voxel_id].append(packed_data)

    return {
        'enc': enc,
        'faces': faces,
        'n': sum(len(v) for v in faces.values()),
        'is_empty': is_empty
    }

def determine_voxel_type(gen_height, wackiness, wx, wy, wz, config, noise_functions):
    water_level = config['water_level']
    world_size = config['world_size']
    equator_temp = config['equator_temp']
    temp_decay = config['temp_decay_coeff']
    
    # Underwater
    if wy < water_level:
        if wy >= gen_height: return VOXEL_TYPES['WATER']
        if wy + 2 >= gen_height: return VOXEL_TYPES['SAND']
        if wy + 8 >= gen_height: return VOXEL_TYPES['MUD']
        if wy + 14 >= gen_height: return VOXEL_TYPES['SANDSTONE']
        return VOXEL_TYPES['STONE']
    
    # Air
    if wy > gen_height: return VOXEL_TYPES['AIR']
    
    # Biome
    moisture = noise_functions['moisture'].eval(wx, wz)
    eq_dist = abs(wz - world_size / 2)
    temperature = clamp(equator_temp - eq_dist * temp_decay, 0, 1)
    
    # Desert
    if moisture <= 0.3 and temperature >= 0.7:
        if wy + 3 >= gen_height: return VOXEL_TYPES['SAND']
        if wy + 7 >= gen_height: return VOXEL_TYPES['SANDSTONE']
        return VOXEL_TYPES['STONE']
    
    # Tundra
    if moisture <= 0.4 and temperature <= 0.3:
        if wy + 1 >= gen_height: return VOXEL_TYPES['SNOW']
        if wy + 6 >= gen_height: return VOXEL_TYPES['PACKED_DIRT']
        return VOXEL_TYPES['STONE']
    
    # Jungle/Mushroom
    if moisture > 0.7 and wackiness > 0.7 and temperature > 0.7:
        if wy + 1 >= gen_height: return VOXEL_TYPES['MYCELIUM']
        if wy + 7 >= gen_height: return VOXEL_TYPES['DIRT']
        return VOXEL_TYPES['STONE']
    
    # Default
    if wy + 1 >= gen_height: return VOXEL_TYPES['GRASS']
    if wy + 6 >= gen_height: return VOXEL_TYPES['DIRT']
    return VOXEL_TYPES['STONE']

def clamp(val, min_val, max_val):
    return max(min_val, min(val, max_val))
```

## Subtopics

### [Chunk-based Voxel World Generation](4832-chunk-based-voxel-world-generation.md)
Chunk-based Voxel World Generation is a procedural technique that constructs a 3D voxel world by sampling 3D noise functions to determine block density and height, then filtering these values against thresholds to place solid blocks. It differs from static generation by organizing the output into fixed-size chunks, enabling efficient memory management and localized updates, and from pure implicit volumes by explicitly discretizing the space into a grid of binary or typed voxels rather than a continuous field.

### [Dense Voxel Terrain Generation](16749-dense-voxel-terrain-generation.md)
Dense Voxel Terrain Generation is a GPU-accelerated technique that produces a dense 3D voxel grid representing terrain by evaluating mathematical functions to determine if each voxel is empty or full. It differs from sparse or chunk-based storage methods by allocating a contiguous array for every voxel in the chunk, enabling simple linear indexing and direct compute shader dispatch, at the cost of higher memory usage for large volumes.

## Operating characteristics
- **Time Complexity:** $O(C^3)$ per chunk, where $C$ is `CHUNK_SIZE`. Noise evaluation is $O(1)$ per voxel. RLE encoding is $O(C^3)$. Face culling is $O(C^3)$ with constant neighbor checks.
- **Memory Complexity:** $O(C^3)$ for voxel data. RLE compression reduces this for homogeneous chunks. Face data scales with surface area $O(C^2)$.
- **Measured Performance:** Implemented in WebGL (S1) and Godot C++ (S2). S1 uses 8x8x8 chunks. S2 uses 32x32x32 chunks. S2 employs background threads for loading and distance-sorted processing to maintain frame rates during player movement [S2, unofficial].
- **Strengths:** Localized updates allow efficient memory management. RLE compression reduces storage for large empty or uniform areas. Face culling reduces vertex count significantly compared to naive meshing.
- **Limitations:** Chunk boundaries can cause visual seams if noise is not continuous across chunk borders (not explicitly addressed in S1, but implied by world-space noise sampling). Water visibility treats water as transparent/empty for culling, which may not be desired for opaque water rendering [S1, unofficial]. Thread safety requires careful management of player position updates to avoid stale chunk generation [S2, unofficial].
- **When to use:** Real-time procedural terrain generation where memory efficiency and dynamic loading are critical. Suitable for voxel-based games with large worlds.
- **When not to use:** When continuous surface rendering (e.g., marching cubes) is required without voxel discretization. When high-frequency detail across large areas is needed without chunking artifacts.

## Background
Originating from procedural generation techniques in voxel-based games, this method was implemented in web-based WebGL environments (S1) and Godot engine C++ extensions (S2) around 2025-2026. It builds on the concept of chunked terrain management to handle infinite or large worlds efficiently. The use of octaved noise for height and separate noise for moisture/wackiness allows for diverse biome generation. The technique is used in real-time procedural terrain loading plugins [S2, unofficial].

## Sources
[S1] andrewbloese-00/web_voxel_terrain, https://github.com/andrewbloese-00/web_voxel_terrain, fetched 2026-10-27 (unofficial)
[S2] Ry645/voxel-terrain-plugin, https://github.com/Ry645/voxel-terrain-plugin, fetched 2026-10-27 (unofficial)
[S3] Rabidus4k/UnityVoxelTerrain, https://github.com/Rabidus4k/UnityVoxelTerrain, fetched 2026-10-28 (unofficial)
