# Chunk-Based Voxel Terrain Generation

Also called: Procedural voxel placement, Voxel Terrain Generation, Voxel Terrain Generation via Chunked Noise Sampling, VoxelChunk generation

## Overview
Chunk-Based Voxel Terrain Generation is a procedural technique that divides a 3D world into fixed-size cubic chunks, populating each with voxel data via multi-octave noise sampling and biome logic, then extracting visible faces for meshing [S1, S2, S4, unofficial]. It differs from raymarching or marching cubes by explicitly generating a dense 3D voxel grid per chunk before mesh extraction, rather than computing surfaces via implicit functions or ray traversal [S2, unofficial].

## How it works
**Inputs:** World seed, chunk dimensions, noise parameters (frequencies, amplitudes), biome thresholds (water level, temperature, moisture), and player position for culling.
**Outputs:** A set of `TerrainChunk` objects containing vertex/index buffers for visible faces, and optionally a dense density field for deformation.

**Data Structures:**
1.  **Chunk Cache Item (`ChunkCacheItem`):**
    *   `enc`: `Uint16Array` of Run-Length Encoded (RLE) voxel data.
    *   `faces`: `Map<VoxelID, number[]>` where `number[]` is packed face data.
    *   `n`: Total face count.
    *   `is_empty`: Boolean flag.
2.  **Packed Face Data:** An array containing `x, y, z` coordinates, `voxel_id`, and `face_direction_index`.
3.  **Vertex Buffers:**
    *   `position_buffer`: `Float32Array` of size `num_vertices * 3`.
    *   `vert_id_buffer`: `Float32Array` of size `num_vertices`.
    *   `packed_buffer`: `Float32Array` of size `num_vertices`.
    *   `indices`: `Uint16Array` of size `num_faces * 6`.
4.  **Density Field (Optional/Variant):** A 3D array of size `(dimensions + 1)^3` [S3, unofficial].

**Parameters:**
*   `CHUNK_SIZE`: 8 (S1) or 32 (S2) voxels per side [S1, S2, unofficial].
*   `VOXEL_SIZE`: 1 meter [S1, unofficial].
*   `CHUNK_LENGTH`: `CHUNK_SIZE^3` (512 for size 8) [S1, unofficial].
*   `max_elevation`: Max height from noise (e.g., 255) [S1, unofficial].
*   `water_level`: Y-coordinate threshold for water (e.g., 40) [S1, unofficial].
*   `_cut_off`: Noise threshold for voxel presence (e.g., -0.75) [S2, unofficial].
*   `_terrain_scope`: Noise scale factor (e.g., 1.0) [S2, unofficial].
*   `smoothIterations`: Blur passes (e.g., 2, range 0-4) [S3, unofficial].
*   `seed`: Procedural noise seed (e.g., 1337) [S3, unofficial].

**Steps:**

1.  **Initialize Noise Functions:**
    *   Create height noise from `seed`.
    *   Create moisture noise from `seed` shuffled by 10.
    *   Create wackiness noise from moisture seed shuffled by 10.
    *   Configure `OctavedNoise2d` parameters:
        *   Height: Frequency 1.2, Amplitude 400.
        *   Moisture: Frequency 1.0, Amplitude 200.
        *   Wackiness: Frequency 1, Amplitude 20 [S1, unofficial].
    *   Define temperature function: $T(z) = \text{clamp}(\text{equator\_temp} - |z - \text{world\_size}/2| \times \text{temp\_decay\_coeff}, 0, 1)$ [S1, unofficial].

2.  **Initialize Loading Threshold:**
    *   Create an `Area3D` with a `SphereShape3D` collision shape.
    *   Set radius: $R = \text{render\_distance} \times \text{CHUNK\_ZONE\_FACTOR}$ (where `CHUNK_ZONE_FACTOR` is 0.25) [S2, unofficial].
    *   Connect `body_exited` signal to unload chunks.
    *   Update global position to player's current position [S2, unofficial].

3.  **Determine Chunk Properties:**
    *   Evaluate wackiness noise at chunk coordinates divided by 100: $w = \text{wackiness}(x/100, z/100)$.
    *   Map $w$ to exponent `chunk_exp`:
        *   $w < 0.2 \rightarrow 16$
        *   $0.2 \le w < 0.4 \rightarrow 8$
        *   $0.4 \le w < 0.6 \rightarrow 2$
        *   $w > 0.8 \rightarrow 0.5$
        *   Else $\rightarrow w$ [S1, unofficial].

4.  **Generate Voxel Data:**
    *   For each voxel $(x, y, z)$ in the chunk:
        *   Calculate world coordinates: $X = x_{chunk} \times \text{CHUNK\_WORLD\_SIZE} + x$, $Z = z_{chunk} \times \text{CHUNK\_WORLD\_SIZE} + z$.
        *   Evaluate height noise: $h = \text{height}(X, Z)$.
        *   Calculate terrain height: $\text{genHeight} = h^{\text{chunk\_exp}} \times \text{max\_elevation}$ [S1, unofficial].
        *   Determine voxel type using `determineGeneratedVoxel(genHeight, wackiness, worldPos)`:
            *   If $Y < \text{water\_level}$:
                *   If $Y \ge \text{genHeight}$: WATER.
                *   If $Y + 2 \ge \text{genHeight}$: SAND.
                *   If $Y + 8 \ge \text{genHeight}$: MUD.
                *   If $Y + 14 \ge \text{genHeight}$: SANDSTONE.
                *   Else: STONE.
            *   If $Y > \text{genHeight}$: AIR.
            *   Else (Land):
                *   Get moisture $m = \text{moisture}(X, Z)$ and temperature $T$.
                *   Desert ($m \le 0.3, T \ge 0.7$): SAND (top 3), SANDSTONE (next 4), STONE.
                *   Tundra ($m \le 0.4, T \le 0.3$): SNOW (top 1), PACKED_DIRT (next 5), STONE.
                *   Jungle/Mushroom (High moisture/wackiness): Specific layers (e.g., MYCELIUM, DIRT, GRASS) [S1, unofficial].
        *   Store voxel type in `Uint8Array` of size `CHUNK_LENGTH`.

5.  **Encode and Check Empty:**
    *   Run-length encode the voxel array.
    *   If encoded length is 2 and the second element is AIR, flag chunk as empty and return [S1, unofficial].

6.  **Identify Visible Faces:**
    *   For each non-AIR/non-WATER voxel:
        *   Check all 6 neighbors (directions: $\pm X, \pm Y, \pm Z$).
        *   If neighbor is OOB (outside chunk bounds $0 \dots \text{CHUNK\_SIZE}-1$) or is AIR/WATER, the face is visible [S1, S3, unofficial].
        *   Pack face data $(x, y, z, \text{voxel\_id}, \text{direction})$ into a Map keyed by voxel ID [S1, unofficial].

7.  **Generate Mesh Geometry:**
    *   For each visible face, create 4 vertices (quad corners).
    *   Store position, vertex ID (0-3), and packed face data in buffers.
    *   Generate 2 triangles (6 indices) per face using vertex indices [S1, unofficial].

8.  **Post-Processing (Variant S3):**
    *   Call `SmoothField()` to apply a blur filter to the density field for `smoothIterations` passes [S3, unofficial].
    *   Call `CreateChunks()` to split the volume into `TerrainChunk` objects [S3, unofficial].

9.  **Asynchronous Loading (Variant S2):**
    *   Start a background thread bound to `_thread_generate_terrain`.
    *   Sort chunks by distance to player chunk.
    *   Iterate sorted chunks:
        *   Check if loading position changed or generation stopped; abort if true [S2, unofficial].
        *   Compute chunk position and call `_generate_chunk`.
        *   Instantiate `VoxelChunk`, append to `_chunks`.
        *   Loop $x,y,z$ from 0 to `_chunk_size`, compute voxel position, sample noise: if $\text{noise}(\text{voxel\_pos} \times \text{terrain\_scope}) < \text{cut\_off}$, set voxel color [S2, unofficial].
        *   Call `chunk->generate_mesh()` and defer adding to scene tree [S2, unofficial].

**Pitfalls:**
*   **Neighbor Boundary Check:** Neighbors outside chunk bounds are considered visible (exposed to void/other chunk) [S1, unofficial].
*   **Water Visibility:** Faces adjacent to WATER are rendered, treating water as transparent/empty for culling [S1, unofficial].
*   **Thread Safety:** `last_loading_position` must not be replaced by `loading_threshold.global_position` directly as it is read by worker threads; this causes runtime errors [S2, unofficial].
*   **Stale Chunk Processing:** If the player moves during generation, the thread must return immediately to avoid generating mismatched chunks [S2, unofficial].

## Subtopics

### Chunk-based Voxel World Generation
Generates terrain by initializing a 3D array `self.voxels` of size `[WORLD_VOL, CHUNK_VOL]` of type `uint8`. For each chunk position $(x, y, z)$, it calculates `chunk_index = x + WORLD_W * z + WORLD_AREA * y`. It iterates $x$ and $z$ from 0 to `CHUNK_SIZE`, calculates `world_height` using `get_height`, and then iterates $y$ from 0 to `local_height`. Inside the Y loop, it calls `set_voxel_id` to determine block type based on height, noise, and random placement (trees) [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(C \cdot V)$ per chunk, where $C$ is the number of chunks and $V$ is voxels per chunk ($CHUNK\_SIZE^3$). Mesh generation is proportional to surface area.
*   **Memory Complexity:** $O(C \cdot V)$ for dense voxel storage, or $O(F)$ for face-based storage where $F$ is visible faces.
*   **Performance:** Efficient for static terrain due to RLE compression and face culling. Real-time loading requires background threading to avoid stutter [S2, unofficial].
*   **Strengths:** Deterministic, easy to implement, supports complex biome logic via noise thresholds.
*   **Limitations:** Dense grid storage is memory-intensive for large worlds without chunking; noise evaluation can be CPU-bound [S2, S4, unofficial].
*   **When to use:** Static or semi-static voxel worlds where deterministic generation is required.
*   **When not to use:** Highly dynamic, player-deformable worlds requiring real-time density field updates (use Density Field methods instead) [S3, unofficial].

## Background
Originating from procedural generation techniques in voxel-based games, this method was implemented in various forms in open-source projects such as `web_voxel_terrain` (JavaScript/WebGL) [S1, unofficial], `voxel-terrain-plugin` (Godot 4 C++) [S2, unofficial], `UnityVoxelTerrain` (Unity C#) [S3, unofficial], and `Minecraft`-style implementations (Python) [S4, unofficial]. It builds on [Chunk Generation with Base Terrain](1110-chunk-generation-with-base-terrain.md).

## Sources
[S1] andrewbloese-00/web_voxel_terrain, https://github.com/andrewbloese-00/web_voxel_terrain, fetched 2026-10-27 (unofficial)
[S2] Ry645/voxel-terrain-plugin, https://github.com/Ry645/voxel-terrain-plugin, fetched 2026-10-27 (unofficial)
[S3] Rabidus4k/UnityVoxelTerrain, https://github.com/Rabidus4k/UnityVoxelTerrain, fetched 2026-10-28 (unofficial)
[S4] StanislavPetrovV/Minecraft, https://github.com/StanislavPetrovV/Minecraft, fetched 2026-10-26 (unofficial)
