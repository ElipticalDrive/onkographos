# Mesh Generation & Entity Spawning Pipeline

Also called: Chunk Meshing Pipeline

## Overview
This pipeline orchestrates the asynchronous generation of triangle-list meshes from voxel data and their subsequent instantiation as renderable entities in a game engine [S1, S2, unofficial]. It solves the problem of decoupling expensive geometry computation from the main render loop, ensuring frame-rate stability by batching mesh creation and controlling entity spawn rates [S2, unofficial]. Unlike synchronous meshing, this method manages dependencies on neighbor chunks to ensure face consistency, deferring requests until all required context is available [S2, unofficial].

## How it works
The pipeline operates in two distinct phases: **Mesh Generation** (asynchronous geometry construction) and **Entity Spawning** (synchronous entity instantiation). It processes `MeshChunkRequest` events, consulting a cache of chunk states to determine whether to generate new geometry, wait for neighbors, or spawn an entity.

### Inputs and Outputs
*   **Input:** `MeshChunkRequest` containing `ChunkPosition` and `ZoomLevel`.
*   **Output:** Bevy `Entity` instances with attached `Mesh` assets, `Transform`, and `Material`.

### Data Structures

**MeshInfo**
Holds the raw geometry data before being committed to a Bevy `Mesh` asset.
*   `positions`: `Vec<[f32; 3]>`
*   `normals`: `Vec<[f32; 3]>`
*   `colors`: `Vec<[f32; 4]>`
*   `uvs`: `Vec<[f32; 2]>`
*   `indices`: `Vec<u32>`

**MeshStatus**
Tracks the lifecycle state of a chunk's mesh in the `meshes` map (keyed by `(ChunkPosition, ZoomLevel)`).
*   `Meshing`: Geometry is currently being computed.
*   `Empty`: Chunk contains no blocks; no mesh is generated.
*   `Meshed(MeshInfo)`: Geometry is ready for asset creation.

**Quad**
Represents a merged face on a chunk boundary.
*   `voxel`: `[u32; 3]` Base coordinate of the quad.
*   `width`: `u32`
*   `height`: `u32`

### Parameters
*   `MAX_DATA_TASKS`: Maximum concurrent data loading tasks. Value: `9` [S1, unofficial].
*   `MAX_MESH_TASKS`: Maximum concurrent mesh generation tasks. Value: `3` [S1, unofficial].
*   `MAX_SCANS`: Maximum chunks scanned per frame for visibility/distance checks. Value: `26000` [S1, unofficial].
*   `CHUNK_SIZE`: Size of a chunk side in voxels. Value: `32` [S1, S2, unofficial].
*   `MESH_SPAWN_RATE`: Controls the rate at which meshed chunks are spawned as entities to prevent lag spikes. Calculated as:
    $$ \text{MESH\_SPAWN\_RATE} = \left\lfloor 128.0 \times \frac{32.0}{\text{CHUNK\_SIZE\_F32}} \right\rfloor $$
    Default calculation scales inversely with chunk size [S2, unofficial].
*   `CHUNK_REQUEST_RATE`: Rate at which mesh requests are processed.
    $$ \text{CHUNK\_REQUEST\_RATE} = \text{MESH\_SPAWN\_RATE} \times 5 $$
    [S2, unofficial].

### Algorithm Steps

1.  **Request Reception:** Receive a `MeshChunkRequest` with `ChunkPosition` and `ZoomLevel`.
2.  **Cache Check:** Look up the chunk in `world.cache` (or the `meshes` map).
    *   If the chunk data does not exist, issue a `GenerateChunkRequest` (to load voxel data) and schedule a `MeshChunkRerequest` for later [S2, unofficial].
    *   If the chunk is currently `Meshing`, schedule a `MeshChunkRerequest` [S2, unofficial].
3.  **Empty Check:** If the chunk data exists but is empty (no blocks), set `MeshStatus::Empty` and skip further processing [S2, unofficial].
4.  **Neighbor Validation:** If the chunk is generated, collect neighbor chunk faces.
    *   For each of the 26 adjacent directions (Moore neighborhood excluding center), determine the neighbor's status [S1, unofficial].
    *   If a neighbor is `Generated`, use its exposed faces.
    *   If a neighbor is `Empty`, use `EMPTY_CHUNK_FACES` (culling all internal faces) [S2, unofficial].
    *   If any neighbor is missing or `Meshing`, schedule a `MeshChunkRerequest` to avoid incorrect culling or geometry artifacts [S2, unofficial].
5.  **Mesh Generation:** If all neighbors are present, call the mesh generation function (`bevy_mesh_greedy_quads` or `bevy_mesh_visible_block_faces`) passing:
    *   The chunk voxel data.
    *   Neighbor face data.
    *   Block textures/materials.
    *   Visibility checker [S2, unofficial].
6.  **Store Result:** Insert the result into the `meshes` map with status `Meshed(MeshInfo)` or `Empty` [S2, unofficial].
7.  **Spawn Phase:** Iterate through chunks with status `SpawnRequested` (or `Meshed` ready for spawning). Spawn up to `MESH_SPAWN_RATE` entities per frame [S2, unofficial].
    *   For each chunk:
        a.  Create a Bevy `Mesh` asset with `PrimitiveTopology::TriangleList` and `RenderAssetUsages::RENDER_WORLD` [S1, unofficial].
        b.  Insert vertex attributes:
            *   `positions` as `ATTRIBUTE_POSITION` (or standard position attribute).
            *   `normals` as `ATTRIBUTE_NORMAL`.
            *   `colors` as `ATTRIBUTE_COLOR` (or custom attribute like `ATTRIBUTE_VOXEL` if specified by shader, but typically color is standard) [S1, S2, unofficial]. Note: S1 mentions `ATTRIBUTE_VOXEL` with mesh vertices, implying custom attributes may be used for block type data [S1, unofficial].
            *   `uvs` as `ATTRIBUTE_UV_0`.
        c.  Insert indices as `Indices::U32` [S1, unofficial].
        d.  Add the mesh to the asset server and retrieve the `Handle<Mesh>` [S1, unofficial].
        e.  Despawn any existing entity at the world position if present [S1, unofficial].
        f.  Spawn a new entity with:
            *   Transform: `world_pos.as_vec3() * Vec3::splat(32.0)` (assuming unit voxel size) [S1, unofficial].
            *   AABB: `from_min_max(Vec3::ZERO, Vec3::splat(32.0))` [S1, unofficial].
            *   Mesh Handle.
            *   Global Material (opaque or translucent based on block types) [S1, S2, unofficial].
        g.  Insert the new entity into the `chunk_entities` map [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
MAX_DATA_TASKS = 9
MAX_MESH_TASKS = 3
MAX_SCANS = 26000
CHUNK_SIZE = 32
MESH_SPAWN_RATE = int(128.0 * (32.0 / CHUNK_SIZE))
CHUNK_REQUEST_RATE = MESH_SPAWN_RATE * 5

# Data Structures
class MeshInfo:
    positions: List[List[float]] # Vec<[f32; 3]>
    normals: List[List[float]]   # Vec<[f32; 3]>
    colors: List[List[float]]    # Vec<[f32; 4]>
    uvs: List[List[float]]       # Vec<[f32; 2]>
    indices: List[int]           # Vec<u32>

class MeshStatus:
    Meshing
    Empty
    Meshed(info: MeshInfo)

# Global State
meshes: Dict[(ChunkPosition, ZoomLevel), MeshStatus] = {}
chunk_entities: Dict[ChunkPosition, Entity] = {}
world_cache: Dict[ChunkPosition, ChunkData] = {}

def process_mesh_request(request: MeshChunkRequest):
    pos = request.chunk_position
    zoom = request.zoom_level
    key = (pos, zoom)

    # 1. Check Cache
    if key not in meshes:
        if pos not in world_cache:
            # Request data generation
            schedule_generate_chunk(pos)
            schedule_rerequest_mesh(pos, zoom)
            return
        else:
            # Data exists, start meshing
            meshes[key] = MeshStatus.Meshing
            schedule_mesh_task(pos, zoom)
            return
    elif meshes[key] == MeshStatus.Meshing:
        schedule_rerequest_mesh(pos, zoom)
        return
    elif meshes[key] == MeshStatus.Empty:
        return # Skip

    # 2. Validate Neighbors
    neighbor_faces = collect_neighbor_faces(pos)
    if neighbor_faces is None:
        # Missing or generating neighbor
        schedule_rerequest_mesh(pos, zoom)
        return

    # 3. Generate Mesh
    chunk_data = world_cache[pos]
    mesh_info = generate_mesh(chunk_data, neighbor_faces)
    
    meshes[key] = MeshStatus.Meshed(mesh_info)
    schedule_spawn_entity(pos, zoom)

def collect_neighbor_faces(pos: ChunkPosition) -> Optional[List[FaceData]]:
    # Check 26 neighbors
    for direction in ADJACENT_CHUNK_DIRECTIONS:
        if direction == (0,0,0): continue
        neighbor_pos = pos + direction
        neighbor_status = get_mesh_status(neighbor_pos)
        
        if neighbor_status == MeshStatus.Meshing or neighbor_status is None:
            return None # Defer
        elif neighbor_status == MeshStatus.Empty:
            # Use empty faces (cull all)
            return EMPTY_CHUNK_FACES
        else:
            # Use generated faces
            return get_neighbor_faces(neighbor_pos)
    return collected_faces

def generate_mesh(chunk: ChunkData, neighbor_faces: List[FaceData]) -> MeshInfo:
    # Call greedy or visible face algorithm
    # Returns MeshInfo with positions, normals, colors, uvs, indices
    return mesh_chunk_greedy_quads(chunk, neighbor_faces)

def spawn_entity(pos: ChunkPosition, zoom: ZoomLevel):
    key = (pos, zoom)
    status = meshes.get(key)
    
    if not isinstance(status, MeshStatus.Meshed):
        return

    mesh_info = status.info
    
    # Create Bevy Mesh Asset
    bevy_mesh = create_bevy_mesh()
    bevy_mesh.insert_attribute("position", mesh_info.positions)
    bevy_mesh.insert_attribute("normal", mesh_info.normals)
    bevy_mesh.insert_attribute("color", mesh_info.colors)
    bevy_mesh.insert_attribute("uv", mesh_info.uvs)
    bevy_mesh.set_indices(mesh_info.indices)
    
    handle = asset_server.add(bevy_mesh)
    
    # Despawn old entity if exists
    if pos in chunk_entities:
        despawn(chunk_entities[pos])
    
    # Spawn new entity
    transform = Transform.from_translation(pos.as_vec3() * 32.0)
    aabb = AABB.from_min_max(Vec3(0,0,0), Vec3(32,32,32))
    
    # Determine material (opaque/translucent)
    material = get_global_material(mesh_info)
    
    entity = spawn_entity_bundle(
        TransformBundle(transform),
        MeshBundle(handle),
        MaterialBundle(material),
        VisibilityBundle(aabb)
    )
    
    chunk_entities[pos] = entity
    meshes[key] = MeshStatus.Empty # Or remove from map, depending on lifecycle

def main_loop():
    # 1. Scan for chunks to request
    scan_count = 0
    for chunk_pos in get_chunks_in_view():
        if scan_count >= MAX_SCANS: break
        if chunk_pos not in chunk_entities:
            schedule_mesh_request(chunk_pos)
            scan_count += 1
            
    # 2. Process Mesh Tasks (Async)
    while active_mesh_tasks < MAX_MESH_TASKS:
        if mesh_queue not empty:
            task = mesh_queue.pop()
            process_mesh_request(task)
            
    # 3. Spawn Entities (Throttled)
    spawn_count = 0
    for pos, status in meshes.items():
        if spawn_count >= MESH_SPAWN_RATE: break
        if status == MeshStatus.Meshed:
            spawn_entity(pos)
            spawn_count += 1
```

### Edge Cases and Pitfalls
*   **Neighbor Dependencies:** Neighbor chunks must be fully generated before meshing can proceed. If any neighbor is missing or still generating, the mesh request is deferred via `MeshChunkRerequest` to avoid incorrect culling or geometry [S2, unofficial].
*   **Empty Chunks:** Empty chunks (no blocks) must be handled explicitly to avoid spawning empty entities. The status `MeshStatus::Empty` causes the chunk request to be removed from the queue [S2, unofficial].
*   **Precision:** Vertex positions are typically integer-based (voxel coordinates) scaled by chunk size. Ensure `f32` precision is sufficient for large world coordinates; consider using `f64` for world positions if chunks exceed $2^{15}$ units [G].

## Operating characteristics
*   **Time Complexity:** Mesh generation is $O(V)$ where $V$ is the number of voxels in the chunk (typically $32^3 = 32,768$). The greedy algorithm reduces this to $O(F)$ where $F$ is the number of visible faces. Entity spawning is $O(1)$ per chunk, throttled by `MESH_SPAWN_RATE`.
*   **Memory Complexity:** $O(F)$ for mesh storage per chunk. `MeshInfo` stores positions, normals, colors, uvs, and indices. For a dense chunk, this is several megabytes.
*   **Performance:** The pipeline uses asynchronous tasks (`MAX_MESH_TASKS = 3`) to prevent blocking the main thread [S1, unofficial]. The spawn rate is dynamically scaled based on chunk size to maintain consistent frame times [S2, unofficial].
*   **Strengths:** Decouples heavy geometry computation from rendering; handles neighbor consistency automatically; prevents lag spikes via throttled spawning.
*   **Limitations:** High memory usage for large worlds with many visible chunks; asynchronous nature introduces latency between chunk generation and visibility.

## Background
This pipeline is part of the Infinigen project, a sandbox for experimenting with Bevy and efficient chunk rendering in Rust [S2, unofficial]. It implements a procedural generation and rendering pipeline that manages chunk loading, mesh generation (greedy or visible face), and entity spawning, handling asynchronous requests and neighbor dependencies [S2, unofficial]. The codebase is maintained as an unofficial repository [S2, unofficial].

## Sources
[S1] Binary Greedy Voxel Meshing Algorithm, https://github.com/TanTanDev/binary_greedy_mesher_demo, fetched 2026-10-26 (unofficial)
[S2] jameshiew/infinigen, https://github.com/jameshiew/infinigen, fetched 2026-10-27 (unofficial)
