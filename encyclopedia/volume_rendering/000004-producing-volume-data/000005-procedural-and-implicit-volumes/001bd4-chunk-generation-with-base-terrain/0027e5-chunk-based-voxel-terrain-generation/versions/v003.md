# Chunk-Based Voxel Terrain Generation

Also called: Procedural voxel placement, Voxel Terrain Generation, Voxel Terrain Generation via Chunked Noise Sampling, VoxelChunk generation

## Overview
Chunk-Based Voxel Terrain Generation is a procedural method that divides a 3D world into fixed-size cubic chunks, populating each with voxel data via 3D noise sampling and biome logic [S1, S2, unofficial]. It differs from heightmap-based approaches by explicitly generating a dense 3D voxel grid per chunk before mesh extraction, allowing for overhangs and caves, while differing from raymarching methods by computing the discrete volume explicitly [S2, unofficial]. This method builds on [Chunk Generation with Base Terrain](1110-chunk-generation-with-base-terrain.md) by defining the specific procedural noise sampling and block placement rules used within each chunk.

## How it works
**Inputs:**
- World configuration: `seed`, `world_size` (X/Z extent), `max_elevation`, `water_level`, `equator_temp`, `temp_decay_coeff` [S1, unofficial].
- Chunk parameters: `CHUNK_SIZE` (e.g., 8 or 32 voxels per side), `VOXEL_SIZE` (e.g., 1 meter) [S1, S2, unofficial].
- Noise parameters: Frequencies and amplitudes for height, moisture, and wackiness noise [S1, unofficial].
- Player position: Used to determine which chunks to generate and load [S2, unofficial].

**Outputs:**
- A set of `VoxelChunk` objects, each containing a voxel array and a generated mesh [S1, S2, unofficial].
- The mesh geometry consists of visible faces (culling internal and water-adjacent faces) [S1, unofficial].

**Data Structures:**
1.  **Voxel Array:** A 1D `Uint8Array` of size `CHUNK_LENGTH = CHUNK_SIZE^3` [S1, unofficial]. Index `i` for voxel `(x,y,z)` is `x + y*CHUNK_SIZE + z*CHUNK_SIZE^2` [S1, unofficial].
2.  **Chunk Cache Item:** Contains `enc` (RLE encoded data), `faces` (Map of VoxelID to packed face data), `n` (face count), `is_empty` (boolean) [S1, unofficial].
3.  **Packed Face Data:** An array `[x, y, z, voxel_id, direction_index]` for each visible face [S1, unofficial].
4.  **Vertex Buffers:** `position_buffer` (Float32, `num_vertices * 3`), `vert_id_buffer` (Float32, `num_vertices`), `packed_buffer` (Float32, `num_vertices`), `index_buffer` (Uint16, `num_faces * 6`) [S1, unofficial].

**Steps:**

1.  **Initialize Noise Functions:**
    - Create height noise from `seed` [S1, unofficial].
    - Create moisture noise from `seed` shuffled by 10 [S1, unofficial].
    - Create wackiness noise from moisture seed shuffled by 10 [S1, unofficial].
    - Configure `OctavedNoise2d` instances:
        - Height: amplitude 1.2, frequency 400 [S1, unofficial].
        - Moisture: amplitude 1.0, frequency 200 [S1, unofficial].
        - Wackiness: amplitude 1.0, frequency 20 [S1, unofficial].
    - Define temperature function: $T(z) = \text{clamp}(E - |z - Z_{eq}| \cdot D, 0, 1)$, where $E$ is `equator_temp`, $Z_{eq}$ is `world_size/2`, and $D$ is `temp_decay_coeff` [S1, unofficial].

2.  **Initialize Loading Threshold (Optional/Engine Specific):**
    - Create an `Area3D` with a `SphereShape3D` [S2, unofficial].
    - Set radius: $R = \text{render\_distance} \times 0.25$ [S2, unofficial].
    - Connect `body_exited` signal to trigger chunk unloading [S2, unofficial].

3.  **Per-Chunk Generation Loop:**
    For each chunk at coordinates $(C_x, C_y, C_z)$:
    
    3.1. **Calculate Chunk Wackiness:**
    - Evaluate wackiness noise at $(C_x/100, C_z/100)$ [S1, unofficial].
    - Map result $W$ to exponent $E_{chunk}$:
        - If $W < 0.2$, $E_{chunk} = 16$ [S1, unofficial].
        - If $0.2 \le W < 0.4$, $E_{chunk} = 8$ [S1, unofficial].
        - If $0.4 \le W < 0.6$, $E_{chunk} = 2$ [S1, unofficial].
        - If $W > 0.8$, $E_{chunk} = 0.5$ [S1, unofficial].
        - Else, $E_{chunk} = W$ [S1, unofficial].

    3.2. **Populate Voxel Array:**
    - Initialize `voxelData` array of size `CHUNK_LENGTH` [S1, unofficial].
    - For each voxel $(x, y, z)$ in chunk ($0 \le x,y,z < \text{CHUNK\_SIZE}$):
        - Calculate world coordinates:
          $X_w = C_x \cdot \text{CHUNK\_SIZE} + x$
          $Z_w = C_z \cdot \text{CHUNK\_SIZE} + z$
          $Y_w = C_y \cdot \text{CHUNK\_SIZE} + y$
        - Evaluate height noise at $(X_w, Z_w)$ [S1, unofficial].
        - Calculate generation height: $H = \text{noise\_height}(X_w, Z_w)^{E_{chunk}} \cdot \text{max\_elevation}$ [S1, unofficial].
        - Determine voxel type using `determineGeneratedVoxel(H, W, (X_w, Y_w, Z_w))` [S1, unofficial]:
            - **Underwater** ($Y_w < \text{water\_level}$):
                - If $Y_w \ge H$: WATER [S1, unofficial].
                - If $Y_w + 2 \ge H$: SAND [S1, unofficial].
                - If $Y_w + 8 \ge H$: MUD [S1, unofficial].
                - If $Y_w + 14 \ge H$: SANDSTONE [S1, unofficial].
                - Else: STONE [S1, unofficial].
            - **Above Surface** ($Y_w > H$): AIR [S1, unofficial].
            - **Surface/Underground**:
                - Evaluate moisture $M = \text{noise\_moisture}(X_w, Z_w)$ [S1, unofficial].
                - Evaluate temperature $T = T(Z_w)$ [S1, unofficial].
                - **Desert** ($M \le 0.3, T \ge 0.7$):
                    - If $Y_w + 3 \ge H$: SAND [S1, unofficial].
                    - If $Y_w + 7 \ge H$: SANDSTONE [S1, unofficial].
                    - Else: STONE [S1, unofficial].
                - **Tundra** ($M \le 0.4, T \le 0.3$):
                    - If $Y_w + 1 \ge H$: SNOW [S1, unofficial].
                    - If $Y_w + 6 \ge H$: PACKED_DIRT [S1, unofficial].
                    - Else: STONE [S1, unofficial].
                - **Jungle/Mushroom** ($M > 0.7$):
                    - If $W > 0.7$ and $T > 0.7$ (Mushroom):
                        - If $Y_w + 1 \ge H$: MYCELIUM [S1, unofficial].
                        - If $Y_w + 7 \ge H$: DIRT [S1, unofficial].
                        - Else: STONE [S1, unofficial].
                    - Else (Jungle):
                        - If $Y_w + 1 \ge H$: GRASS [S1, unofficial].
                        - Else: DIRT [S1, unofficial] (Note: Source code snippet cuts off, but implies standard dirt below grass) [S1, unofficial].
        - Store voxel type in `voxelData` at index $x + y \cdot \text{CHUNK\_SIZE} + z \cdot \text{CHUNK\_SIZE}^2$ [S1, unofficial].

    3.3. **Encode and Check Empty:**
    - Run-length encode `voxelData` [S1, unofficial].
    - If encoded length is 2 and second element is AIR, flag chunk as empty [S1, unofficial].

    3.4. **Generate Mesh (Visible Faces):**
    - Initialize buffers [S1, unofficial].
    - For each voxel $(x,y,z)$ with type $V \neq \text{AIR}$ and $V \neq \text{WATER}$:
        - For each of 6 directions $D \in \{[1,0,0], [-1,0,0], [0,1,0], [0,-1,0], [0,0,1], [0,0,-1]\}$:
            - Calculate neighbor coordinates $(nx, ny, nz) = (x,y,z) + D$ [S1, unofficial].
            - If neighbor is out-of-bounds ($<0$ or $\ge \text{CHUNK\_SIZE}$) OR neighbor voxel is AIR or WATER:
                - Face is visible [S1, unofficial].
                - Pack face data: $[x, y, z, V, \text{direction\_index}]$ [S1, unofficial].
                - Add to `faces` map [S1, unofficial].
    - For each visible face:
        - Create 4 vertices for the quad [S1, unofficial].
        - Generate 2 triangles (6 indices) using vertex indices [S1, unofficial].
        - Store positions and packed data in buffers [S1, unofficial].

    3.5. **Finalize Chunk:**
    - Instantiate `VoxelChunk` with mesh and voxel data [S2, unofficial].
    - Add to scene tree [S2, unofficial].

**Parameters:**
- `CHUNK_SIZE`: 8 [S1, unofficial] or 32 [S2, unofficial].
- `VOXEL_SIZE`: 1 meter [S1, unofficial].
- `render_distance`: 256 meters [S2, unofficial].
- `cut_off`: -0.75 (for alternative noise-based placement) [S2, unofficial].
- `terrain_scope`: 1.0 (noise zoom factor) [S2, unofficial].
- `smoothIterations`: 2 (blur passes, range 0-4) [S3, unofficial].
- `caveNoiseScale`: 0.07f (for cave generation) [S3, unofficial].
- `spawnRoomRadius`: 9 meters [S3, unofficial].
- `borderThickness`: 2 voxels [S3, unofficial].

**Edge Cases and Pitfalls:**
- **Neighbor Boundary Check:** Neighbors outside chunk bounds are treated as visible faces (exposed to void/other chunk) [S1, unofficial].
- **Water Visibility:** Faces adjacent to WATER are rendered, treating water as transparent/empty for culling [S1, unofficial].
- **Thread Safety:** `last_loading_position` must not be replaced by `loading_threshold.global_position` directly in worker threads to avoid race conditions [S2, unofficial].
- **Stale Chunk Processing:** If player moves during generation, the thread must abort to avoid generating mismatched chunks [S2, unofficial].
- **Precision:** Noise evaluation uses floating point; height calculation involves exponentiation which can amplify small noise differences [S1, unofficial].

## Subtopics

### [Chunk-based Voxel World Generation](4832-chunk-based-voxel-world-generation.md)
Generates a 3D voxel world by sampling 3D simplex noise to determine surface height and internal density, filtering blocks against water level and density thresholds. It differs from sibling methods by using a separate noise function for vertical density rather than a single heightmap, allowing for overhangs and caves, while maintaining a simple threshold-based placement logic. It builds on [Chunk-Based Voxel Terrain Generation](4140-chunk-based-voxel-terrain-generation.md) by defining the specific procedural noise sampling and block placement rules used within each chunk.

## Operating characteristics
- **Time Complexity:** $O(C \cdot S^3)$ per chunk, where $C$ is the number of chunks and $S$ is chunk size. Mesh generation is proportional to the surface area of the terrain within the chunk.
- **Memory Complexity:** $O(C \cdot S^3)$ for voxel data, plus $O(F)$ for mesh geometry, where $F$ is the number of visible faces [S1, S2, unofficial].
- **Performance:** Real-time generation is achieved via background threads and distance-sorted loading [S2, unofficial]. Chunk size of 8x8x8 is small enough for fast generation but may lead to higher overhead per chunk; 32x32x32 reduces overhead but increases per-chunk generation time [S1, S2, unofficial].
- **Strengths:** Simple implementation, supports complex biomes and verticality (caves/overhangs) via 3D noise, efficient culling of internal faces [S1, S2, unofficial].
- **Limitations:** Dense voxel grid can be memory-intensive for large worlds; noise artifacts may appear at chunk boundaries if not smoothed; thread safety requires careful state management [S1, S2, unofficial].
- **When to Use:** When a fully volumetric, explorable terrain with caves and overhangs is required, and performance constraints allow for dense voxel storage.
- **When Not to Use:** When memory is extremely limited, or when terrain is strictly 2D heightmap-based (use [Procedural Terrain Heightmap Generation](1850-procedural-terrain-heightmap-generation.md)).

## Background
Originating from early voxel games and popularized by titles like *Minecraft*, chunk-based generation allows for infinite, procedurally generated worlds. The specific implementation details cited here are derived from community projects: [S1, unofficial] provides a JavaScript/WebGL implementation with octaved noise and biome logic; [S2, unofficial] provides a Godot 4 C++ extension with threaded loading; [S3, unofficial] provides a Unity implementation with density fields and deformation.

## Sources
[S1] web_voxel_terrain, https://github.com/andrewbloese-00/web_voxel_terrain, fetched 2026-10-27 (unofficial)
[S2] voxel-terrain-plugin, https://github.com/Ry645/voxel-terrain-plugin, fetched 2026-10-27 (unofficial)
[S3] UnityVoxelTerrain, https://github.com/Rabidus4k/UnityVoxelTerrain, fetched 2026-10-28 (unofficial)
