# Voxelization Optimization


## Overview
Voxelization Optimization encompasses techniques to reduce the computational and memory overhead of converting geometry into voxel grids, addressing the high cost of numerous draw calls and redundant vertex transformations. It distinguishes itself from general grid management methods by focusing on the rendering pipeline efficiency (instancing, stream output) and spatial culling during the conversion phase, rather than post-processing the resulting data structure.

## How it works
The method comprises several distinct strategies applied during or immediately after the voxelization process.

### Inputs and Outputs
*   **Inputs**: Source geometry (static meshes, deforming/skinned meshes, obstacles), camera/view parameters, voxel grid resolution, and transformation matrices.
*   **Outputs**: Optimized voxel grid data, reduced number of draw calls, cached intermediate vertex data.

### Data Structures
*   **Stream Output Buffer**: A GPU buffer storing transformed vertex positions and attributes, indexed by instance or slice ID [S1, unofficial].
*   **Min/Max Altitude Maps**: Two byte arrays (`minAltitude`, `maxAltitude`) of size $W \times H$ (where $W, H$ are chunk dimensions), storing the lowest and highest occupied Y-coordinates for each XZ column [S3, unofficial].

### Steps

#### 1. Stream Output Caching for Deforming Meshes
For deforming meshes (e.g., skinned characters), vertex transformations are expensive. Recomputing them for every voxel slice is inefficient.
1.  Render the mesh to a stream output buffer, transforming vertices once per frame into world or voxel space [S1, unofficial].
2.  Use this cached buffer for subsequent voxelization slices, avoiding per-slice CPU-to-GPU vertex upload or per-slice vertex shader computation [S1, unofficial].

#### 2. Instanced Slicing
Instead of issuing one draw call per voxel slice (or layer):
1.  Prepare a single draw call using instanced rendering [S1, unofficial].
2.  Pass the slice index (or transformation matrix for the slice) as an instance attribute [S1, unofficial].
3.  The vertex/geometry shader uses the instance ID to determine the current slice's position or transformation [S1, unofficial].

#### 3. Level of Detail (LOD) Simplification
To speed up the rasterization of obstacles:
1.  Replace high-polygon obstacle meshes with low-polygon approximations [S1, unofficial].
2.  Use these simplified meshes for voxelization to reduce fill-rate and vertex processing costs, accepting minor geometric inaccuracies [S1, unofficial].

#### 4. Static Geometry Caching
1.  Identify static geometry that does not change position or shape [S2, unofficial].
2.  Voxelize static objects once and store the result [S2, unofficial].
3.  Skip re-voxelization of static objects unless a dynamic element (e.g., a light source) intersects their bounding volume, which may require partial or full re-voxelization to update lighting contributions [S2, unofficial].

#### 5. Analytic Transformation Inversion
For objects with simple analytic transformations (rigid body motion):
1.  Precompute the voxelization of the object in its local space [S1, unofficial].
2.  When accessing the voxel data, apply the inverse transformation to the query coordinates rather than transforming the geometry [S1, unofficial].

#### 6. Min/Max Altitude Heightmap Optimization (Meshing Phase)
When generating meshes from voxel data (e.g., Marching Cubes), skip empty or fully occupied columns:
1.  Initialize `minAltitude` array to $ChunkSize$ (e.g., 32) and `maxAltitude` to -1 for all columns [S3, unofficial].
2.  On voxel addition at $(x, y, z)$:
    *   Update `minAltitude[x][z] = min(minAltitude[x][z], y)` [S3, unofficial].
    *   Update `maxAltitude[x][z] = max(maxAltitude[x][z], y)` [S3, unofficial].
3.  On voxel removal at $(x, y, z)$:
    *   If $y == minAltitude[x][z]$, scan the column to find the new minimum and update `minAltitude[x][z]` [S3, unofficial].
    *   If $y == maxAltitude[x][z]$, scan the column to find the new maximum and update `maxAltitude[x][z]` [S3, unofficial].
4.  During meshing, skip columns where `minAltitude[x][z] > maxAltitude[x][z]` (empty) or where the column is fully occupied (if applicable) [S3, unofficial].

### Parameters
*   **LOD Level**: Determined by the trade-off between geometric accuracy and performance. No specific polygon count is mandated, but "low level of detail" is recommended for obstacles [S1, unofficial].
*   **Chunk Size**: Commonly 32x32x32 or similar powers of two for heightmap optimization [S3, unofficial].

### Edge Cases and Pitfalls
*   **Dynamic Lights**: Dynamic lights intersecting static geometry may require re-voxelization of the static object's contribution, complicating the "static cache" assumption [S2, unofficial].
*   **Precision**: Applying inverse transformations to query coordinates may introduce floating-point precision errors compared to pre-transforming geometry [S1, unofficial].
*   **Heightmap Updates**: Updating min/max altitude maps on voxel removal requires scanning the column, which can be costly if done frequently. [S3, unofficial].

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
Stores per-column minimum and maximum occupied voxel heights to skip empty or fully occupied columns during meshing [S3, unofficial]. On voxel addition (index > 0), `OnVoxelAdded(i, j, k)` updates the heightmaps to reflect the new occupied column. On voxel removal (index == 0), `OnVoxelRemoved(i, j, k)` updates the heightmaps to reflect the cleared column. The `minAltitude` array is a byte array of size `ChunkSizeSquared` (1024 bytes), storing the lowest occupied Y index per XZ column. The `maxAltitude` array is a byte array of size `ChunkSizeSquared` (1024 bytes), storing the highest occupied Y index per XZ column. The `minAltitude` array is initialized to `Constants.ChunkSize` (32) for all columns, indicating initially empty columns [S3, unofficial].

## Operating characteristics
*   **Time Complexity**: Instancing reduces draw call overhead from $O(N)$ to $O(1)$ where $N$ is the number of slices [S1, unofficial]. Stream output reduces vertex transformation cost from $O(N \cdot V)$ to $O(V)$ per frame for deforming meshes, where $V$ is vertex count [S1, unofficial].
*   **Memory Complexity**: Stream output buffers require additional GPU memory proportional to the vertex count [S1, unofficial]. Min/Max altitude maps add $O(W \cdot H)$ memory per chunk [S3, unofficial].
*   **Performance**: Techniques allow real-time voxelization on fast GPUs, used in titles like *Rise of Tomb Raider* (voxel AO) and *The Tomorrow Children* [S2, unofficial].
*   **Strengths**: Significant reduction in CPU-GPU bandwidth and draw call overhead; efficient handling of dynamic geometry.
*   **Limitations**: Stream output requires GPU support; LOD simplification may introduce visual artifacts; static caching is complex to manage with dynamic lighting [S1, unofficial][S2, unofficial].

## Background
Voxelization optimization techniques have been developed to address the high computational cost of real-time voxelization, particularly for fluid simulation and global illumination. Stream output and instancing were highlighted in *GPU Gems 3* for real-time fluid simulation [S1, unofficial]. The use of voxel cascades and sparse voxel octrees for optimization was discussed in the context of voxel-based global illumination by Turanszkij [S2, unofficial]. Heightmap optimizations for voxel meshing are found in open-source implementations like `vercidium-patreon/meshing` [S3, unofficial].

## Sources
[S1] Real-Time Simulation and Rendering of 3D Fluids, https://wo1fsea.github.io/files/About_NavMesh/P5_GPU_Gems_3_Real-Time_Simulation_and_Rendering_of_3D_Fluids.pdf, fetched 2026-10-27 (unofficial)
[S2] Voxel-based Global Illumination – Wicked Engine, https://turanszkij.wordpress.com/2017/08/30/voxel-based-global-illumination/, fetched 2026-10-28 (unofficial)
[S3] vercidium-patreon/meshing, https://github.com/vercidium-patreon/meshing, fetched 2026-10-26 (unofficial)
