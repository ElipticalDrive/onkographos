# Chunk System (Dynamic Loading)
Also called: Dynamic Chunk Loading, On-Demand Chunk Generation
## Overview
The Chunk System (Dynamic Loading) manages procedural world generation by loading and unloading fixed-size chunks on-demand based on camera position, enabling infinite or very large worlds without pre-generating geometry [S1, unofficial]. It distinguishes itself from static LOD approaches by relying on deterministic procedural regeneration rather than pre-cached static meshes, ensuring memory usage remains bounded by the viewer's vicinity [S1, unofficial].
## How it works
**Inputs and Outputs:**
- Input: Camera position, chunk size, load/unload radii, world seed.
- Output: A set of active chunk meshes rendered in the scene.

**Data Structures:**
- **Chunk:** A fixed-size volume of world units. Contains voxel density data and generated mesh geometry.
- **Chunk Registry:** A map or hash table mapping chunk coordinates $(C_x, C_y, C_z)$ to active Chunk objects.
- **World Seed:** A deterministic integer used to generate procedural content.

**Parameters:**
- Chunk size: $50 \times 50 \times 50$ world units [S1, unofficial].
- Load radius: 3 chunks (150 units) [S1, unofficial].
- Unload radius: 5 chunks (250 units) [S1, unofficial].

**Steps:**
1. **Initialization:** Load voxel registry definitions once at startup to prevent redundant disk access [S2, unofficial]. Initialize the chunk registry as empty.
2. **Update Loop:** Each frame, call `chunk_manager.Update(renderer.camera_position)` to synchronize loaded chunks with the camera [S1, unofficial].
3. **Coordinate Calculation:** Determine the chunk coordinates $(C_x, C_y, C_z)$ containing the camera. Calculate the bounding box of chunks to load based on the load radius $R_{load}$:
   $$ C_{min} = \lfloor \frac{P_{camera}}{S_{chunk}} \rfloor - R_{load} $$
   $$ C_{max} = \lfloor \frac{P_{camera}}{S_{chunk}} \rfloor + R_{load} $$
   where $S_{chunk}$ is the chunk size [S1, unofficial].
4. **Load Chunks:** For each chunk coordinate in the bounding box:
   - If the chunk is not in the registry, generate it procedurally using the world seed and current coordinates.
   - Add the chunk to the registry.
5. **Unload Chunks:** For each chunk in the registry:
   - Calculate the distance from the camera to the chunk center.
   - If the distance exceeds the unload radius $R_{unload}$, remove the chunk from the registry and free its memory [S1, unofficial].
6. **Generation (Procedural):** For a new chunk at $(C_x, C_y, C_z)$:
   - Generate voxel data deterministically. The same chunk coordinates always produce the same islands [S1, unofficial].
   - Apply the **Halo Trick**: Generate voxels with a $+1$ block border ($+2 \times +2 \times +2$ padding) to ensure geometry continuity across boundaries [S2, unofficial].
   - Convert voxel data to mesh geometry. For each voxel at $(i, j, k)$, compute the voxel center:
     $$ V_{center} = (C_x \cdot S_{chunk} + i + 0.5, C_y \cdot S_{chunk} + j + 0.5, C_z \cdot S_{chunk} + k + 0.5) $$
     Check if the voxel center is close to a triangle plane (e.g., for floating islands) or if density is below an iso-value for terrain. If a face is exposed, add vertices and triangles to the mesh [S1, unofficial].
   - Store the mesh in the chunk object.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class ChunkManager:
    def __init__(self, chunk_size=50, load_radius=3, unload_radius=5, seed=0):
        self.chunk_size = chunk_size
        self.load_radius = load_radius
        self.unload_radius = unload_radius
        self.seed = seed
        self.active_chunks = {}  # Map: (cx, cy, cz) -> Chunk

    def update(self, camera_pos):
        # Calculate current chunk coordinates
        cx = floor(camera_pos.x / self.chunk_size)
        cy = floor(camera_pos.y / self.chunk_size)
        cz = floor(camera_pos.z / self.chunk_size)

        # Determine load bounds
        min_cx = cx - self.load_radius
        max_cx = cx + self.load_radius
        min_cy = cy - self.load_radius
        max_cy = cy + self.load_radius
        min_cz = cz - self.load_radius
        max_cz = cz + self.load_radius

        # Load new chunks
        for x in range(min_cx, max_cx + 1):
            for y in range(min_cy, max_cy + 1):
                for z in range(min_cz, max_cz + 1):
                    key = (x, y, z)
                    if key not in self.active_chunks:
                        chunk = self.generate_chunk(x, y, z)
                        self.active_chunks[key] = chunk

        # Unload distant chunks
        keys_to_remove = []
        for key, chunk in self.active_chunks.items():
            chunk_center = self.get_chunk_center(key)
            dist = distance(camera_pos, chunk_center)
            if dist > self.unload_radius * self.chunk_size:
                keys_to_remove.append(key)
        
        for key in keys_to_remove:
            del self.active_chunks[key]

    def generate_chunk(self, cx, cy, cz):
        # Deterministic generation based on seed and coordinates
        # Apply Halo Trick: generate with +1 border for continuity
        # Convert voxels to mesh
        # Return Chunk object with mesh data
        pass

    def get_chunk_center(self, key):
        cx, cy, cz = key
        return (
            (cx + 0.5) * self.chunk_size,
            (cy + 0.5) * self.chunk_size,
            (cz + 0.5) * self.chunk_size
        )
```
## Subtopics

### [Voxel LOD System](16413-voxel-lod-system.md)
The Voxel LOD System manages the rendering of massive procedural voxel worlds (up to $2^{31}-1$ units per axis) by dynamically adjusting chunk resolution based on viewer proximity, ensuring that only necessary data is queried from world generators. It distinguishes itself from static LOD approaches by combining on-the-fly procedural generation with a transition mesh system to eliminate visual seams between chunks of differing resolutions.

### Procedural Bypass Logic
Pure procedural chunks are regenerated from seed rather than saved to disk to minimize I/O overhead in the demo [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of chunks within the load radius. Generation is deterministic and fast for procedural content.
- **Memory Complexity:** $O(N)$ where $N$ is the number of active chunks. Memory is bounded by the load radius.
- **Strengths:** Enables infinite worlds; low memory footprint; no pre-generation required.
- **Limitations:** Procedural generation can cause stutter if not optimized; requires careful handling of boundaries (Halo Trick) to avoid seams.
- **When to use:** Infinite or very large procedural worlds.
- **When not to use:** Static, hand-crafted worlds with complex geometry where pre-baking is feasible.

## Background
Originating from procedural voxel game development, this system allows for the creation of vast, explorable environments without the storage overhead of pre-generated data [S1, unofficial]. It is commonly used in sandbox games and voxel editors.

## Sources
[S1] lijiemark/floating-islands-generator, https://github.com/lijiemark/floating-islands-generator, fetched 2026-10-27 (unofficial)
[S2] herbal1st/pyvorengi-sdk-demo, https://github.com/herbal1st/pyvorengi-sdk-demo, fetched 2026-10-27 (unofficial)
