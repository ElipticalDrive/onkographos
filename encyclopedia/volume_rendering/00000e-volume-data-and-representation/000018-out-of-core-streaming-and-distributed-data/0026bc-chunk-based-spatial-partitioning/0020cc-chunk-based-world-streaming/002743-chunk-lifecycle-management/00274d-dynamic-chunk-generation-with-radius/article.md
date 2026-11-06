# Dynamic Chunk Generation with Radius

Also called: Chunk Lifecycle Management with Radius, Chunk Loading/Unloading (Render Distance Culling), Chunk Streaming & Loading, Chunk culling, Distance-based chunk management

## Overview
Dynamic Chunk Generation with Radius manages chunk instantiation and removal based on a spherical render distance around the player, using set difference operations to determine which chunks to load or unload [S1, unofficial]. It builds on [Chunk Lifecycle Management](3987-chunk-lifecycle-management.md) by replacing static or scanner-based triggers with a continuous distance check against the player's current position [G]. Unlike [Chunk Loading/Unloading Scanner](4404-chunk-loading-unloading-scanner.md), which scans discrete regions, this method evaluates every chunk within a calculated bounding box relative to the player to maintain a consistent spherical visibility volume [G].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Player position ($P$), render radius ($R$), chunk size ($S$), and the current set of loaded chunks.
*   **Outputs:** Updated set of loaded chunks (instantiated or destroyed) [S1, S3, unofficial].

**Data Structures:**
*   **Loaded Chunks Dictionary:** A hash map (dictionary) mapping chunk coordinates to chunk state or object references [S2, unofficial]. Keys are integer tuples $(x, y, z)$ representing the chunk grid index [G]. This structure provides fast access for existence checks [S3, unofficial].
*   **Chunk States:** Chunks are assigned one of three labels during the update cycle: **KEEP** (exists and within radius), **DRAW** (does not exist but within radius), or **DONE** (exists but outside radius) [S3, unofficial].

**Algorithm Steps:**
1.  **Calculate Bounding Sphere Diameter:** Determine the diameter of the render sphere in chunk units.
    $$ \text{render\_diameter} = \left\lfloor \frac{\text{radius} \times 2}{\text{chunk\_size}} \right\rfloor $$
    [S1, unofficial].
2.  **Determine Grid Range:** Calculate the minimum and maximum chunk indices ($x, y, z$) that could possibly intersect the render sphere.
    $$ \text{chunk\_range\_start} = \left\lfloor \frac{\text{render\_diameter}}{2} \right\rfloor \times -1 $$
    $$ \text{chunk\_range\_end} = \text{render\_diameter} + \text{chunk\_range\_start} $$
    [S1, unofficial].
3.  **Identify Target Chunks:** Iterate through all integer coordinates $(x, y, z)$ within the range $[\text{chunk\_range\_start}, \text{chunk\_range\_end}]$. Compute chunk position: $\text{chunk\_possibly\_in\_radius} = (x, y, z) + \text{get\_chunk\_from\_point}(P, \text{chunk\_size})$. Compute world location: $\text{chunk\_location} = \text{chunk\_possibly\_in\_radius} \times \text{chunk\_size}$. If $\text{chunk\_location.distance\_to}(P) < \text{radius}$, add the chunk key $(x, y, z)$ to a temporary set `surrounding_chunks` [S1, unofficial].
4.  **Compute Set Differences:** Compare `surrounding_chunks` with the keys of the currently loaded chunks dictionary (`loaded_chunks`) to find load/remove sets [S1, unofficial].
5.  **Execute Lifecycle Actions:** If a chunk exists in `loaded_chunks` and is in `surrounding_chunks`, keep it for the next frame. If it does not exist in `loaded_chunks` but is in `surrounding_chunks`, create it. If it exists in `loaded_chunks` but not in `surrounding_chunks`, mark it as DONE and destroy it [S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def get_chunk_from_point(point, chunk_size):
    return (int(point[0] // chunk_size), int(point[1] // chunk_size), int(point[2] // chunk_size))

def distance(p1, p2):
    return ((p1[0]-p2[0])**2 + (p1[1]-p2[1])**2 + (p1[2]-p2[2])**2)**0.5

def update_chunks(player_pos, render_radius, chunk_size, loaded_chunks_dict):
    # 1. Calculate bounds
    render_diameter = int(render_radius * 2) // chunk_size
    range_start = (render_diameter // 2) * -1
    range_end = render_diameter + range_start
    
    surrounding_chunks = {}
    
    # 2. Iterate and filter by distance
    for x in range(range_start, range_end + 1):
        for y in range(range_start, range_end + 1):
            for z in range(range_start, range_end + 1):
                # Compute chunk world position
                chunk_possibly_in_radius = (x, y, z) + get_chunk_from_point(player_pos, chunk_size)
                chunk_location = tuple(v * chunk_size for v in chunk_possibly_in_radius)
                
                # Distance check
                if distance(chunk_location, player_pos) < render_radius:
                    surrounding_chunks[(x, y, z)] = True

    # 3. Determine load/unload sets
    loaded_keys = set(loaded_chunks_dict.keys())
    target_keys = set(surrounding_chunks.keys())
    
    chunks_to_load = target_keys - loaded_keys
    chunks_to_unload = loaded_keys - target_keys
    
    # 4. Execute actions
    for key in chunks_to_load:
        # Instantiate chunk and add to loaded_chunks_dict [G]
        pass
        
    for key in chunks_to_unload:
        # Mark chunk as DONE and remove from loaded_chunks_dict [G]
        del loaded_chunks_dict[key]
        
    # Keys in both sets are implicitly kept; no action needed
```

**Parameters:**
*   **Render Radius ($R$):** The distance from the player within which chunks are generated. Not given in the sources.
*   **Chunk Size ($S$):** The physical dimension of a chunk. Not given in the sources.
*   **m_RenderDistance:** Radius in chunks for loading/unloading [S2, unofficial].
*   **m_MaxRemeshPerFrame:** Maximum remesh jobs pushed per frame [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Stale Data:** Dictionary keys must be cleared before population to avoid retaining stale data from previous frames [S1, unofficial].
*   **Performance:** A naive single-threaded implementation checking every chunk in the radius is extremely slow and inefficient for open worlds [S3, unofficial].
*   **Generation Latency:** On older CPUs with fewer cores, the player might run out of the generated world and fall; a solution is to slow down the player to give the CPU more time to generate new chunks [S3, unofficial].

## Subtopics

### Multithreaded Recursive Flood Fill Generation
A multithreaded generation technique that uses a recursive flood fill algorithm to build chunks simultaneously around the player, significantly reducing loading and deletion time compared to the single-threaded radius check [S3, unofficial]. Instead of iterating a bounding box, it initiates a search in 4 directions from the player's position, recursively calling the generation algorithm until the radius limit is reached [S3, unofficial]. It employs concurrent dictionaries and queues to allow multiple threads to access chunks without deadlocks [S3, unofficial]. On older CPUs with fewer cores, the player might outrun the generation; a mitigation is to slow down the player to allow the CPU time to generate chunks [S3, unofficial]. Custom concurrency primitives were required in Unity 5.4 (.NET 4.6) because built-in `ConcurrentQueue` and `ConcurrentDictionary` were unavailable, though newer .NET versions provide these [S3, unofficial].

### World and Chunk Caching
A management approach that maintains the world state, including the chunk generator and a cache for chunk generation status and data [S4, unofficial]. The `World` object contains an `Arc<dyn WorldGen>` trait object for chunk generation and an `AHashMap` mapping `(ChunkPosition, ZoomLevel)` to `ChunkStatus` [S4, unofficial]. The `ChunkStatus` enum tracks whether a chunk is `Generating`, `Generated` (holding an `Arc<ChunkInfo>`), or `Empty` [S4, unofficial]. `ChunkInfo` contains opaque geometry (`Box<Array3Chunk>`), translucent geometry (`Vec<Array3Chunk>`), and face data (`StaticCopyMap<Direction, ChunkFace>`) [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Simple to implement; guarantees a spherical visibility volume; efficient lookups with hash maps [S1, S3, unofficial].
*   **Limitations:** Single-threaded version is inefficient for large radii; requires careful handling of generation latency to prevent players from outpacing world generation; bounding box iteration includes chunks outside the radius that are quickly discarded, wasting CPU cycles if the radius is large [S3, unofficial].
*   **When to use:** Suitable for moderate render distances and when generation is fast or pre-cached. Requires multithreading or asynchronous generation for large open worlds to avoid stuttering [S3, unofficial].

## Background
The method originated in voxel-based procedural terrain generation systems for open-world games where static generation was insufficient [S3, unofficial]. The initial implementation used a simple radius value updated as the player moved, labeling chunks as KEEP, DRAW, or DONE [S3, unofficial]. This approach was found too slow for open worlds, leading to the development of multithreaded variants [S3, unofficial]. The concept is implemented in various voxel engines, including Godot-based plugins and Rust-based generators like Infinigen [S1, S4, unofficial].

## Sources
[S1] Ry645/voxel-terrain-plugin, https://github.com/Ry645/voxel-terrain-plugin, fetched 2026-10-27 (unofficial)
[S2] adrienlabayle/VoxelEngine, https://github.com/adrienlabayle/VoxelEngine, fetched 2026-10-27 (unofficial)
[S3] Voxel-Based Procedural Terrain Generation | PDF | Voxel | Rendering ..., https://www.scribd.com/document/840779393/Procedural-Terrain-Generation-Using-Voxels, fetched 2026-10-29 (unofficial)
[S4] jameshiew/infinigen, https://github.com/jameshiew/infinigen, fetched 2026-10-27 (unofficial)
