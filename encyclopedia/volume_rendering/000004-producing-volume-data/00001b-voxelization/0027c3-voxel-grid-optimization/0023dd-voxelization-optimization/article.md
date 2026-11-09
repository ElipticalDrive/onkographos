# Voxelization Optimization


## Overview
Voxelization Optimization reduces the computational and memory costs of generating voxel grids from geometric primitives without altering the fundamental coordinate mapping defined in [Voxelization](12-voxelization.md) [S1, unofficial][S2, unofficial]. It distinguishes itself from sibling methods by focusing on pipeline efficiency (instancing, stream output) and spatial culling during the generation phase, rather than post-processing the grid data [S1, unofficial][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Geometric primitives (meshes, obstacles), camera/view parameters, and voxel grid configuration (resolution, bounds).
*   **Output:** A populated voxel grid (3D texture or structured array) and associated metadata (e.g., occupancy maps).

**Data Structures:**
*   **Voxel Grid:** A 3D array or texture where each cell represents a voxel. The layout is determined by [Voxelization](12-voxelization.md).
*   **Stream Output Buffer:** A GPU buffer storing transformed vertex positions, used to cache results of vertex shaders [S1, unofficial].
*   **Min/Max Altitude Heightmaps:** (Optional, for meshing optimization) Two byte arrays, `minAltitude` and `maxAltitude`, of size $W \times H$ (where $W, H$ are the XZ dimensions of the chunk/grid) [S3, unofficial].

**Steps:**
1.  **Preprocessing (Static vs. Dynamic):**
    *   Identify static objects. Do not re-voxelize them unless necessary [S2, unofficial].
    *   Identify dynamic objects or static objects intersected by dynamic lights. These must be re-voxelized [S2, unofficial].
    *   For obstacles, use a low level of detail (LOD) mesh to speed up the rasterization process [S1, unofficial].

2.  **Transformation Caching (Stream Output):**
    *   For deforming meshes (e.g., skinned characters), use stream output to cache the entire buffer of transformed vertices [S1, unofficial].
    *   This avoids recomputing vertex transformations for each slice or frame [S1, unofficial].
    *   *Pseudocode for Stream Output Setup:*
        ```python
        # Assuming a GPU context with stream output capabilities
        # 1. Configure the output stream to capture vertex shader outputs
        stream_output = create_stream_output_buffer(size=vertex_count * sizeof(Vertex))
        
        # 2. Bind the stream output target to the vertex shader output
        bind_stream_output_target(stream_output)
        
        # 3. Execute the voxelization draw call (e.g., using instancing)
        # The vertex shader transforms vertices; the rasterizer projects them to voxels
        draw_voxelization_call()
        
        # 4. Unbind stream output to read the cached vertices if needed for further processing
        unbind_stream_output_target()
        ```

3.  **Instanced Rendering:**
    *   Use instancing to draw all slices (or voxels) in a single draw call [S1, unofficial].
    *   Use the instance ID to specify the target slice or voxel coordinate [S1, unofficial].
    *   This reduces GPU command overhead compared to making a separate draw call for each slice [S1, unofficial].

4.  **Analytic Transformation Precomputation:**
    *   If an obstacle undergoes a simple analytic transformation (e.g., rigid translation/rotation), precompute the voxelization [S1, unofficial].
    *   Apply the inverse transformation when accessing the 3D textures during rendering [S1, unofficial].

5.  **Heightmap Maintenance (for Meshing Optimization):**
    *   Maintain `minAltitude` and `maxAltitude` arrays to track occupied columns [S3, unofficial].
    *   Initialize `minAltitude` to `Constants.ChunkSize` (e.g., 32) for all columns, indicating empty columns [S3, unofficial].
    *   On voxel addition (index > 0), update `minAltitude` and `maxAltitude` for the corresponding XZ column [S3, unofficial].
    *   On voxel removal (index == 0), update `minAltitude` and `maxAltitude` to reflect the new bounds or mark as empty [S3, unofficial].
    *   *Pseudocode for Heightmap Update:*
        ```python
        CHUNK_SIZE = 32
        CHUNK_SIZE_SQUARED = CHUNK_SIZE * CHUNK_SIZE
        
        # Initialize heightmaps
        min_altitude = [CHUNK_SIZE] * CHUNK_SIZE_SQUARED
        max_altitude = [-1] * CHUNK_SIZE_SQUARED
        
        def on_voxel_added(x, y, z):
            col_index = x + z * CHUNK_SIZE
            if y < min_altitude[col_index]:
                min_altitude[col_index] = y
            if y > max_altitude[col_index]:
                max_altitude[col_index] = y
                
        def on_voxel_removed(x, y, z):
            col_index = x + z * CHUNK_SIZE
            # If the removed voxel was the min or max, we may need to scan the column
            # to find the new min/max, or mark as empty if the column is now empty.
            # For simplicity, if it was the max, set max to -1 (empty) or recompute.
            if y == max_altitude[col_index]:
                # Recompute max for this column or set to -1 if empty
                max_altitude[col_index] = recompute_max_for_column(x, z)
            if y == min_altitude[col_index]:
                # Recompute min for this column or set to CHUNK_SIZE if empty
                min_altitude[col_index] = recompute_min_for_column(x, z)
                
        def recompute_max_for_column(x, z):
            # Scan the column to find the highest occupied voxel
            # This is a fallback if the removed voxel was the max
            max_y = -1
            for y in range(CHUNK_SIZE):
                if is_voxel_occupied(x, y, z):
                    max_y = y
            return max_y
            
        def recompute_min_for_column(x, z):
            # Scan the column to find the lowest occupied voxel
            min_y = CHUNK_SIZE
            for y in range(CHUNK_SIZE):
                if is_voxel_occupied(x, y, z):
                    min_y = y
                    break
            return min_y
        ```

**Parameters:**
*   **LOD Level:** The level of detail for obstacle meshes. Lower LOD reduces voxelization cost [S1, unofficial].
*   **Chunk Size:** Typically 32 voxels per side for heightmap arrays [S3, unofficial].

**Edge Cases and Pitfalls:**
*   **Dynamic Lights:** Dynamic lights intersecting static objects force re-voxelization of those static objects [S2, unofficial].
*   **Precision:** Stream output caching requires sufficient buffer size to hold all transformed vertices [S1, unofficial].
*   **Empty Columns:** Heightmap optimization assumes columns can be efficiently scanned or tracked; sparse columns may require fallback scanning [S3, unofficial].

## Subtopics

### [32nd Bit Avoidance Modification](3351-32nd-bit-avoidance-modification.md)
The 32nd Bit Avoidance Modification adjusts the solid voxelization and density computation pipelines to exclude the highest bit of 32-bit storage formats, which is reserved for other purposes. It restricts voxel and density data to the lower bits, distinguishing it from general sparse voxelization by targeting bit-packing constraints in dense grids.

### [Sparse Voxelization with Oracle and Parallel Scan](3687-sparse-voxelization-with-oracle-and-parallel-scan.md)
Sparse Voxelization with Oracle and Parallel Scan is a real-time, high-resolution technique for reconstructing 3D surface models from image-based inputs (calibrated RGB camera streams) using a coarse-to-fine voxel hierarchy. It solves the problem of on-the-fly surface reconstruction by iteratively refining a sparse list of voxels, using an oracle kernel to cull empty space and a parallel prefix sum scan to compact the list, ensuring that only relevant voxels are subdivided. Unlike bottom-up approaches or methods designed for static models, this method is top-down and optimized for interactive rates (e.g., >30 fps at 1024^3 resolution) by leveraging GPU parallelism for both computation and rendering.

### [Greedy Voxel Mesh Optimization](3979-greedy-voxel-mesh-optimization.md)
Greedy Voxel Mesh Optimization is a greedy algorithm that optimizes voxel mesh generation for 2D tile grids (e.g., Minecraft-like games) by merging adjacent identical tiles into larger rectangular quads to reduce draw calls. It differs from its siblings, [32nd Bit Avoidance Modification](3351-32nd-bit-avoidance-modification.md) and [Sparse Voxelization with Oracle and Parallel Scan](3687-sparse-voxelization-with-parallel-scan.md), by explicitly constructing maximal rectangular regions of uniform tile properties through iterative expansion, rather than relying on bit-packing tricks or parallel scan-based sparse structures.

### Stream Output for Voxelization
A technique to optimize voxelization of deforming meshes by caching transformed vertices, avoiding recomputation for each slice [S1, unofficial]. It uses stream output to cache an entire buffer of transformed vertices during the voxelization of deforming meshes (e.g., skinned characters) [S1, unofficial].

### Min/Max Altitude Heightmap Optimization
Stores per-column minimum and maximum occupied voxel heights to skip empty or fully occupied columns during meshing [S3, unofficial]. On voxel addition (index > 0), `OnVoxelAdded(i, j, k)` updates the heightmaps to reflect the new occupied column [S3, unofficial]. On voxel removal (index == 0), `OnVoxelRemoved(i, j, k)` updates the heightmaps to reflect the cleared column [S3, unofficial]. Initialize `minAltitude` array to `Constants.ChunkSize` (32) for all columns, indicating initially empty columns [S3, unofficial]. Parameters include `minAltitude` (byte array of size `ChunkSizeSquared` (1024 bytes), stores the lowest occupied Y index per XZ column) and `maxAltitude` (byte array of size `ChunkSizeSquared` (1024 bytes), stores the highest occupied Y index per XZ column) [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the number of primitives and voxels. Instancing reduces draw call overhead to $O(1)$ per frame regardless of slice count [S1, unofficial]. Stream output reduces vertex transformation cost to $O(V)$ per frame (cached) instead of $O(V \times S)$ where $S$ is the number of slices [S1, unofficial].
*   **Memory Complexity:** Stream output requires $O(V)$ memory for the vertex buffer [S1, unofficial]. Heightmap optimization requires $O(W \times H)$ memory for the altitude arrays [S3, unofficial].
*   **Measured Performance:** Techniques are used in games like *Rise of Tomb Raider* (voxel AO) and *The Tomorrow Children* (on consoles) [S2, unofficial]. Performance is highly dependent on GPU capability; aggressive optimization is required for real-time performance [S2, unofficial].
*   **Strengths:** Significant reduction in CPU/GPU overhead for dynamic scenes; efficient handling of deforming meshes; reduced memory footprint for meshing via heightmaps.
*   **Limitations:** Requires careful management of static vs. dynamic objects; stream output may have latency; heightmap optimization is specific to certain meshing algorithms.
*   **When to Use:** Real-time applications with dynamic geometry or frequent updates; applications where draw call overhead is a bottleneck.
*   **When Not to Use:** Static scenes with no dynamic lights; scenes where memory bandwidth is the primary bottleneck and stream output caching is not feasible.

## Background
Voxelization optimization techniques have been developed to address the high computational cost of rasterizing geometry into voxel grids for real-time applications. Stream output and instancing were introduced as standard GPU optimization techniques [S1, unofficial]. The use of voxelization for global illumination and ambient occlusion in games like *Rise of Tomb Raider* and *The Tomorrow Children* demonstrated the viability of these optimizations on consumer hardware [S2, unofficial]. Heightmap optimization for voxel meshing is a more specialized technique for efficient surface extraction from voxel data [S3, unofficial].

## Sources
[S1] PDF Real-Time Simulation and Rendering of 3D Fluids, https://wo1fsea.github.io/files/About_NavMesh/P5_GPU_Gems_3_Real-Time_Simulation_and_Rendering_of_3D_Fluids.pdf, fetched 2026-10-27 (unofficial)
[S2] Voxel-based Global Illumination – Wicked Engine, https://turanszkij.wordpress.com/2017/08/30/voxel-based-global-illumination/, fetched 2026-10-28 (unofficial)
[S3] vercidium-patreon/meshing, https://github.com/vercidium-patreon/meshing, fetched 2026-10-26 (unofficial)
