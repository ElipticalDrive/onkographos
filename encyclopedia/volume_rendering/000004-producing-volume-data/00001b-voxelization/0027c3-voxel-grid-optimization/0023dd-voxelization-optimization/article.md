# Voxelization Optimization


## Overview
Voxelization Optimization encompasses techniques to reduce the computational cost and memory bandwidth of generating voxel grids from geometric primitives, particularly in real-time rendering pipelines. It addresses the bottleneck of performing numerous draw calls and vertex transformations required to populate a 3D texture or volume buffer. Unlike downstream optimizations that compress or smooth the resulting data, these methods focus on the acquisition phase, leveraging GPU features like stream output and instancing to minimize redundant calculations [S1, unofficial].

## How it works
The core strategy involves reducing the number of vertex shader invocations and draw calls during the rasterization of geometry into voxel space.

### Inputs and Outputs
- **Input**: 3D meshes (static or deforming/skinned), voxel grid dimensions (resolution), camera/view parameters.
- **Output**: A populated voxel grid (3D texture or structured buffer) containing occupancy, density, or normal data.

### Data Structures
- **Transformed Vertex Buffer**: A GPU buffer storing the final world-space positions of vertices after skinning/transformation. This buffer is reused across multiple slice passes [S1, unofficial].
- **Instance Data**: A buffer or constant buffer containing slice-specific parameters (e.g., slice index, transformation matrices) to support instanced rendering [S1, unofficial].

### Steps
1.  **Mesh Simplification**: Replace high-polygon obstacle meshes with low-polygon Level of Detail (LOD) meshes. This reduces the number of vertices processed per slice, allowing for higher frame rates when voxelizing every frame [S1, unofficial].
2.  **Stream Output Caching (Deforming Meshes)**:
    - For deforming meshes (e.g., skinned characters), compute the transformed vertex positions once per frame.
    - Use GPU Stream Output to capture these transformed vertices into a dedicated buffer.
    - This avoids recomputing skinning matrices and vertex transformations for each voxel slice pass [S1, unofficial].
3.  **Instanced Slicing**:
    - Instead of issuing one draw call per voxel slice (e.g., per Y-slice), issue a single instanced draw call.
    - Pass the slice index or slice-specific transformation as an instance attribute.
    - The geometry shader or vertex shader uses the instance ID to determine which slice to rasterize into, or to transform vertices into the appropriate slice space [S1, unofficial].
4.  **Static Object Handling**:
    - Do not re-voxelize static objects unless their geometry changes or they are intersected by dynamic elements (e.g., dynamic lights that affect illumination calculations) [S2, unofficial].
    - For static objects with simple analytic transformations (rigid body motion), precompute the voxelization and apply the inverse transformation during access rather than re-voxelizing [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for optimized voxelization pipeline
def voxelize_scene(meshes, voxel_grid, frame_time):
    # 1. Process Deforming Meshes
    for mesh in meshes:
        if mesh.is_deforming:
            # Compute transformed vertices once
            transformed_vertices = compute_skinning(mesh)
            
            # Cache using Stream Output
            stream_output_buffer = gpu_stream_output(transformed_vertices)
            
            # 2. Instanced Drawing for Slices
            # Draw all slices in one call
            draw_instanced(
                geometry=mesh.geometry,
                vertex_buffer=stream_output_buffer,
                instance_count=voxel_grid.slice_count,
                instance_data=generate_slice_instances(voxel_grid)
            )
        else:
            # 3. Static Object Optimization
            if mesh.is_static and not needs_revoxelization(mesh):
                continue # Skip re-voxelization
            
            # Simple analytic transformation: Precomputed or inverse transform
            if mesh.has_simple_transform:
                apply_inverse_transform_to_voxel_access(mesh.transform)
            else:
                # Standard voxelization for complex static/dynamic changes
                draw_slices(mesh)

def generate_slice_instances(voxel_grid):
    # Returns an array of instance data, one per slice
    # Each entry contains slice index or slice-specific matrix
    instances = []
    for i in range(voxel_grid.slice_count):
        instances.append({
            'slice_id': i,
            'transform': voxel_grid.get_slice_transform(i)
        })
    return instances
```

### Parameters
- **LOD Level**: The reduction factor for obstacle meshes. Not given in the sources, but described as "low level of detail" [S1, unofficial].
- **Update Frequency**: Static objects are updated only when necessary. Dynamic lights intersecting static objects force re-voxelization [S2, unofficial].

### Edge Cases and Pitfalls
- **Dynamic Lights**: Even static objects must be re-voxelized if dynamic lights intersect them, as this affects lighting calculations (e.g., voxel cone tracing) [S2, unofficial].
- **Precision**: Stream output requires sufficient buffer size to hold all transformed vertices. If the mesh is highly subdivided, memory bandwidth may become a bottleneck.

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
- **Time Complexity**: Reduces vertex processing time for deforming meshes from $O(N \cdot S)$ to $O(N + S)$, where $N$ is the number of vertices and $S$ is the number of slices, by caching transformed vertices [S1, unofficial].
- **Memory Complexity**: Requires additional GPU memory for the stream output buffer and instance data structures [S1, unofficial].
- **Performance**: Allows voxelization of obstacles at every frame with little performance cost when using low-poly LOD meshes [S1, unofficial]. Techniques like these are used in *Rise of Tomb Raider* (voxel AO) and *The Tomorrow Children* (on consoles) [S2, unofficial].
- **Strengths**: Significant reduction in draw calls and vertex shader overhead for dynamic scenes.
- **Limitations**: Requires GPU support for stream output and instancing. Static object optimization requires careful tracking of dynamic light intersections [S2, unofficial].

## Background
Voxelization optimization techniques have been developed to make real-time voxel-based rendering feasible on consumer hardware. The use of stream output for caching transformed vertices and instancing for slice drawing is detailed in *GPU Gems 3* [S1, unofficial]. The application of these techniques in commercial games like *Rise of Tomb Raider* and *The Tomorrow Children* demonstrates their viability for real-time global illumination and ambient occlusion [S2, unofficial].

## Sources
[S1] Real-Time Simulation and Rendering of 3D Fluids, https://wo1fsea.github.io/files/About_NavMesh/P5_GPU_Gems_3_Real-Time_Simulation_and_Rendering_of_3D_Fluids.pdf, fetched 2026-10-27 (unofficial)
[S2] Voxel-based Global Illumination – Wicked Engine, https://turanszkij.wordpress.com/2017/08/30/voxel-based-global-illumination/, fetched 2026-10-28 (unofficial)
[S3] vercidium-patreon/meshing, https://github.com/vercidium-patreon/meshing, fetched 2026-10-26 (unofficial)
