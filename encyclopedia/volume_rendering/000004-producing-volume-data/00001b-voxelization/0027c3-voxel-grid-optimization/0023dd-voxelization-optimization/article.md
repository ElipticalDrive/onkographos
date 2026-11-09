# Voxelization Optimization


## Overview
Voxelization Optimization encompasses techniques to reduce the computational and memory costs of generating voxel grids from geometric primitives, addressing the high draw-call overhead and redundant vertex transformations inherent in naive voxelization. It distinguishes itself from sibling methods like [Sparse Voxel Octrees](2628-occupancy-voxel-creation.md) or [Binary Voxel RLE Encoding](1071-binary-voxel-rle-encoding.md) by focusing on the *generation pipeline* efficiency (caching, instancing, LOD) rather than the *storage* or *post-processing* of the resulting grid.

## How it works
The method optimizes the voxelization pipeline through four primary mechanisms: vertex caching via stream output, draw-call reduction via instancing, geometric simplification via LOD, and static/dynamic separation.

### Inputs and Outputs
- **Input**: Source geometry (meshes), voxel grid parameters (resolution, bounds), transformation matrices (model-view-projection), and animation data (for deforming meshes).
- **Output**: Optimized voxel grid data structure (dense or sparse) and reduced GPU command buffer.

### Data Structures
1.  **Transformed Vertex Buffer**: A GPU-resident buffer storing pre-transformed vertex positions and normals. Size: $N \times V$, where $N$ is the number of vertices and $V$ is the vertex size (e.g., 3 floats for position). [S1, unofficial]
2.  **Instance Buffer**: Contains per-slice transformation matrices or slice indices. Size: $S \times M$, where $S$ is the number of slices and $M$ is the matrix size (e.g., 4x4 float matrix). [S1, unofficial]
3.  **Min/Max Altitude Maps** (Optional, for heightmap-based optimization):
    - `minAltitude`: Byte array of size $C^2$, storing the lowest occupied Y-index per XZ column. Initialized to $C$ (empty). [S3, unofficial]
    - `maxAltitude`: Byte array of size $C^2$, storing the highest occupied Y-index per XZ column. [S3, unofficial]
    - Where $C$ is the chunk size (e.g., 32). [S3, unofficial]

### Steps
1.  **Preprocessing (Static vs. Dynamic)**:
    - Identify static obstacles. Do not re-voxelize them unless a dynamic light intersects their bounding volume. [S2, unofficial]
    - For static objects with simple analytic transformations (rigid translation/rotation), precompute the voxelization and apply the inverse transformation during access. [S1, unofficial]
    - For deforming meshes (e.g., skinned characters), enable Stream Output optimization. [S1, unofficial]

2.  **Stream Output Caching (Deforming Meshes)**:
    - Render the mesh to a vertex buffer using a geometry shader or compute shader that outputs transformed vertices.
    - Cache this buffer. In subsequent frames, reuse the cached transformed vertices instead of recomputing skinning/transformations for each slice. [S1, unofficial]

3.  **LOD Selection**:
    - Replace high-poly obstacle meshes with low-poly Level of Detail (LOD) meshes for the voxelization pass. This reduces the number of primitives processed per slice. [S1, unofficial]

4.  **Instanced Slicing**:
    - Instead of issuing $S$ separate draw calls (one per slice), use instanced rendering.
    - Pass the slice index or slice-specific transformation matrix as an instance attribute.
    - Execute a single draw call that processes all slices. [S1, unofficial]

5.  **Heightmap Maintenance (For Chunked/Heightmap Systems)**:
    - When adding a voxel at $(x, y, z)$:
        - If the voxel index becomes non-zero (occupied):
            - Update `minAltitude[x][z] = min(minAltitude[x][z], y)`
            - Update `maxAltitude[x][z] = max(maxAltitude[x][z], y)`
        - [S3, unofficial]
    - When removing a voxel at $(x, y, z)$:
        - If the voxel index becomes zero (empty):
            - Recalculate `minAltitude[x][z]` by scanning the column $y \in [0, C)$ for the first occupied voxel.
            - Recalculate `maxAltitude[x][z]` by scanning the column $y \in [0, C)$ for the last occupied voxel.
            - [S3, unofficial]
    - Initialization: Set all `minAltitude` entries to $C$ (indicating empty). [S3, unofficial]

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def optimize_voxelization(meshes, grid_params, is_deforming):
    # Step 1: LOD Selection
    lod_meshes = [mesh.get_lowest_lod() for mesh in meshes]
    
    # Step 2: Static/Dynamic Separation
    static_meshes = [m for m in lod_meshes if not m.is_dynamic()]
    dynamic_meshes = [m for m in lod_meshes if m.is_dynamic()]
    
    # Cache transformed vertices for dynamic meshes using Stream Output
    if is_deforming:
        cached_vertices = stream_output_cache(dynamic_meshes)
    else:
        cached_vertices = None
        
    # Step 3: Instanced Rendering Setup
    slice_count = grid_params.slice_count
    instance_data = generate_instance_buffers(grid_params)
    
    # Step 4: Draw Calls
    # Single draw call for all slices
    draw_instanced(lod_meshes, instance_data, cached_vertices)
    
    return voxel_grid

def update_heightmaps(chunk, voxel_index, voxel_value):
    x, z = chunk.get_xz_from_index(voxel_index)
    y = chunk.get_y_from_index(voxel_index)
    
    if voxel_value > 0:
        # Voxel Added
        if y < chunk.min_altitude[x][z]:
            chunk.min_altitude[x][z] = y
        if y > chunk.max_altitude[x][z]:
            chunk.max_altitude[x][z] = y
    else:
        # Voxel Removed
        # Scan column to find new min/max
        new_min = chunk.chunk_size # Constants.ChunkSize
        new_max = -1
        
        for k in range(chunk.chunk_size):
            idx = chunk.get_index(x, k, z)
            if chunk.voxels[idx] > 0:
                if k < new_min:
                    new_min = k
                if k > new_max:
                    new_max = k
                    
        chunk.min_altitude[x][z] = new_min
        chunk.max_altitude[x][z] = new_max
```

### Parameters
- **LOD Level**: Determined by distance to camera or voxel resolution. [S1, unofficial]
- **Chunk Size**: Typically 32 voxels per side for heightmap optimization. [S3, unofficial]
- **Re-voxelization Threshold**: Dynamic lights intersecting static objects force re-voxelization. [S2, unofficial]

### Edge Cases and Pitfalls
- **Dynamic Lights**: Dynamic lights intersecting static geometry require the static geometry to be re-voxelized, negating the static optimization. [S2, unofficial]
- **Precision**: Stream output caching requires careful management of vertex buffer lifetimes to avoid stale data for deforming meshes. [S1, unofficial]
- **Heightmap Recalculation**: Removing a voxel requires a full column scan to update min/max, which is $O(C)$ per removal. [S3, unofficial]

## Subtopics

### [32nd Bit Avoidance Modification](3351-32nd-bit-avoidance-modification.md)
The 32nd Bit Avoidance Modification adjusts the solid voxelization and density computation pipelines to exclude the highest bit of 32-bit storage formats, which is reserved for other purposes. It restricts voxel and density data to the lower bits, distinguishing it from general sparse voxelization by targeting bit-packing constraints in dense grids.

### [Sparse Voxelization with Oracle and Parallel Scan](3687-sparse-voxelization-with-oracle-and-parallel-scan.md)
Sparse Voxelization with Oracle and Parallel Scan is a real-time, high-resolution technique for reconstructing 3D surface models from image-based inputs (calibrated RGB camera streams) using a coarse-to-fine voxel hierarchy. It solves the problem of on-the-fly surface reconstruction by iteratively refining a sparse list of voxels, using an oracle kernel to cull empty space and a parallel prefix sum scan to compact the list, ensuring that only relevant voxels are subdivided. Unlike bottom-up approaches or methods designed for static models, this method is top-down and optimized for interactive rates (e.g., >30 fps at 1024^3 resolution) by leveraging GPU parallelism for both computation and rendering.

### [Greedy Voxel Mesh Optimization](3979-greedy-voxel-mesh-optimization.md)
Greedy Voxel Mesh Optimization is a greedy algorithm that optimizes voxel mesh generation for 2D tile grids (e.g., Minecraft-like games) by merging adjacent identical tiles into larger rectangular quads to reduce draw calls. It differs from its siblings, [32nd Bit Avoidance Modification](3351-32nd-bit-avoidance-modification.md) and [Sparse Voxelization with Oracle and Parallel Scan](3687-sparse-voxelization-with-oracle-and-parallel-scan.md), by explicitly constructing maximal rectangular regions of uniform tile properties through iterative expansion, rather than relying on bit-packing tricks or parallel scan-based sparse structures.

### Stream Output for Voxelization
A technique to optimize voxelization of deforming meshes by caching transformed vertices, avoiding recomputation for each slice. Use stream output to cache an entire buffer of transformed vertices during the voxelization of deforming meshes (e.g., skinned characters). [S1, unofficial]

### Min/Max Altitude Heightmap Optimization
Stores per-column minimum and maximum occupied voxel heights to skip empty or fully occupied columns during meshing. On voxel addition (index > 0), `OnVoxelAdded(i, j, k)` updates the heightmaps to reflect the new occupied column. On voxel removal (index == 0), `OnVoxelRemoved(i, j, k)` updates the heightmaps to reflect the cleared column. Initialize `minAltitude` array to `Constants.ChunkSize` (32) for all columns, indicating initially empty columns. Parameters: `minAltitude` is a byte array of size `ChunkSizeSquared` (1024 bytes), storing the lowest occupied Y index per XZ column. `maxAltitude` is a byte array of size `ChunkSizeSquared` (1024 bytes), storing the highest occupied Y index per XZ column. [S3, unofficial]

## Operating characteristics
- **Time Complexity**:
    - Naive Voxelization: $O(S \times P)$, where $S$ is slice count and $P$ is primitive count.
    - Optimized (Instancing): $O(P)$ for geometry processing, $O(1)$ draw calls. [S1, unofficial]
    - Heightmap Update: $O(1)$ for addition, $O(C)$ for removal (column scan). [S3, unofficial]
- **Memory Complexity**:
    - Stream Output: Adds $O(V)$ memory for cached vertices. [S1, unofficial]
    - Sparse Voxel Octrees: Reduce memory requirements compared to dense grids. [S2, unofficial]
- **Measured Performance**:
    - Used in *Rise of Tomb Raider* (voxel AO) and *The Tomorrow Children* (on consoles). [S2, unofficial]
    - Sparse Voxel Octrees and voxel cascades (clever updating frequency changes) can improve framerates. [S2, unofficial]
- **Strengths**:
    - Significant reduction in draw calls. [S1, unofficial]
    - Reduced vertex transformation overhead for animated meshes. [S1, unofficial]
    - Efficient culling of empty columns in heightmap systems. [S3, unofficial]
- **Limitations**:
    - Dynamic lights intersecting static objects force re-voxelization. [S2, unofficial]
    - Heightmap removal is costly due to column scanning. [S3, unofficial]
    - Requires careful management of LOD and static/dynamic separation. [S1, unofficial]

## Background
Voxelization optimization techniques have been developed to address the performance bottlenecks of real-time voxel-based rendering, particularly in global illumination and fluid simulation. The use of stream output for caching transformed vertices and instancing for slice rendering was detailed in *GPU Gems 3* (Blythe 2006). [S1, unofficial] Further optimizations, such as Sparse Voxel Octrees and voxel cascades, have been applied in commercial titles like *Rise of Tomb Raider* and *The Tomorrow Children* to achieve real-time performance on consoles. [S2, unofficial] Heightmap-based optimizations for chunked voxel systems are documented in open-source implementations like *vercidium-patreon/meshing*. [S3, unofficial]

## Sources
[S1] PDF Real-Time Simulation and Rendering of 3D Fluids, https://wo1fsea.github.io/files/About_NavMesh/P5_GPU_Gems_3_Real-Time_Simulation_and_Rendering_of_3D_Fluids.pdf, fetched 2026-10-27 (unofficial)
[S2] Voxel-based Global Illumination – Wicked Engine, https://turanszkij.wordpress.com/2017/08/30/voxel-based-global-illumination/, fetched 2026-10-28 (unofficial)
[S3] vercidium-patreon/meshing, https://github.com/vercidium-patreon/meshing, fetched 2026-10-26 (unofficial)
