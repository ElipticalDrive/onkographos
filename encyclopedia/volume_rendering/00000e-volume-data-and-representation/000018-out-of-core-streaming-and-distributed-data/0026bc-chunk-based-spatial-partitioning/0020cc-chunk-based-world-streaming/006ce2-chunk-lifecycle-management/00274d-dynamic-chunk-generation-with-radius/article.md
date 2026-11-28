# Dynamic Chunk Generation with Radius

Also called: Chunk Lifecycle Management with Radius, Chunk Loading/Unloading (Render Distance Culling), Chunk Streaming & Loading, Chunk culling, Distance-based chunk management

## Overview
Dynamic Chunk Generation with Radius manages the instantiation and removal of voxel chunks based on a spherical render distance around the player or camera [S1, unofficial][S2, unofficial][S3, unofficial]. It solves the problem of maintaining a contiguous, procedurally generated world in memory by loading new chunks as the player approaches and unloading distant ones to free resources [S1, unofficial][S2, unofficial][S3, unofficial]. Unlike [Chunk Loading/Unloading Scanner](4404-chunk-loading-unloading-scanner.md), which typically scans a fixed grid or uses a queue-based approach, this method explicitly calculates the set difference between a spherical volume of interest and the currently loaded chunks to determine load and unload actions [S1, unofficial]. It differs from [Budget Plans](5497-budget-plans.md) by focusing on spatial radius and lifecycle states (`KEEP`, `DRAW`, `DONE`) for creation and deletion rather than computational cost limits [S1, unofficial][S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Player or camera position ($P$), render radius ($R$), chunk size ($C_{size}$), and a map of currently loaded chunks ($M_{loaded}$) [S1, unofficial][S2, unofficial][S3, unofficial].
*   **Outputs:** Sets of chunk coordinates to load ($S_{load}$) and chunks to unload ($S_{unload}$) [S1, unofficial].

**Data Structures:**
*   **Chunk Map ($M_{loaded}$):** A dictionary or hash map where keys are chunk coordinates (e.g., $(x, y, z)$ or a flattened integer) and values are chunk objects or state containers [S1, unofficial][S3, unofficial].
*   **Chunk State Labels:** Each chunk in the map is associated with a lifecycle state [S3, unofficial]:
    *   `KEEP`: Chunk is within radius and must remain loaded for the next frame [S3, unofficial].
    *   `DRAW`: Chunk is within radius and ready for rendering [S3, unofficial].
    *   `DONE`: Chunk is outside radius and marked for destruction [S3, unofficial].

**Steps:**
1.  **Calculate Bounding Grid Range:**
    Determine the grid range of chunk coordinates that could potentially fall within the render radius [S1, unofficial].
    $$ R_{diam} = \lfloor \frac{R \times 2}{C_{size}} \rfloor $$
    $$ Start = \lfloor \frac{R_{diam}}{2} \rfloor \times -1 $$
    $$ End = R_{diam} + Start $$
    [S1, unofficial]

2.  **Iterate Potential Chunks:**
    Iterate through all integer coordinates $(x, y, z)$ within the range $[Start, End]$ [S1, unofficial][S2, unofficial].
    *   For each $(x, y, z)$, compute the chunk's world position $L$ [S1, unofficial].
        $$ L = (Vector3(x, y, z) + \text{player\_chunk\_origin}) \times C_{size} $$
        [S1, unofficial]
    *   Calculate the distance from the player position $P$ to $L$: $d = \| L - P \|$ [S1, unofficial].
    *   If $d < R$, add the chunk coordinate to a temporary set $S_{surrounding}$ [S1, unofficial].

    *Alternative 2D Terrain Approach:*
    Calculate chunk coordinates from camera position:
    $$ ChunkX = \lfloor \frac{CameraX}{C_{size}} \rfloor, \quad ChunkZ = \lfloor \frac{CameraZ}{C_{size}} \rfloor $$
    [S2, unofficial]
    Iterate radius $[-R_{chunks}, R_{chunks}]$, skipping corners where $dx^2 + dz^2 > R_{chunks}^2$ [S2, unofficial].

3.  **Determine Load/Unload Sets:**
    Compare $S_{surrounding}$ with $M_{loaded}$ [S1, unofficial].
    *   **Unload:** If a chunk key exists in $M_{loaded}$ but not in $S_{surrounding}$, add it to $S_{unload}$ [S1, unofficial][S3, unofficial].
    *   **Load:** If a chunk key exists in $S_{surrounding}$ but not in $M_{loaded}$, add it to $S_{load}$ [S1, unofficial][S3, unofficial].

4.  **Execute Lifecycle Actions:**
    *   **Unload:** For each chunk in $S_{unload}$, mark it as `DONE` and destroy or remove it from $M_{loaded}$ [S3, unofficial].
    *   **Load:** For each chunk in $S_{load}$, instantiate the chunk, add it to $M_{loaded}$, and set a `NeedGeneration` flag if procedural generation is required [S2, unofficial][S3, unofficial].
    *   **Keep:** Chunks present in both $S_{surrounding}$ and $M_{loaded}$ retain their current lifecycle state (`KEEP` or `DRAW`) for the next frame [S3, unofficial].

5.  **State Update:**
    Clear the temporary dictionary or set used for $S_{surrounding}$ before the next frame to avoid stale data [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def update_chunk_lifecycle(player_pos, render_radius, chunk_size, loaded_chunks_map):
    # 1. Calculate bounding grid range in chunk coordinates
    render_diameter = int(render_radius * 2 / chunk_size)
    chunk_range_start = -(render_diameter // 2)
    chunk_range_end = render_diameter + chunk_range_start

    # Precompute the chunk coordinate of the player's position
    player_chunk_origin = [int(p // chunk_size) * chunk_size for p in player_pos]

    surrounding_chunks = {}

    # 2. Iterate potential chunks within the bounding box
    for x in range(chunk_range_start, chunk_range_end + 1):
        for y in range(chunk_range_start, chunk_range_end + 1):
            for z in range(chunk_range_start, chunk_range_end + 1):
                # Compute world position of the chunk at offset (x,y,z) from player's chunk
                chunk_world_pos = [player_chunk_origin[i] + x * chunk_size for i in range(3)]

                # Check if chunk is within render radius
                dist_sq = sum((chunk_world_pos[i] - player_pos[i]) ** 2 for i in range(3))
                if dist_sq < render_radius ** 2:
                    surrounding_chunks[tuple(chunk_world_pos)] = True

    # 3. Determine load/unload sets by set difference
    load_set = [pos for pos in surrounding_chunks if pos not in loaded_chunks_map]
    unload_set = [pos for pos in loaded_chunks_map if pos not in surrounding_chunks]

    # 4. Execute lifecycle actions
    for chunk_pos in unload_set:
        chunk = loaded_chunks_map[chunk_pos]
        chunk.state = "DONE"
        chunk.destroy()
        del loaded_chunks_map[chunk_pos]

    for chunk_pos in load_set:
        # [G] Chunk instantiation is implementation-specific; here we assume a Chunk class exists
        new_chunk = Chunk(chunk_pos)
        new_chunk.state = "DRAW"
        new_chunk.need_generation = True
        loaded_chunks_map[chunk_pos] = new_chunk

    # 5. Clear temporary data for next frame
    surrounding_chunks.clear()

class Chunk:
    def __init__(self, pos):
        self.pos = pos
        self.state = "DRAW"
        self.need_generation = True
    def destroy(self):
        pass
```

**Parameters:**
*   `render_radius` ($R$): The spherical distance from the player within which chunks are loaded [S1, unofficial][S2, unofficial][S3, unofficial].
*   `chunk_size` ($C_{size}$): The physical size of a single chunk [S1, unofficial][S2, unofficial].
*   `m_RenderDistance`: Radius in chunks for loading and unloading [S2, unofficial].
*   `m_MaxRemeshPerFrame`: Maximum number of remesh jobs pushed per frame to limit generation load [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Stale Data:** Dictionary keys must be cleared before population to avoid stale data from previous frames [S1, unofficial].
*   **Initial Performance:** The initial single-threaded dictionary approach was extremely slow and inefficient for an open world, requiring multithreading [S3, unofficial].
*   **CPU Bottleneck:** On older CPUs with fewer cores or threads, the player might run out of the generated world and fall if generation cannot keep up with movement [S3, unofficial]. A solution is to slow down the player to give the CPU more time to generate new chunks [S3, unofficial].

## Subtopics

### [World and Chunk Caching](4901-world-and-chunk-caching.md)
World and Chunk Caching manages the state of a procedurally generated voxel world by maintaining a cache of chunk generation statuses and their associated data. It solves the problem of coordinating asynchronous chunk generation with rendering and network synchronization, ensuring that clients or renderers only access fully generated or currently generating chunks. Unlike its parent method, Dynamic Chunk Generation with Radius, which focuses on the spatial radius and lifecycle states (`KEEP`, `DRAW`, `DONE`) for creation and deletion, this method specifically handles the data structure mapping positions to generation states and the serialization of chunk data for snapshots.

### Multithreaded Recursive Flood Fill Generation
A multithreaded generation technique using a recursive flood fill algorithm to build chunks simultaneously around the player, reducing loading and deletion time [S3, unofficial]. It presents a top-down grid of the terrain with the player in the middle and initiates a search in four directions, recursively calling the algorithm until the radius limit is reached [S3, unofficial]. It uses a concurrent dictionary and a concurrent queue to allow multiple threads to access chunks simultaneously without deadlocks [S3, unofficial]. On older CPUs, the player might run out of the generated world, and a solution is to slow down the player [S3, unofficial]. Custom concurrency primitives were needed in Unity 5.4 (.NET 4.6), whereas newer versions (.NET 4.7.2) have built-in `ConcurrentQueue` and `ConcurrentDictionary` [S3, unofficial].

### Circular Chunk Loading
Identifies chunks within a circular view radius to load or activate, avoiding square boundary artifacts [S4, unofficial]. It calculates current chunk coordinates using ceiling division of the player position relative to chunk size [S4, unofficial]. It loops through a square bounding box around the current chunk and skips iterations where the squared distance exceeds the squared view radius [S4, unofficial]. If a chunk is not in the dictionary, it is added to a load list, and if it is in the dictionary but inactive, it is activated [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(R^3)$ for 3D or $O(R^2)$ for 2D terrain, where $R$ is the render radius in chunks, due to iterating the bounding box of the sphere [S1, unofficial][S2, unofficial].
*   **Memory Complexity:** Proportional to the number of chunks within the render radius [S1, unofficial].
*   **Strengths:** Simple to implement; ensures the player is always surrounded by loaded chunks; efficient for static or slowly changing worlds [S3, unofficial].
*   **Limitations:** Single-threaded implementation is inefficient for large open worlds [S3, unofficial]; requires careful handling of concurrency primitives to avoid deadlocks [S3, unofficial]; can cause stuttering if generation is not asynchronous [S3, unofficial].
*   **When to Use:** Suitable for games with moderate render distances and where chunk generation is fast or asynchronous [S3, unofficial].
*   **When Not to Use:** Unsuitable for very large open worlds with high render distances without multithreading or advanced caching [S3, unofficial].

## Background
The method originated from the need to create "never-ending" worlds in voxel-based games, moving beyond static generation [S3, unofficial]. Early implementations used a simple radius value updated as the player moved, using a dictionary to manage chunk states (`KEEP`, `DRAW`, `DONE`) [S3, unofficial]. This initial single-threaded approach was found to be extremely slow for open worlds, leading to the adoption of multithreaded recursive flood fill generation [S3, unofficial]. The technique is widely used in voxel engines and procedural terrain generation systems, such as those found in Godot voxel plugins and custom C++ voxel engines [S1, unofficial][S2, unofficial].

## Sources
[S1] Ry645/voxel-terrain-plugin, https://github.com/Ry645/voxel-terrain-plugin, fetched 2026-10-27 (unofficial)
[S2] adrienlabayle/VoxelEngine, https://github.com/adrienlabayle/VoxelEngine, fetched 2026-10-27 (unofficial)
[S3] Voxel-Based Procedural Terrain Generation | PDF | Voxel | Rendering ..., https://www.scribd.com/document/840779393/Procedural-Terrain-Generation-Using-Voxels, fetched 2026-10-29 (unofficial)
[S4] Javier-Garzo/Marching-cubes-on-Unity-3D, https://github.com/Javier-Garzo/Marching-cubes-on-Unity-3D, fetched 2026-10-26 (unofficial)
