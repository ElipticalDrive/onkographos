# Marching Cubes Chunk Generation Pipeline

Also called: Chunk-based Marching Cubes, Procedural Terrain Generation with Marching Cubes

## Overview
This method is a procedural terrain generation pipeline that divides a 3D volume into overlapping chunks, populates them with noise-derived node values, and prepares chunk meshes for Marching Cubes surface extraction [S1, unofficial]. It builds on [GPU Marching Cubes Terrain Pipeline with Indirect Rendering](3803-gpu-marching-cubes-terrain-pipeline-with-indirect-rendering.md) by handling the CPU-side data preparation and chunk management required for the GPU-accelerated extraction. Unlike single-volume extraction, this approach manages spatial locality and boundary continuity through overlapping node grids, ensuring seamless transitions between adjacent chunks [S1, unofficial].

## How it works
**Inputs and Outputs**
- **Inputs**: Chunk dimensions (`xChunks`, `yChunks`, `zChunks`), resolution enum (`cubeResolution`), noise parameters (`seed`, `scale`, `octaves`, `persistence`, `lacunarity`, `dim`), and isovalue threshold (`surfaceLevel`) [S1, unofficial].
- **Outputs**: A collection of `ChunkMatrix` objects, each containing a 3D array of `Node` structures with position and value, ready for the Marching Cubes lookup and configuration step [S1, unofficial].

**Data Structures**
1.  **`Node`**: A struct containing:
    -   `position`: `Vector3` representing the 3D coordinate in world space.
    -   `value`: `float` representing the scalar field value at that node.
2.  **`ChunkMatrix`**: A class containing:
    -   `nodes`: 3D array of `Node` objects. Dimensions are `(xChunkLength + 2) x (yChunkLength + 2) x (zChunkLength + 2)` to include overlapping boundary nodes [S1, unofficial].
    -   `vertices`: List of `Vector3` for generated mesh vertices.
    -   `indices`: List of `int` for generated mesh indices.
    -   `chunkObj`: Reference to the GameObject representing the chunk.
    -   `chunkMesh`: Reference to the MeshFilter/Mesh component.
    -   `chunksIndex`: `int[3]` storing the chunk's grid coordinates (x, y, z).
    -   `position`: `Vector3` storing the chunk's world-space origin.
3.  **`mapNoise`**: A 3D `float` array of size `[xMatrix, yMatrix, zMatrix]` storing precomputed noise values for the entire volume [S1, unofficial].

**Parameters**
-   `cubeResolution`: Enum controlling voxel density per chunk. Values: `INSANE=8`, `HIGH=4`, `MED=2`, `LOW=1` [S1, unofficial].
-   `surfaceLevel`: Isovalue threshold for Marching Cubes. Range `[0, 1]`, default `0.6f` [S1, unofficial].
-   `xChunks`, `yChunks`, `zChunks`: Number of chunks in each dimension. Defaults: `1` [S1, unofficial].
-   `seed`: Noise generation seed. Default `0` [S1, unofficial].
-   `scale`: Noise frequency multiplier. Default `0.15f`. Scaled by `cubeResolution` during initialization [S1, unofficial].
-   `octaves`: Number of noise octaves. Range `[0, 6]`, default `4` [S1, unofficial].
-   `persistence`: Amplitude multiplier per octave. Range `[0, 1]`, default `0.5f` [S1, unofficial].
-   `lacunarity`: Frequency multiplier per octave. Range `[0, 10]`, default `2` [S1, unofficial].
-   `dim`: Noise type selector. Options: `TWO_DIMENSIONAL_NOISE` (heightmap), `THREE_DIMENSIONAL_NOISE` (volumetric), `HALF_FULL` (binary split) [S1, unofficial].
-   `openMiddle`: Parameter for 2D heightmap noise generation. Default `10f` [S1, unofficial].
-   `softEdge`: Boolean flag enabling interpolation between voxels. Default `true` [S1, unofficial].

**Steps**
1.  **Calculate Global Dimensions**: Compute the total matrix dimensions for the noise map.
    $$xMatrix = (xChunkCubeLength - 2) \times xChunks + 2$$
    Apply the same formula for $yMatrix$ and $zMatrix$, where $xChunkCubeLength$ is the base chunk resolution derived from `cubeResolution` [S1, unofficial].
2.  **Generate Noise Map**: Call `Noise.Generate2DNoise`, `Generate3DNoise`, or `GenerateHallFullField` based on the `dim` enum. Fill the `mapNoise` array with dimensions `[xMatrix, yMatrix, zMatrix]` [S1, unofficial].
3.  **Iterate Chunks**: Loop through chunk coordinates $y$ from $0$ to $yChunks$, $z$ from $0$ to $zChunks$, and $x$ from $0$ to $xChunks$. For each chunk, call `CreateChunkMatrix(x, y, z)` [S1, unofficial].
4.  **Initialize Chunk Nodes**: Inside `CreateChunkMatrix`, loop over the chunk dimensions. Instantiate `Node(Vector3.zero, 0f)` for each grid position. The node grid size is `(xChunkLength + 2) x (yChunkLength + 2) x (zChunkLength + 2)` [S1, unofficial].
5.  **Set Chunk Position**: Calculate the world-space position of the chunk.
    $$currChunkMatrix.position = \left( \frac{x \times xChunkLength}{cubeResolution}, \frac{y \times yChunkLength}{cubeResolution}, \frac{z \times zChunkLength}{cubeResolution} \right)$$
    Note: The source code divides by `cubeResolution` to normalize coordinates, implying `xChunkLength` is in voxel units relative to the resolution enum [S1, unofficial].
6.  **Populate Node Values**: Assign values to each node.
    -   For internal nodes (non-boundary), sample from `mapNoise` using the chunk's local offset.
    -   For boundary nodes (the +2 padding), sample from `mapNoise` using the global coordinates to ensure overlap with adjacent chunks. This solves the seam artifact issue [S1, unofficial].
    -   If `softEdge` is true, apply interpolation between voxels [S1, unofficial].
7.  **Initialize Winding Order**: Set `windingOrder = new int[] { 2, 1, 0 }` to demonstrate triangle direction consistency [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_chunks(x_chunks, y_chunks, z_chunks, cube_resolution, surface_level, seed, scale, octaves, persistence, lacunarity, dim, open_middle, soft_edge):
    # Map resolution enum to base length
    # INSANE=8, HIGH=4, MED=2, LOW=1
    base_length = {
        'INSANE': 8,
        'HIGH': 4,
        'MED': 2,
        'LOW': 1
    }[cube_resolution]
    
    # Calculate global noise map dimensions
    # Formula: (chunk_length - 2) * num_chunks + 2
    x_matrix = (base_length - 2) * x_chunks + 2
    y_matrix = (base_length - 2) * y_chunks + 2
    z_matrix = (base_length - 2) * z_chunks + 2
    
    # Generate global noise map
    map_noise = generate_noise(dim, x_matrix, y_matrix, z_matrix, seed, scale * cube_resolution, octaves, persistence, lacunarity, open_middle)
    
    chunks = []
    
    for y in range(y_chunks):
        for z in range(z_chunks):
            for x in range(x_chunks):
                chunk = create_chunk_matrix(x, y, z, base_length, cube_resolution, map_noise, soft_edge)
                chunks.append(chunk)
                
    return chunks

def create_chunk_matrix(x, y, z, base_length, cube_resolution, map_noise, soft_edge):
    # Chunk node dimensions include 2 extra nodes per dimension for overlap
    chunk_x_len = base_length + 2
    chunk_y_len = base_length + 2
    chunk_z_len = base_length + 2
    
    nodes = [[[] for _ in range(chunk_z_len)] for _ in range(chunk_y_len)]
    for i in range(chunk_y_len):
        for j in range(chunk_z_len):
            nodes[i][j] = [None] * chunk_x_len
            
    # Set chunk world position
    # Note: Source divides by cube_resolution to normalize
    pos_x = (x * base_length) / cube_resolution
    pos_y = (y * base_length) / cube_resolution
    pos_z = (z * base_length) / cube_resolution
    position = Vector3(pos_x, pos_y, pos_z)
    
    # Populate nodes
    for i in range(chunk_y_len):
        for j in range(chunk_z_len):
            for k in range(chunk_x_len):
                # Calculate global noise map coordinates
                # The chunk is offset by (x * (base_length - 2)) in the global map
                # But the chunk itself has size base_length + 2
                # The overlap logic implies the chunk's internal grid maps to the global map
                # with a stride of (base_length - 2)
                
                global_x = x * (base_length - 2) + k
                global_y = y * (base_length - 2) + i
                global_z = z * (base_length - 2) + j
                
                # Clamp or handle boundaries if necessary, though formula ensures valid indices
                # if 0 <= global_x < x_matrix and 0 <= global_y < y_matrix and 0 <= global_z < z_matrix
                
                noise_value = map_noise[global_x][global_y][global_z]
                
                # Calculate world position for the node
                node_pos = Vector3(
                    pos_x + (k / cube_resolution),
                    pos_y + (i / cube_resolution),
                    pos_z + (j / cube_resolution)
                )
                
                nodes[i][j][k] = Node(node_pos, noise_value)
                
    chunk = ChunkMatrix()
    chunk.nodes = nodes
    chunk.position = position
    chunk.chunks_index = [x, y, z]
    chunk.vertices = []
    chunk.indices = []
    
    return chunk
```

**Edge Cases and Pitfalls**
-   **Chunk Boundary Seams**: Without overlapping boundary nodes, transitions between chunks will show artifacts. This is solved by adding 2 extra nodes per dimension to each chunk's node grid, which are filled with noise from the global `mapNoise` array [S1, unofficial].
-   **Resolution Scaling Mismatch**: Brush interaction or editing values must be scaled by `cube_resolution` to maintain consistent behavior across different voxel densities [S1, unofficial].

## Subtopics

### Marching Cubes Lookup and Configuration
The next step that processes the `ChunkMatrix` nodes to generate triangle indices and vertices using the standard lookup table. [Marching Cubes Lookup and Configuration](52-marching-cubes-lookup-and-configuration.md)

### GPU Marching Cubes Terrain Pipeline with Indirect Rendering
The parent pipeline that manages LOD and renders the generated chunks via indirect draw calls. [GPU Marching Cubes Terrain Pipeline with Indirect Rendering](3803-gpu-marching-cubes-terrain-pipeline-with-indirect-rendering.md)

## Operating characteristics
-   **Time Complexity**: $O(N \cdot M)$ where $N$ is the number of chunks and $M$ is the number of nodes per chunk. Noise generation is $O(V)$ where $V$ is the total volume size.
-   **Memory Complexity**: $O(V)$ for the global noise map, plus $O(N \cdot M)$ for the chunk node structures.
-   **Strengths**: Modular generation allows for streaming or partial updates. Overlapping nodes ensure visual continuity without complex post-processing.
-   **Limitations**: CPU-bound noise generation and chunk management can become a bottleneck for very large worlds or high resolutions. Not given in the sources for specific measured performance.
-   **When to use**: Procedural terrain generation where seamless chunking is required.
-   **When not to use**: Real-time dynamic voxel editing where global noise recalculation is too expensive.

## Background
This method originates from an educational Marching Cubes project by millerc3, designed to be readable and followable for learners transitioning from C/C++ to Unity/C# [S1, unofficial]. It is based heavily on Paul Bourke's original Marching Cubes algorithm and the Scrawk repository implementation [S1, unofficial]. The project aims to provide a clear, unoptimized reference implementation for understanding the algorithm and engine integration.

## Sources
[S1] millerc3/Marching-Cubes, https://github.com/millerc3/Marching-Cubes, fetched 2026-10-26 (unofficial)
