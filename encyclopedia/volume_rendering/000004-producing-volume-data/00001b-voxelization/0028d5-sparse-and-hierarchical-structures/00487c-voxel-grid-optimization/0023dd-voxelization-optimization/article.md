# Voxelization Optimization


## Overview
Voxelization Optimization encompasses techniques to minimize the computational and memory cost of converting continuous geometry or implicit fields into discrete voxel representations. It addresses the high draw-call overhead and redundant transformation costs inherent in rasterizing 3D scenes into voxel grids. It distinguishes itself from sibling methods like [Sparse Voxel Octrees](3687-sparse-voxelization-with-oracle-and-parallel-scan.md) by focusing on optimizing the rasterization pipeline and data access patterns for dense or semi-dense grids, often leveraging GPU hardware features directly.

## How it works
The method relies on reducing redundant calculations and draw calls through hardware-accelerated caching and instancing.

**Inputs:**
*   Geometry data (vertices, indices, transformation matrices).
*   Voxel grid parameters (resolution, bounds).
*   Scene state (static vs. dynamic objects).

**Outputs:**
*   Voxelized data structure (e.g., 3D texture, occupancy grid).

**Data Structures:**
*   **Transformed Vertex Buffer:** A buffer storing pre-computed vertex positions in world or voxel space, used to avoid per-slice transformation [S1, unofficial].
*   **Instance Data:** Structs containing slice-specific parameters (e.g., slice index, transformation offset) to support instanced rendering [S1, unofficial].

**Steps:**
1.  **Preprocessing Static Geometry:** Identify static objects. Do not re-voxelize them unless necessary. If an object undergoes a simple analytic transformation (e.g., rigid translation/rotation), precompute the voxelization and apply the inverse transformation during access [S2, unofficial].
2.  **Handling Dynamic/Deforming Geometry:** For deforming meshes (e.g., skinned characters), use **Stream Output** to cache transformed vertices. This avoids recomputing skinning transformations for every slice [S1, unofficial].
    *   *Step 2a:* Render the mesh with a geometry shader or compute shader that outputs transformed vertices to a buffer.
    *   *Step 2b:* Use this cached buffer for subsequent voxelization slices.
3.  **Instanced Slicing:** Use **Instanced Rendering** to draw all slices in a single draw call.
    *   *Step 3a:* Define an instance buffer where each instance corresponds to a slice (e.g., a specific Y-plane).
    *   *Step 3b:* Pass the instance ID to the shader to determine the target slice coordinate.
    *   *Step 3c:* The shader rasterizes the geometry onto the specific slice of the 3D texture [S1, unofficial].
4.  **Level of Detail (LOD) Simplification:** Use low-poly meshes for obstacles to speed up voxelization. Simplified models reduce the number of primitives processed per slice, allowing for per-frame voxelization of obstacles with minimal performance cost [S1, unofficial].
5.  **Dynamic Updates:** Re-voxelize dynamic objects and static objects intersected by dynamic lights. Dynamic lights may require re-voxelization of static objects to update lighting contributions correctly [S2, unofficial].

**Parameters:**
*   **LOD Level:** Determined by the trade-off between visual fidelity and performance. Not given in the sources.
*   **Slice Count:** Determined by the grid resolution in the slicing dimension.

**Edge Cases and Pitfalls:**
*   **Dynamic Lights:** Failing to re-voxelize static objects when intersected by dynamic lights leads to incorrect lighting [S2, unofficial].
*   **Complex Skinning:** Not using stream output for skinned meshes results in excessive CPU/GPU bandwidth usage for vertex transformations [S1, unofficial].
*   **High-Fidelity Meshes:** Using high-poly meshes for obstacles causes significant performance degradation during per-frame voxelization [S1, unofficial].

## Subtopics

### [32nd Bit Avoidance Modification](3351-32nd-bit-avoidance-modification.md)
The 32nd Bit Avoidance Modification adjusts the solid voxelization and density computation pipelines to exclude the highest bit of 32-bit storage formats, which is reserved for other purposes. It restricts voxel and density data to the lower bits, distinguishing it from general sparse voxelization by targeting bit-packing constraints in dense grids.

### [Sparse Voxelization with Oracle and Parallel Scan](3687-sparse-voxelization-with-oracle-and-parallel-scan.md)
Sparse Voxelization with Oracle and Parallel Scan is a real-time, high-resolution technique for reconstructing 3D surface models from image-based inputs (calibrated RGB camera streams) using a coarse-to-fine voxel hierarchy. It solves the problem of on-the-fly surface reconstruction by iteratively refining a sparse list of voxels, using an oracle kernel to cull empty space and a parallel prefix sum scan to compact the list, ensuring that only relevant voxels are subdivided. Unlike bottom-up approaches or methods designed for static models, this method is top-down and optimized for interactive rates (e.g., >30 fps at 1024^3 resolution) by leveraging GPU parallelism for both computation and rendering.

### [Greedy Voxel Mesh Optimization](3979-greedy-voxel-mesh-optimization.md)
Greedy Voxel Mesh Optimization is a greedy algorithm that optimizes voxel mesh generation by merging adjacent identical tiles into larger rectangular quads to reduce draw calls. It operates on 2D slices (e.g., the XZ plane) of solid voxels, differing from its parent [Voxelization Optimization](3191-voxelization-optimization.md) by focusing on the geometric merging phase after occlusion culling. It differs from [Disjoint Set Mesh Optimization](4495-disjoint-set-mesh-optimization.md) by using a simple iterative expansion rather than a Disjoint Set Union structure for the merging logic.

### Stream Output for Voxelization
Stream Output for Voxelization is a technique to optimize voxelization of deforming meshes by caching transformed vertices, avoiding recomputation for each slice [S1, unofficial]. It uses stream output to cache an entire buffer of transformed vertices during the voxelization of deforming meshes (e.g., skinned characters) [S1, unofficial].

### Min/Max Altitude Heightmap Optimization
Min/Max Altitude Heightmap Optimization stores per-column minimum and maximum occupied voxel heights to skip empty or fully occupied columns during meshing [S3, unofficial]. On voxel addition (index > 0), `OnVoxelAdded(i, j, k)` updates the heightmaps to reflect the new occupied column. On voxel removal (index == 0), `OnVoxelRemoved(i, j, k)` updates the heightmaps to reflect the cleared column. The `minAltitude` array is a byte array of size `ChunkSizeSquared` (1024 bytes), storing the lowest occupied Y index per XZ column. The `maxAltitude` array is a byte array of size `ChunkSizeSquared` (1024 bytes), storing the highest occupied Y index per XZ column. `minAltitude` is initialized to `Constants.ChunkSize` (32) for all columns, indicating initially empty columns [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the number of primitives and slices. Instancing reduces draw call overhead from $O(N_{slices})$ to $O(1)$ [S1, unofficial]. Stream output adds an initial cost for caching but reduces per-slice cost for deforming meshes [S1, unofficial].
*   **Memory Complexity:** Stream output requires additional buffer memory for transformed vertices [S1, unofficial]. Sparse Voxel Octrees can reduce memory requirements compared to dense grids [S2, unofficial].
*   **Measured Performance:** Techniques are used in games like *Rise of Tomb Raider* (voxel AO) and *The Tomorrow Children* (on consoles) [S2, unofficial]. Performance is highly dependent on GPU capabilities; aggressive optimization is required for real-time performance on fast GPUs [S2, unofficial].
*   **Strengths:** Significant reduction in draw calls and transformation overhead. Suitable for dynamic scenes with deforming geometry.
*   **Limitations:** Requires careful management of static vs. dynamic objects. Dynamic lights complicate static object caching. High-resolution grids still demand significant memory and bandwidth.
*   **When to Use:** Real-time rendering of dynamic scenes, fluid simulation visualization, voxel-based global illumination.
*   **When Not to Use:** Static scenes with no dynamic lighting interactions (precomputed voxelization is better). Scenes with extremely high-frequency geometry where LOD simplification causes unacceptable artifacts.

## Background
Voxelization optimization techniques have been developed to address the performance bottlenecks of early GPU-based voxelization methods. The use of stream output for deforming meshes and instancing for slicing was highlighted in *GPU Gems 3* [S1, unofficial]. The application of these techniques to global illumination, such as voxel cone tracing, was explored in the context of the Wicked Engine and games like *Rise of Tomb Raider* and *The Tomorrow Children* [S2, unofficial]. The Min/Max Altitude Heightmap Optimization is associated with specific meshing implementations like those in the `vercidium-patreon/meshing` repository [S3, unofficial].

## Sources
[S1] PDF Real-Time Simulation and Rendering of 3D Fluids, https://wo1fsea.github.io/files/About_NavMesh/P5_GPU_Gems_3_Real-Time_Simulation_and_Rendering_of_3D_Fluids.pdf, fetched 2026-10-27 (unofficial)
[S2] Voxel-based Global Illumination – Wicked Engine, https://turanszkij.wordpress.com/2017/08/30/voxel-based-global-illumination/, fetched 2026-10-28 (unofficial)
[S3] vercidium-patreon/meshing, https://github.com/vercidium-patreon/meshing, fetched 2026-10-26 (unofficial)
