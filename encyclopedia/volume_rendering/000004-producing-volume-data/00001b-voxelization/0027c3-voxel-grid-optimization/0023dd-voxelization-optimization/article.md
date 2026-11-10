# Voxelization Optimization


## Overview
Voxelization Optimization encompasses techniques to reduce the computational and memory costs of generating voxel grids from geometric primitives, particularly in real-time rendering pipelines. It addresses the high cost of numerous draw calls and vertex transformations required to populate a 3D texture or volume buffer. Unlike general volume data processing, these methods focus on the *production* phase, leveraging GPU-specific features like stream output and instancing to bypass redundant calculations for static or deforming geometry [S1, unofficial][S2, unofficial].

## How it works
The optimization strategies operate on the inputs of the voxelization pipeline (mesh vertices, transformation matrices, slice targets) to minimize the number of GPU operations and memory bandwidth usage.

### Inputs and Outputs
*   **Inputs**: Source mesh geometry (vertices, indices), transformation matrices (rigid or skinning), camera/view parameters, and voxel grid resolution.
*   **Outputs**: A populated voxel grid (3D texture or structured buffer) containing occupancy or density values.

### Data Structures
1.  **Transformed Vertex Buffer**: A GPU buffer storing vertices after transformation. Size: $N \times V$, where $N$ is the number of vertices and $V$ is the vertex size (e.g., 3 floats for position). [S1, unofficial]
2.  **Heightmaps (Min/Max Altitude)**: Two 2D arrays storing the vertical extent of occupied voxels per column.
    *   `minAltitude`: Byte array of size $C^2$ (where $C$ is chunk size, e.g., 32). Stores the lowest occupied Y index. Initialized to $C$ (empty). [S3, unofficial]
    *   `maxAltitude`: Byte array of size $C^2$. Stores the highest occupied Y index. [S3, unofficial]

### Steps

#### 1. Stream Output Caching for Deforming Meshes
For meshes undergoing deformation (e.g., skinned characters), vertex transformations are expensive if repeated for every slice.
1.  **Cache Transformed Vertices**: Use GPU Stream Output to capture the output of the vertex shader (transformed positions) into a buffer. [S1, unofficial]
2.  **Reuse Buffer**: Subsequent slice rasterizations read from this cached buffer instead of re-executing the vertex shader for each slice. [S1, unofficial]

#### 2. Instanced Slice Rendering
To avoid $N$ draw calls for $N$ slices:
1.  **Prepare Instance Data**: Create an instance buffer where each instance corresponds to a slice index $s$. [S1, unofficial]
2.  **Single Draw Call**: Issue one draw call with instancing.
3.  **Slice Selection**: In the geometry or fragment shader, use the instance ID to determine the target slice coordinate (e.g., $Z = \text{instanceID}$). [S1, unofficial]

#### 3. Low-Poly Obstacle Approximation
To speed up voxelization of complex static obstacles:
1.  **Generate LOD Mesh**: Create a simplified (low-polygon) version of the obstacle mesh. [S1, unofficial]
2.  **Voxelize LOD**: Use the LOD mesh for the voxelization pass. This reduces the number of triangles processed per slice. [S1, unofficial]

#### 4. Min/Max Altitude Heightmap Optimization
Used to skip empty or fully occupied columns during subsequent meshing or traversal phases.
1.  **Initialization**: Set `minAltitude[i][j] = Constants.ChunkSize` for all $i, j \in [0, C-1]$. [S3, unofficial]
2.  **Voxel Addition**: When a voxel at $(x, y, z)$ is added (index > 0):
    *   Update `minAltitude[x][z] = min(minAltitude[x][z], y)`
    *   Update `maxAltitude[x][z] = max(maxAltitude[x][z], y)`
3.  **Voxel Removal**: When a voxel at $(x, y, z)$ is removed (index == 0):
    *   Recompute `minAltitude[x][z]` and `maxAltitude[x][z]` by scanning the column or maintaining a count of occupied voxels per height. [S3, unofficial]

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Stream Output Caching
def voxelize_deforming_mesh(mesh, slices, stream_output_buffer):
    """
    Voxelize a deforming mesh using cached transformed vertices.
    """
    # Step 1: Transform vertices once and cache them
    # In a real GPU pipeline, this is done via a Vertex Shader -> Stream Output stage
    transformed_vertices = transform_vertices(mesh.vertices, mesh.skinning_matrices)
    stream_output_buffer.write(transformed_vertices)

    # Step 2: Render slices using instancing
    for slice_idx in slices:
        # Use instance ID to set the slice coordinate in the shader
        draw_instanced(
            geometry=stream_output_buffer,
            instance_count=1,
            instance_data={'slice_z': slice_idx}
        )

# Min/Max Altitude Update
def update_heightmaps(chunk_size, min_altitude, max_altitude, voxel_grid, x, y, z, is_addition):
    """
    Update min/max altitude heightmaps when a voxel changes state.
    """
    if is_addition:
        # Update min and max for the column (x, z)
        if y < min_altitude[x][z]:
            min_altitude[x][z] = y
        if y > max_altitude[x][z]:
            max_altitude[x][z] = y
    else:
        # Removal requires re-scanning the column to find new bounds
        # or maintaining a count of occupied voxels per height level
        new_min = chunk_size
        new_max = -1
        for h in range(chunk_size):
            if voxel_grid[x][h][z] > 0:
                if h < new_min:
                    new_min = h
                if h > new_max:
                    new_max = h
        min_altitude[x][z] = new_min
        max_altitude[x][z] = new_max
```

### Parameters
*   **Chunk Size**: Typically 32 voxels per side for heightmap optimization [S3, unofficial].
*   **LOD Level**: Determined by the visual importance of the obstacle; simplified models reduce triangle count significantly [S1, unofficial].

### Edge Cases and Pitfalls
*   **Dynamic Lights**: Static objects must be re-voxelized if dynamic lights intersect them, as lighting contributions change [S2, unofficial].
*   **Precision**: Stream output buffers must be large enough to hold all transformed vertices; overflow leads to data loss [S1, unofficial].
*   **Heightmap Consistency**: On voxel removal, naive updates may leave stale min/max values if the removed voxel was the boundary. A full column scan or auxiliary count structure is required [S3, unofficial].

## Subtopics

### [32nd Bit Avoidance Modification](3351-32nd-bit-avoidance-modification.md)
The 32nd Bit Avoidance Modification adjusts the solid voxelization and density computation pipelines to exclude the highest bit of 32-bit storage formats, which is reserved for other purposes. It restricts voxel and density data to the lower bits, distinguishing it from general sparse voxelization by targeting bit-packing constraints in dense grids.

### [Sparse Voxelization with Oracle and Parallel Scan](3687-sparse-voxelization-with-oracle-and-parallel-scan.md)
Sparse Voxelization with Oracle and Parallel Scan is a real-time, high-resolution technique for reconstructing 3D surface models from image-based inputs (calibrated RGB camera streams) using a coarse-to-fine voxel hierarchy. It solves the problem of on-the-fly surface reconstruction by iteratively refining a sparse list of voxels, using an oracle kernel to cull empty space and a parallel prefix sum scan to compact the list, ensuring that only relevant voxels are subdivided. Unlike bottom-up approaches or methods designed for static models, this method is top-down and optimized for interactive rates (e.g., >30 fps at 1024^3 resolution) by leveraging GPU parallelism for both computation and rendering.

### [Greedy Voxel Mesh Optimization](3979-greedy-voxel-mesh-optimization.md)
Greedy Voxel Mesh Optimization is a greedy algorithm that optimizes voxel mesh generation for 2D tile grids (e.g., Minecraft-like games) by merging adjacent identical tiles into larger rectangular quads to reduce draw calls. It differs from its siblings, [32nd Bit Avoidance Modification](3351-32nd-bit-avoidance-modification.md) and [Sparse Voxelization with Oracle and Parallel Scan](3687-sparse-voxelization-with-oracle-and-parallel-scan.md), by explicitly constructing maximal rectangular regions of uniform tile properties through iterative expansion, rather than relying on bit-packing tricks or parallel scan-based sparse structures.

### Stream Output for Voxelization
A technique to optimize voxelization of deforming meshes by caching transformed vertices, avoiding recomputation for each slice [S1, unofficial]. It uses stream output to cache an entire buffer of transformed vertices during the voxelization of deforming meshes (e.g., skinned characters) [S1, unofficial].

### Min/Max Altitude Heightmap Optimization
Stores per-column minimum and maximum occupied voxel heights to skip empty or fully occupied columns during meshing [S3, unofficial]. It updates heightmaps on voxel addition and removal, initializing minAltitude to Constants.ChunkSize (32) for all columns to indicate initially empty columns [S3, unofficial].

## Operating characteristics
*   **Time Complexity**:
    *   Stream Output: $O(V)$ for initial transformation, $O(1)$ per slice for rasterization (vs $O(V)$ per slice without caching). [S1, unofficial]
    *   Instancing: Reduces draw call overhead from $O(S)$ to $O(1)$, where $S$ is the number of slices. [S1, unofficial]
    *   Heightmap Update: $O(1)$ for addition, $O(H)$ for removal (where $H$ is column height), or $O(1)$ with auxiliary counts. [S3, unofficial]
*   **Memory Complexity**:
    *   Stream Output: Requires additional GPU memory for the vertex buffer ($O(V)$). [S1, unofficial]
    *   Heightmaps: $O(C^2)$ per chunk, where $C$ is chunk size. [S3, unofficial]
*   **Performance**:
    *   Using LOD meshes allows voxelizing obstacles at every frame with little performance cost [S1, unofficial].
    *   Techniques are used in commercial titles like *Rise of Tomb Raider* (voxel AO) and *The Tomorrow Children* (consoles) [S2, unofficial].
*   **Strengths**: Significant reduction in GPU workload for dynamic scenes; efficient use of modern GPU features (instancing, stream output).
*   **Limitations**: Requires careful management of buffer sizes; static objects may still require re-voxelization due to dynamic lighting [S2, unofficial].

## Background
Voxelization optimization techniques have evolved alongside real-time rendering demands. Stream output and instancing were introduced to mitigate the bottleneck of multiple draw calls and vertex transformations [S1, unofficial]. The use of LOD meshes for voxelization was highlighted in *GPU Gems 3* for real-time fluid simulation [S1, unofficial]. Heightmap-based optimizations are common in voxel-based game engines (e.g., Minecraft-like games) to accelerate mesh generation [S3, unofficial]. Sparse Voxel Octrees and voxel cascades have been explored to further reduce memory and update costs [S2, unofficial].

## Sources
[S1] Real-Time Simulation and Rendering of 3D Fluids, https://wo1fsea.github.io/files/About_NavMesh/P5_GPU_Gems_3_Real-Time_Simulation_and_Rendering_of_3D_Fluids.pdf, fetched 2026-10-27 (unofficial)
[S2] Voxel-based Global Illumination – Wicked Engine, https://turanszkij.wordpress.com/2017/08/30/voxel-based-global-illumination/, fetched 2026-10-28 (unofficial)
[S3] vercidium-patreon/meshing, https://github.com/vercidium-patreon/meshing, fetched 2026-10-26 (unofficial)
