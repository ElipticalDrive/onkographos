# Chunk System (Dynamic Loading)
Also called: Dynamic Chunk Loading, On-Demand Chunk Generation
## Overview
The Chunk System manages procedural world generation by loading and unloading fixed-size chunks on-demand based on camera position, enabling infinite or very large worlds without pre-generating geometry [S1, unofficial]. It differs from static chunk-based generation by actively managing the lifecycle of chunk data in memory relative to the viewer, ensuring that only relevant geometry is computed and rendered [S1, unofficial].
## How it works
**Inputs and Outputs:**
- Input: Camera position, chunk size, load radius, unload radius, procedural generation seed.
- Output: A set of active chunks containing voxel data and generated geometry (meshes) for the rendering pipeline.

**Data Structures:**
- **Chunk:** A fixed-size volume of world units.
    - Layout: 3D array of voxels.
    - Size: $50 \times 50 \times 50$ world units [S1, unofficial].
    - Halo: Chunks include a $+1$ block border on all sides ($+2$ in each dimension) to ensure geometry continuity across boundaries [S2, unofficial].
- **Chunk Registry:** A map or hash table storing active chunks keyed by their integer grid coordinates $(x, y, z)$ [S2, unofficial].
- **Voxel Storage:** Voxels are stored in $(Z, Y, X)$ order to align with NumPy C-order contiguity and optimize flat serialization [S2, unofficial].

**Parameters:**
- Chunk Size: $50$ units [S1, unofficial].
- Load Radius: $3$ chunks ($150$ units) [S1, unofficial].
- Unload Radius: $5$ chunks ($250$ units) [S1, unofficial].

**Algorithm Steps:**
1. **Initialization:** Load voxel registry definitions once at startup to prevent redundant disk access [S2, unofficial].
2. **Update Loop:** Each frame, call `chunk_manager.Update(renderer.camera_position)` [S1, unofficial].
3. **Coordinate Calculation:** Determine the chunk coordinates $(cx, cy, cz)$ corresponding to the camera's current position.
4. **Load Check:** For each chunk within the load radius ($3$ chunks) of the camera's chunk:
    - If the chunk is not in the registry, generate it procedurally.
    - Generation is deterministic: the same chunk coordinates always produce the same result [S1, unofficial].
    - Store the chunk in the registry.
5. **Unload Check:** For each chunk in the registry:
    - If the distance from the camera to the chunk exceeds the unload radius ($5$ chunks), remove it from the registry and free its memory [S1, unofficial].
6. **Geometry Generation:** For each active chunk, generate mesh geometry from the voxel data.
    - Use Marching Cubes or similar algorithm.
    - Handle boundary faces: If a voxel at the edge of a chunk has a neighbor outside the chunk (or in an unloaded chunk), the face is not generated locally but relies on the neighbor chunk's geometry [S1, unofficial]. The halo border ensures seamless connection [S2, unofficial].
    - Example face generation logic (simplified):
      ```python
      def generate_chunk_mesh(chunk_data, iso_value, voxel_size):
          vertices = []
          triangles = []
          # Iterate over voxels, excluding the halo border for internal processing
          # but including it for neighbor checks
          for i in range(1, chunk_size - 1):
              for j in range(1, chunk_size - 1):
                  for k in range(1, chunk_size - 1):
                      density = chunk_data[i, j, k]
                      if density >= iso_value:
                          continue
                      
                      # Check left face (i-1)
                      if i == 1 or chunk_data[i-1, j, k] < iso_value:
                          # Create left face vertices and triangles
                          # ... (see S1 for specific vertex math)
                          pass
                      
                      # Check bottom face (j-1)
                      if j == 1 or chunk_data[i, j-1, k] < iso_value:
                          # Create bottom face vertices and triangles
                          # ... (see S1 for specific vertex math)
                          pass
          return vertices, triangles
      ```
7. **Procedural Bypass Logic:** Pure procedural chunks are regenerated from the seed rather than saved to disk to minimize I/O overhead [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Precision:** Floating-point precision may affect voxel center calculations. Use double precision for distance checks if necessary [S1, unofficial].
- **Boundary Continuity:** Ensure the halo border is correctly handled to avoid gaps between chunks [S2, unofficial].
- **Memory Leaks:** Ensure chunks are properly removed from the registry when unloaded [S1, unofficial].
- **Determinism:** The procedural generation must be deterministic to ensure consistency when chunks are reloaded [S1, unofficial].

## Subtopics

### Procedural Bypass Logic
Pure procedural chunks are regenerated from seed rather than saved to disk to minimize I/O overhead [S2, unofficial]. This approach is suitable for demos or scenarios where disk space is limited or I/O latency is high. It trades CPU cycles for reduced disk usage.

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of chunks within the load radius. Generation is deterministic and can be cached.
- **Memory Complexity:** $O(M)$ where $M$ is the number of active chunks. Each chunk consumes memory proportional to its size ($50^3$ voxels) plus geometry data.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales well with world size as only a fixed number of chunks are active at any time. Performance depends on the complexity of the procedural generation algorithm and the mesh generation step.
- **Strengths:** Enables infinite worlds, reduces initial load time, minimizes disk I/O.
- **Limitations:** Requires efficient procedural generation and meshing. Potential for pop-in if chunks are loaded too late.
- **When to use:** Large or infinite procedural worlds.
- **When not to use:** Small, static worlds where pre-generation is feasible and faster.

## Background
Originated in voxel-based game engines and procedural generation systems. The specific implementation details are drawn from unofficial sources [S1, unofficial][S2, unofficial].

## Sources
[S1] lijiemark/floating-islands-generator, https://github.com/lijiemark/floating-islands-generator, fetched 2026-10-27 (unofficial)
[S2] herbal1st/pyvorengi-sdk-demo, https://github.com/herbal1st/pyvorengi-sdk-demo, fetched 2026-10-27 (unofficial)
