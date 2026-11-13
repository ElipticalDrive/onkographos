# Voxelization Optimization


## Overview
Voxelization Optimization encompasses techniques to accelerate the conversion of geometric primitives into voxel grids, addressing the high computational cost of generating large 3D textures for volume rendering and global illumination. It distinguishes itself from general sparse voxel structures by focusing on the efficiency of the rasterization and data generation pipeline itself, leveraging GPU hardware features like stream output and instancing to reduce draw calls and vertex recomputation [S1, unofficial][S2, unofficial].

## How it works
The method optimizes the voxelization pass through four primary mechanisms: stream output caching, instanced rendering, level-of-detail (LOD) simplification, and static/dynamic separation.

**Inputs and Outputs:**
*   **Input:** 3D scene geometry (static and dynamic meshes), camera parameters, voxel grid resolution, and slice configuration.
*   **Output:** A populated 3D voxel texture (or sparse data structure) representing occupancy, density, or material properties.

**Data Structures:**
*   **Stream Output Buffer:** A GPU buffer storing transformed vertex positions and attributes, indexed by instance or primitive ID.
*   **Instance Buffer:** Contains per-slice transformation matrices or offsets, allowing a single draw call to populate multiple slices of the 3D texture.
*   **Min/Max Altitude Maps (Optional):** For meshing optimizations, byte arrays storing the minimum and maximum occupied Y-coordinates per XZ column [S3, unofficial].

**Steps:**

1.  **Preprocessing Static Geometry:**
    *   Identify static objects in the scene.
    *   Voxelize static objects once into the initial grid state [S2, unofficial].
    *   *Note:* If a dynamic light intersects a static object, the static object may need to be re-voxelized or updated to reflect lighting changes, depending on the specific illumination algorithm [S2, unofficial].

2.  **Handling Deforming/Dynamic Geometry:**
    *   For deforming meshes (e.g., skinned characters), avoid recomputing vertex transformations for every voxel slice.
    *   Use **Stream Output** to cache the transformed vertices of the mesh into a buffer after the initial vertex shader execution [S1, unofficial].
    *   Subsequent slices read from this cached buffer rather than re-invoking the vertex shader for the same geometry [S1, unofficial].

3.  **Instanced Slice Rendering:**
    *   Instead of issuing one draw call per voxel slice (layer), use **Instanced Rendering**.
    *   Pass an instance buffer where each instance corresponds to a specific slice (Z-coordinate or layer index) [S1, unofficial].
    *   Use the `InstanceID` in the geometry or fragment shader to determine the target slice coordinate, writing directly to the appropriate layer of the 3D texture [S1, unofficial].

4.  **Level of Detail (LOD) Simplification:**
    *   Replace high-polygon obstacle meshes with low-polygon approximations for the voxelization pass [S1, unofficial].
    *   This reduces the number of primitives processed during rasterization, significantly lowering the cost of voxelizing obstacles at every frame [S1, unofficial].

5.  **Analytic Transformation Inversion (Optional):**
    *   For objects with simple analytic transformations (rigid body translation/rotation), precompute the voxelization.
    *   Apply the inverse transformation to the texture coordinates during access rather than re-voxelizing the geometry [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxelize_scene(scene, grid_resolution, slice_count):
    # 1. Initialize Grid
    voxel_grid = allocate_3d_texture(grid_resolution, slice_count)
    
    # 2. Process Static Objects
    for obj in scene.static_objects:
        voxelize_mesh(obj.mesh, voxel_grid)
        
    # 3. Process Dynamic/Deforming Objects using Instancing and Stream Output
    # Create instance buffer where each instance represents a slice
    instance_buffer = create_instance_buffer(slice_count)
    
    # Cache transformed vertices using Stream Output
    # This avoids recomputing skinning/transformations for each slice
    cached_vertices = stream_output_cache(scene.dynamic_mesh)
    
    # Single draw call for all slices
    for slice_idx in range(slice_count):
        # Set instance ID to current slice
        set_instance_id(slice_idx)
        
        # Render using cached vertices and instance buffer
        # The shader uses instance ID to write to the correct Z-layer
        render_slice(cached_vertices, instance_buffer, slice_idx, voxel_grid)

def voxelize_mesh(mesh, grid):
    # Use LOD mesh if available to speed up rasterization
    lod_mesh = get_lowest_detail_mesh(mesh)
    rasterize_to_grid(lod_mesh, grid)
```

**Parameters:**
*   **LOD Level:** The simplification factor for obstacle meshes. Not given in the sources, but typically chosen to balance visual fidelity of the voxel boundary against rasterization speed [S1, unofficial].
*   **Chunk Size:** For altitude map optimizations, a fixed size (e.g., 32x32) defines the column resolution [S3, unofficial].

**Edge Cases and Pitfalls:**
*   **Dynamic Lights on Static Objects:** Static objects must be re-voxelized if dynamic lights intersect them, which can be difficult to detect efficiently [S2, unofficial].
*   **Memory Bandwidth:** Caching transformed vertices requires sufficient VRAM for the stream output buffer [S1, unofficial].
*   **Precision:** Using simplified LOD meshes may introduce aliasing or jagged edges in the voxel boundary [S1, unofficial].

## Subtopics

### [32nd Bit Avoidance Modification](3351-32nd-bit-avoidance-modification.md)
The 32nd Bit Avoidance Modification adjusts the solid voxelization and density computation pipelines to exclude the highest bit of 32-bit storage formats, which is reserved for other purposes. It restricts voxel and density data to the lower bits, distinguishing it from general sparse voxelization by targeting bit-packing constraints in dense grids.

### [Sparse Voxelization with Oracle and Parallel Scan](3687-sparse-voxelization-with-oracle-and-parallel-scan.md)
Sparse Voxelization with Oracle and Parallel Scan is a real-time, high-resolution technique for reconstructing 3D surface models from image-based inputs (calibrated RGB camera streams) using a coarse-to-fine voxel hierarchy. It solves the problem of on-the-fly surface reconstruction by iteratively refining a sparse list of voxels, using an oracle kernel to cull empty space and a parallel prefix sum scan to compact the list, ensuring that only relevant voxels are subdivided. Unlike bottom-up approaches or methods designed for static models, this method is top-down and optimized for interactive rates (e.g., >30 fps at 1024^3 resolution) by leveraging GPU parallelism for both computation and rendering.

### [Greedy Voxel Mesh Optimization](3979-greedy-voxel-mesh-optimization.md)
Greedy Voxel Mesh Optimization is a greedy algorithm that optimizes voxel mesh generation by merging adjacent identical tiles into larger rectangular quads to reduce draw calls. It operates on 2D slices (e.g., the XZ plane) of solid voxels, differing from its parent [Voxelization Optimization](3191-voxelization-optimization.md) by focusing on the geometric merging phase after occlusion culling. It differs from [Disjoint Set Mesh Optimization](4495-disjoint-set-mesh-optimization.md) by using a simple iterative expansion rather than a Disjoint Set Union structure for the merging logic.

### Stream Output for Voxelization
A technique to optimize voxelization of deforming meshes by caching transformed vertices, avoiding recomputation for each slice [S1, unofficial]. It uses stream output to cache an entire buffer of transformed vertices during the voxelization of deforming meshes (e.g., skinned characters) [S1, unofficial].

### Min/Max Altitude Heightmap Optimization
Stores per-column minimum and maximum occupied voxel heights to skip empty or fully occupied columns during meshing [S3, unofficial]. On voxel addition (index > 0), `OnVoxelAdded(i, j, k)` updates the heightmaps to reflect the new occupied column. On voxel removal (index == 0), `OnVoxelRemoved(i, j, k)` updates the heightmaps to reflect the cleared column. Initialize `minAltitude` array to `Constants.ChunkSize` (32) for all columns, indicating initially empty columns. Parameters include `minAltitude`: a byte array of size `ChunkSizeSquared` (1024 bytes) storing the lowest occupied Y index per XZ column, and `maxAltitude`: a byte array of size `ChunkSizeSquared` (1024 bytes) storing the highest occupied Y index per XZ column [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Reduces the complexity of dynamic voxelization from $O(N \cdot S)$ (where $N$ is primitives and $S$ is slices) to $O(N + S)$ by caching transformations and using instancing [S1, unofficial].
*   **Memory Complexity:** Stream output buffers increase VRAM usage proportional to the vertex count of dynamic meshes [S1, unofficial]. Sparse Voxel Octrees can be used to reduce memory requirements for the resulting grid [S2, unofficial].
*   **Measured Performance:** Techniques are used in commercial titles like *Rise of Tomb Raider* (voxel AO) and *The Tomorrow Children* (consoles), indicating viability on high-end GPUs and constrained hardware respectively [S2, unofficial].
*   **Strengths:** Significant reduction in draw calls and vertex shader invocations for dynamic scenes; efficient handling of deforming geometry.
*   **Limitations:** Requires careful management of static/dynamic separation; LOD simplification may reduce geometric accuracy; complex skinning operations may still be costly if stream output buffers are not managed efficiently [S1, unofficial][S2, unofficial].
*   **When to Use:** Real-time volume rendering or global illumination with dynamic/deforming geometry.
*   **When Not to Use:** Static scenes with no dynamic lights or simple transformations, where precomputed voxelization or analytic inversion is sufficient [S1, unofficial][S2, unofficial].

## Background
Voxelization optimization techniques have been developed to address the performance bottlenecks of GPU-based voxelization, particularly for real-time applications like global illumination and fluid simulation. Key implementations include voxel AO in *Rise of Tomb Raider* and voxel cone tracing in *The Tomorrow Children* [S2, unofficial]. The use of stream output for caching transformed vertices was highlighted in *GPU Gems 3* for real-time fluid simulation [S1, unofficial].

## Sources
[S1] PDF Real-Time Simulation and Rendering of 3D Fluids, https://wo1fsea.github.io/files/About_NavMesh/P5_GPU_Gems_3_Real-Time_Simulation_and_Rendering_of_3D_Fluids.pdf, fetched 2026-10-27 (unofficial)
[S2] Voxel-based Global Illumination – Wicked Engine, https://turanszkij.wordpress.com/2017/08/30/voxel-based-global-illumination/, fetched 2026-10-28 (unofficial)
[S3] vercidium-patreon/meshing, https://github.com/vercidium-patreon/meshing, fetched 2026-10-26 (unofficial)
