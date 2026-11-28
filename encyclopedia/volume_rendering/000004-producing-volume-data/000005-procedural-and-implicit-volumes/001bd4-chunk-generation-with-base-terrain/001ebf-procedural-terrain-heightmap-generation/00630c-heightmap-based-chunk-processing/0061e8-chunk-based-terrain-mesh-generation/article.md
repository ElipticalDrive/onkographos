# Chunk-based Terrain Mesh Generation

Also called: Chunked Terrain Mesh Generation, Spatial Chunk Terrain Generation

## Overview
Chunk-based terrain mesh generation creates a 3D terrain mesh from a procedural noise function by dividing the visible area into spatial chunks for efficient real-time rendering [S1, unofficial]. It solves the problem of generating seamless, large-scale terrain meshes on the CPU to free GPU resources, differing from direct shader sampling by pre-calculating height maps [S1, unofficial]. It builds on [Heightmap-based Chunk Processing](18783-heightmap-based-chunk-processing.md) by explicitly managing the mesh topology and normal calculation across chunk boundaries.

## How it works
**Inputs:**
- Noise function $f(x, z)$ capable of returning a height value for any 2D coordinate.
- Chunk size parameter $N$ (vertices per dimension).
- Global height scaling constant $H_{scale}$.
- View frustum or camera position to determine visible chunks.

**Outputs:**
- A collection of terrain meshes, one per visible chunk, ready for rendering.

**Data Structures:**
- **Chunk Grid:** A 2D array of size $(N+2) \times (N+2)$ floats, storing height samples. The extra border of 1 vertex on each side is required for normal calculation.
- **Mesh Data:** Vertex positions $(x, y, z)$ and normals $(n_x, n_y, n_z)$ for the inner $N \times N$ grid points.

**Steps:**
1. **Determine Visible Chunks:** Identify the set of chunks $(c_x, c_z)$ that intersect the current view frustum or camera area.
2. **Allocate Sample Array:** For each visible chunk, allocate a 2D array `heights` of size $(N+2) \times (N+2)$.
3. **Sample Noise Function:** Iterate over all indices $i, j$ from $0$ to $N+1$. Calculate the world coordinate for each sample point and evaluate the noise function:
   $$ h_{i,j} = f(x_{world}, z_{world}) $$
   where $x_{world}$ and $z_{world}$ correspond to the grid position relative to the chunk's origin.
4. **Calculate Normals:** Compute vertex normals for the inner $N \times N$ vertices using the height values from the border. For an inner vertex at $(i, j)$ where $1 \le i, j \le N$, the normal is derived from the cross product of tangent vectors formed by adjacent height samples:
   $$ \vec{t}_x = (1, h_{i+1,j} - h_{i-1,j}, 0) $$
   $$ \vec{t}_z = (0, h_{i+1,j} - h_{i-1,j}, 1) $$
   $$ \vec{n} = \vec{t}_x \times \vec{t}_z $$
   Normalize $\vec{n}$. The border values ($i=0, i=N+1, j=0, j=N+1$) are strictly required for this step to ensure correct normals at chunk edges [S1, unofficial].
5. **Calculate Vertex Positions:** Generate the final vertex positions for the mesh using the inner $N \times N$ height values. Scale the height by the global constant:
   $$ y_{final} = h_{i,j} \times H_{scale} $$
   The $x$ and $z$ coordinates are determined by the chunk's world position and the vertex index within the chunk.
6. **Dispose Unused Chunks:** Remove mesh data for chunks that are no longer in the visible area to manage memory.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_terrain_mesh(noise_func, chunk_size, global_scale, camera_view):
    """
    Generates terrain meshes for visible chunks.
    
    Args:
        noise_func: Callable function f(x, z) -> float
        chunk_size: Integer N, number of vertices per dimension
        global_scale: Float H_scale for height adjustment
        camera_view: Object with method get_visible_chunks() returning list of (cx, cz)
        
    Returns:
        List of Mesh objects
    """
    visible_chunks = camera_view.get_visible_chunks()
    meshes = []
    
    for cx, cz in visible_chunks:
        # Step 2: Allocate (N+2)x(N+2) array
        # Using flat list for simplicity, or 2D array
        heights = [[0.0 for _ in range(chunk_size + 2)] for _ in range(chunk_size + 2)]
        
        # Step 3: Sample noise function
        for i in range(chunk_size + 2):
            for j in range(chunk_size + 2):
                # Calculate world coordinates for sample point
                # Assuming chunk origin is at (cx * chunk_size, 0, cz * chunk_size)
                # and each vertex is spaced by 1 unit
                world_x = cx * chunk_size + i
                world_z = cz * chunk_size + j
                heights[i][j] = noise_func(world_x, world_z)
                
        # Step 4 & 5: Generate vertices and normals for inner N x N grid
        vertices = []
        normals = []
        
        for i in range(1, chunk_size + 1):
            for j in range(1, chunk_size + 1):
                # Calculate normal using border values
                # Tangent in X direction: difference between right and left neighbors
                dx = heights[i+1][j] - heights[i-1][j]
                # Tangent in Z direction: difference between front and back neighbors
                dz = heights[i][j+1] - heights[i][j-1]
                
                # Normal vector calculation (cross product of (1, dx, 0) and (0, dz, 1))
                # n = (dx * 1 - 0 * dz, 0 * 0 - 1 * 1, 1 * dz - dx * 0)
                # n = (dx, -1, dz)
                nx = dx
                ny = -1.0
                nz = dz
                
                # Normalize normal
                length = (nx*nx + ny*ny + nz*nz) ** 0.5
                if length > 0:
                    nx /= length
                    ny /= length
                    nz /= length
                    
                # Calculate final vertex position
                world_x = cx * chunk_size + i
                world_z = cz * chunk_size + j
                world_y = heights[i][j] * global_scale
                
                vertices.append((world_x, world_y, world_z))
                normals.append((nx, ny, nz))
                
        # Create mesh from vertices and normals
        # Note: Triangulation logic (e.g., simple grid triangulation) is implied
        # but not detailed in source. Standard grid triangulation connects
        # vertex (i,j) to (i+1,j), (i,j+1), (i+1,j+1).
        mesh = create_mesh_from_grid(vertices, normals, chunk_size)
        meshes.append(mesh)
        
    return meshes
```

**Parameters:**
- $N$: Chunk size in vertices per dimension. Typical values depend on desired resolution and memory constraints. The sampling array is sized $(N+2) \times (N+2)$ [S1, unofficial].
- $H_{scale}$: Global constant to scale the final height range. The range is determined by the noise function's layers and redistribution function [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Border Values:** Omitting border values in the $(N+2) \times (N+2)$ array causes artifacts at chunk edges because normals cannot be calculated correctly for vertices adjacent to the chunk boundary [S1, unofficial].
- **Seamlessness:** The method ensures seamless fitting to neighboring chunks by using shared border samples for normal calculation, ensuring consistent slopes across chunk boundaries [S1, unofficial].

## Subtopics

### [Terrain Chunk Update](21228-terrain-chunk-update.md)
Updates a terrain grid when the camera moves by shifting existing vertex data and regenerating only the newly exposed rows/columns. It differs from full regeneration by reusing valid vertex data, reducing computational cost for small camera displacements.

### Terrain Preview Generator
A debug/preview tool that generates a small, isolated terrain chunk grid to visualize noise parameters without affecting the main world manager [S2, unofficial]. It clears existing chunks, iterates through a grid defined by `testSize`, creates GameObjects for each chunk, and initializes them with noise data using a `chunkOffset` to shift the pattern [S2, unofficial]. Parameters include `testSize` (1-20, default 1) and `chunkOffset` (2D integer vector) [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(V \cdot C)$, where $V$ is the number of visible chunks and $C$ is the number of vertices per chunk ($N^2$). Each vertex requires a noise evaluation and normal calculation.
- **Memory Complexity:** $O(V \cdot N^2)$ for storing height maps and mesh data.
- **Performance:** Calculating the height map on the CPU frees GPU resources for other tasks, whereas direct sampling in the shader consumes shader cycles [S1, unofficial]. This trade-off depends on the application's GPU/CPU balance.
- **Strengths:** Efficient real-time rendering, seamless mesh generation, CPU offloading.
- **Limitations:** Requires pre-calculation, potential memory overhead for large visible areas.
- **When to Use:** Real-time applications requiring large, seamless procedural terrains where GPU resources are needed for other effects.
- **When Not to Use:** When dynamic, per-vertex noise evaluation is preferred for extreme memory constraints or when the terrain is very small.

## Background
Originated in the context of terrain synthesis using noise functions, particularly in academic prototypes for assessing preliminary results of terrain design processes [S1, unofficial]. The method leverages random access features of procedural noise to calculate terrain in chunks, a common technique in game engines and simulation software for managing large-scale environments.

## Sources
[S1] Terrain synthesis using noise, https://trepo.tuni.fi/handle/10024/101043, fetched 2026-10-26 (unofficial)
[S2] Javier-Garzo/Marching-cubes-on-Unity-3D, https://github.com/Javier-Garzo/Marching-cubes-on-Unity-3D, fetched 2026-10-26 (unofficial)
