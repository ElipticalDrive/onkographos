# Voxelization Optimization


## Overview
Voxelization Optimization encompasses techniques to reduce the computational and memory costs of converting geometric primitives into a discrete 3D voxel grid, addressing the high draw call count and vertex transformation overhead inherent in GPU-based voxelization. It distinguishes itself from downstream processing optimizations (such as [3D Voxel Grid Smoothing](833-3d-voxel-grid-smoothing.md) or [Binary Voxel RLE Encoding](1071-binary-voxel-rle-encoding.md)) by focusing on the input pipeline: minimizing redundant vertex transformations, reducing the number of draw calls via instancing, and optimizing the geometry used for the voxelization pass itself.

## How it works
The method operates on the geometric primitives (meshes) before they are rasterized into the voxel grid. It assumes a valid, populated 3D array or structured representation of voxel data as input [G].

### Inputs and Outputs
*   **Input**: Source geometry (static or deforming meshes), camera/view parameters, voxel grid resolution.
*   **Output**: A populated voxel grid (occupancy, density, or attributes) with reduced generation latency.

### Data Structures
*   **Transformed Vertex Buffer**: A GPU buffer storing the final world-space positions of vertices, used to cache results of skinning or transformation operations [S1, unofficial].
*   **Instance Data**: Per-slice transformation matrices or offsets, allowing a single draw call to render multiple slices [S1, unofficial].
*   **Min/Max Altitude Heightmaps**: Two byte arrays (`minAltitude`, `maxAltitude`) of size $N^2$ (where $N$ is the chunk dimension, e.g., 32), storing the lowest and highest occupied Y-index for each XZ column in a chunk [S3, unofficial].

### Steps
1.  **Geometry Simplification**: Replace high-poly obstacle meshes with low-level-of-detail (LOD) meshes for the voxelization pass. This reduces the number of vertices processed per slice, allowing for per-frame voxelization of dynamic obstacles with minimal performance cost [S1, unofficial].
2.  **Vertex Transformation Caching (Stream Output)**: For deforming meshes (e.g., skinned characters), use GPU stream output to cache the transformed vertices in a buffer. This avoids recomputing skinning transformations for every slice of the voxel grid [S1, unofficial].
3.  **Instanced Slicing**: Instead of issuing a separate draw call for each voxel slice (plane), use instanced rendering. Pass the slice index or transformation matrix as instance data. The vertex shader uses the instance ID to compute the correct slice position, drawing all slices in a single draw call [S1, unofficial].
4.  **Static Object Culling**: Do not re-voxelize static objects unless necessary. Static geometry can be voxelized once and stored. However, if dynamic lights or other dynamic entities intersect the static object's bounding volume, the static object must be re-voxelized to account for the interaction [S2, unofficial].
5.  **Analytic Transformation Inversion**: If an obstacle undergoes a simple analytic transformation (e.g., rigid translation/rotation), precompute the voxelization and apply the inverse transformation during texture access, rather than re-voxelizing the geometry [S1, unofficial].
6.  **Heightmap Maintenance (Chunk-Level)**: For chunk-based voxel grids, maintain `minAltitude` and `maxAltitude` arrays to quickly identify empty or fully occupied columns during subsequent meshing or culling phases [S3, unofficial].
    *   **Initialization**: Set `minAltitude[i] = ChunkSize` and `maxAltitude[i] = -1` (or 0) for all columns $i$ in the chunk [S3, unofficial].
    *   **On Voxel Addition**: When a voxel at $(x, y, z)$ is added (index > 0), update `minAltitude[xz]` and `maxAltitude[xz]` if $y$ is outside the current range [S3, unofficial].
    *   **On Voxel Removal**: When a voxel at $(x, y, z)$ is removed (index == 0), update `minAltitude[xz]` and `maxAltitude[xz]` by scanning the column to find the new bounds [S3, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxelize_deforming_mesh(mesh, slices, stream_output_buffer):
    """
    Optimized voxelization for deforming meshes using stream output and instancing.
    """
    # Step 1: Cache transformed vertices using stream output
    # If the buffer is empty or invalid, compute skinning once
    if not stream_output_buffer.is_valid():
        transformed_vertices = compute_skinning(mesh)
        stream_output_buffer.write(transformed_vertices)
    
    # Step 2: Use instanced rendering to draw all slices
    # Pass slice index as instance ID
    draw_call_count = 1
    for slice_idx in range(len(slices)):
        # The vertex shader uses instance ID to determine slice position
        render_slice(stream_output_buffer, slice_idx)
        
    return draw_call_count

def update_heightmaps(chunk, x, y, z, action):
    """
    Maintains min/max altitude heightmaps for a chunk.
    """
    col_idx = x * chunk.size + z
    
    if action == "ADD":
        if y < chunk.min_altitude[col_idx]:
            chunk.min_altitude[col_idx] = y
        if y > chunk.max_altitude[col_idx]:
            chunk.max_altitude[col_idx] = y
            
    elif action == "REMOVE":
        # Re-scan column to find new bounds
        new_min = chunk.size
        new_max = -1
        for yy in range(chunk.size):
            if chunk.get_voxel(x, yy, z) is not None:
                if yy < new_min: new_min = yy
                if yy > new_max: new_max = yy
        
        chunk.min_altitude[col_idx] = new_min
        chunk.max_altitude[col_idx] = new_max
```

### Parameters
*   **LOD Level**: The simplification factor for obstacle meshes. Not given in the sources.
*   **Chunk Size**: Typically 32x32x32 for heightmap arrays, resulting in 1024 bytes per heightmap array [S3, unofficial].

### Edge Cases and Pitfalls
*   **Dynamic Lights**: Dynamic lights intersecting static objects force re-voxelization of the static object. Failing to detect this intersection leads to incorrect lighting in the voxel grid [S2, unofficial].
*   **Stream Output Latency**: Caching transformed vertices requires synchronization between the CPU and GPU. If the stream output buffer is not properly managed, it may stall the pipeline [S1, unofficial].
*   **Heightmap Updates**: Removing a voxel may require a full column scan to update the heightmaps, which can be costly if done frequently. This is acceptable for sparse updates but may bottleneck dense voxel changes [S3, unofficial].

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
Stores per-column minimum and maximum occupied voxel heights to skip empty or fully occupied columns during meshing [S3, unofficial]. On voxel addition (index > 0), `OnVoxelAdded(i, j, k)` updates the heightmaps to reflect the new occupied column. On voxel removal (index == 0), `OnVoxelRemoved(i, j, k)` updates the heightmaps to reflect the cleared column. The `minAltitude` array is initialized to `Constants.ChunkSize` (32) for all columns, indicating initially empty columns. The `minAltitude` and `maxAltitude` are byte arrays of size `ChunkSizeSquared` (1024 bytes), storing the lowest and highest occupied Y index per XZ column, respectively [S3, unofficial].

## Operating characteristics
*   **Time Complexity**: Reduces the number of draw calls from $O(S)$ (where $S$ is the number of slices) to $O(1)$ per mesh using instancing [S1, unofficial]. Vertex transformation cost is reduced from $O(S \cdot V)$ to $O(V)$ per frame for deforming meshes, where $V$ is the number of vertices [S1, unofficial].
*   **Memory Complexity**: Stream output buffers require additional GPU memory to store transformed vertices [S1, unofficial]. Heightmap arrays add $2 \cdot N^2$ bytes per chunk [S3, unofficial].
*   **Measured Performance**: Techniques are used in commercial titles such as *Rise of Tomb Raider* (voxel AO) and *The Tomorrow Children* (on consoles), demonstrating viability on high-end GPUs and consoles [S2, unofficial].
*   **Strengths**: Significant reduction in CPU-GPU overhead for dynamic scenes; enables real-time voxelization of complex deforming meshes.
*   **Limitations**: Requires GPU support for stream output and instancing; static object re-voxelization logic can be complex to implement correctly [S2, unofficial].
*   **When to Use**: Real-time applications with deforming geometry or high-frequency updates.
*   **When Not to Use**: Static scenes with no dynamic lights or interactions, where pre-computed voxelization is sufficient.

## Background
Voxelization optimization techniques have been developed to address the performance bottlenecks of GPU-based voxelization, particularly in real-time rendering applications. The use of stream output for caching transformed vertices and instancing for slice rendering was detailed in *GPU Gems 3* [S1, unofficial]. Further optimizations, such as sparse voxel octrees and voxel cascades, were discussed in the context of voxel-based global illumination in the *Wicked Engine* project [S2, unofficial]. Heightmap-based optimizations for chunked voxel grids are documented in the *vercidium-patreon/meshing* repository [S3, unofficial].

## Sources
[S1] Real-Time Simulation and Rendering of 3D Fluids, https://wo1fsea.github.io/files/About_NavMesh/P5_GPU_Gems_3_Real-Time_Simulation_and_Rendering_of_3D_Fluids.pdf, fetched 2026-10-27 (unofficial)
[S2] Voxel-based Global Illumination – Wicked Engine, https://turanszkij.wordpress.com/2017/08/30/voxel-based-global-illumination/, fetched 2026-10-28 (unofficial)
[S3] vercidium-patreon/meshing, https://github.com/vercidium-patreon/meshing, fetched 2026-10-26 (unofficial)
