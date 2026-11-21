# GPU Acceleration and Parallel Pipelines

Also called: GPU Marching Cubes, Compute Shader Marching Cubes, Parallel Isosurface Extraction

## Overview
GPU Acceleration and Parallel Pipelines refers to the family of isosurface extraction techniques that offload the voxel classification, edge intersection calculation, and triangle generation stages of Marching Cubes to the GPU, typically using compute shaders or fragment shaders. This approach solves the performance bottleneck of CPU-based extraction by leveraging massive parallelism to process thousands of voxels simultaneously [S1, S2, unofficial]. It differs from CPU implementations by eliminating the host-device data transfer overhead for intermediate geometry and from other GPU variants by the specific memory management strategy employed (e.g., atomic counters, histogram pyramids, or slab-based prefix sums) to handle the variable output size of the algorithm [S1, S2, unofficial].

## Shared principles
The family relies on the precomputed data structures defined in [Marching Cubes Lookup and Configuration](2518-marching-cubes-lookup-and-configuration.md), specifically the `edgeTable` and `triangleTable`.

1.  **Parallel Dispatch**: The 3D voxel grid is mapped to a 3D grid of GPU threads (e.g., in a compute shader) or a 2D grid of fragments (e.g., in a fragment shader). Each thread/fragment processes one voxel independently.
2.  **Lookup Table Access**: The `edgeTable` and `triangleTable` are stored in GPU-accessible memory (textures, constant buffers, or global memory) to allow fast random access by all threads.
3.  **Variable Output Handling**: Since the number of triangles generated per voxel varies (0 to 5), a mechanism is required to write the resulting vertices/indices into a contiguous output buffer without race conditions or buffer overflows. Common mechanisms include:
    *   **Atomic Counters**: Threads atomically increment a global counter to reserve slots in a pre-allocated buffer [S1, S2, unofficial].
    *   **Histogram/Prefix Sum**: A two-pass approach where the first pass counts triangles per voxel and computes offsets, and the second pass writes data to exact positions [S1, S2, unofficial].
    *   **Append Buffers**: Hardware-supported append buffers that automatically manage the write index [S1, unofficial].
4.  **Boundary Handling**: Threads at the grid boundary must check if their voxel neighbors exist to avoid out-of-bounds memory access when sampling the 8 corners of the cube [S1, S2, unofficial].

## Subtopics

### [Primal Grid Marching Cubes](1211-primal-grid-marching-cubes.md)
A GPU-accelerated technique that processes a 3D scalar volume grid using compute shaders to output a triangle mesh. It differs from CPU implementations by offloading voxel processing and triangle generation entirely to the GPU, and from other GPU variants by relying on a fixed-size output buffer managed by a global atomic counter and a subsequent cleanup pass, rather than hierarchical reduction or capacity-safe slab-based allocation. This method builds on [GPU Acceleration and Parallel Pipelines](4136-gpu-acceleration-and-parallel-pipelines.md) by mapping the primal grid directly to parallel thread groups.

### [GPU Marching Cubes with Histogram Pyramid](2317-gpu-marching-cubes-with-histogram-pyramid.md)
This method extracts isosurfaces from volumetric scalar fields on the GPU using OpenCL and OpenGL, employing a Histogram Pyramid (HP) to count triangles per voxel and compactly store vertex data in a Vertex Buffer Object (VBO) without atomic operations or sorting. It builds on the lookup tables and parallel dispatch principles defined in [Marching Cubes Lookup and Configuration](2518-marching-cubes-lookup-and-configuration.md) and [GPU Acceleration and Parallel Pipelines](4136-gpu-acceleration-and-parallel-pipelines.md).

### [CPU Immediate Mode Marching Cubes](2463-cpu-immediate-mode-marching-cubes.md)
CPU Immediate Mode Marching Cubes is a classic CPU-based surface extraction method that generates mesh vertices and normals and renders them directly via graphics API draw calls (e.g., WebGL `gl.drawArrays`) without storing a persistent, complex mesh data structure in application memory. It solves the problem of visualizing scalar fields (such as metaballs) by computing the geometry on the CPU every frame and streaming it to the GPU. It is distinguished from its siblings, such as [GPU Acceleration and Parallel Pipelines](4136-gpu-acceleration-and-parallel-pipelines.md), by performing all field evaluation, topology classification, and vertex interpolation on the CPU, whereas the GPU siblings offload these computations to shaders.

### [Marching Cubes with Gaussian Deformation](3769-marching-cubes-with-gaussian-deformation.md)
This method extracts an isosurface from a scalar field using a Geometry Shader, applying a Gaussian-based deformation to the generated vertices along a user-provided force vector. It differs from standard GPU Marching Cubes by integrating the deformation directly into the geometry shader's vertex emission phase, allowing for real-time, dynamic modification of the mesh topology and shape based on external forces. Unlike [GPU Marching Cubes with Atomic Counter Emission](4082-gpu-marching-cubes-with-atomic-counter-emission.md) which focuses on buffer management, this variant prioritizes the mathematical deformation of the surface geometry itself.

### [GPU Marching Cubes with Atomic Counter Emission](4082-gpu-marching-cubes-with-atomic-counter-emission.md)
This method is a real-time GPU-based isosurface extraction technique that eliminates the need for prefix sums or histogram pyramids by using atomic operations to reserve vertex slots directly within indirect draw arguments. It assigns one compute thread per grid cell to classify density, emit interpolated vertices into a capped pool buffer, and update a global vertex counter atomically, enabling immediate rendering via `drawIndirect`. Unlike sibling methods that rely on pre-computed output sizes or hierarchical data structures, this approach dynamically manages memory allocation on the GPU with minimal synchronization overhead, making it suitable for real-time fluid simulation visualizations where mesh topology changes rapidly.

### [Marching Cubes Job Scheduling](4446-marching-cubes-job-scheduling.md)
Marching Cubes Job Scheduling is a parallel processing strategy that decomposes a volumetric dataset into spatial chunks and dispatches independent Marching Cubes extraction tasks to a job scheduler (specifically the Unity Job System) for concurrent execution. It solves the problem of CPU-bound isosurface extraction bottlenecks by offloading the classification, interpolation, and triangle emission steps to background worker threads, allowing the main thread to continue rendering or handling user input. Unlike immediate-mode CPU approaches, it leverages multi-core parallelism through explicit job scheduling; unlike pure GPU compute shaders, it manages memory allocation and job synchronization via a high-level job system API, often targeting dynamic terrain or voxel worlds where chunks are generated or modified independently.

### [Capacity-safe slab-based Marching Cubes](4595-capacity-safe-slab-based-marching-cubes.md)
This method is a GPU-accelerated isosurface extraction pipeline that partitions the z-axis into slabs and uses a two-pass counting/exact allocation strategy to prevent output-buffer overflow and eliminate boundary cracks in large scalar grids. It differs from standard Marching Cubes by resolving topological discontinuities via boundary-consistent inclusive slab indexing, and differs from [GPU Marching Cubes with Atomic Counter Emission](4082-gpu-marching-cubes-with-atomic-counter-emission.md) by using a deterministic two-pass scan-and-generate approach rather than atomic operations, ensuring exact buffer allocation without over-provisioning.

### [Capacity-Safe Marching Cubes Pipeline](4596-capacity-safe-marching-cubes-pipeline.md)
A GPU-based isosurface extraction method that avoids buffer overflow by determining the exact vertex count before geometry generation, using a two-pass approach: first computing counts/offsets, then allocating exact buffers and generating triangles. Unlike fixed-capacity baselines that reserve static buffers, this method determines the exact vertex count before generation, ensuring stability and preventing buffer overflow without excessive static reservation. It builds on [GPU Acceleration and Parallel Pipelines](4136-gpu-acceleration-and-parallel-pipelines.md) by adding a precise memory management layer to the parallel voxel processing.

### [GPU Marching Cubes (Compute Shader)](4687-gpu-marching-cubes-compute-shader.md)
This method implements the Marching Cubes isosurface extraction algorithm entirely on the GPU using a compute shader, typically within a framework like Unity. It solves the performance bottleneck of CPU-based volume rendering by offloading the classification, edge intersection calculation, and triangle generation to parallel shader cores. Unlike its siblings, this specific implementation relies on a CPU-generated density field (Texture3D) and uses an append buffer for dynamic triangle emission, making it a straightforward but potentially bottlenecked approach for real-time applications where the density field generation is not also GPU-accelerated.

### GPU Scalar Field Generation (Tiled Texture)
- **Definition**: Computes the scalar field values on the GPU using a fragment shader over a full-screen quad, outputting to a tiled 2D texture [S1, unofficial].
- **Steps**: Determine tiling based on `gridSize`: 16->4x4, 32->8x4, 64->8x8, 128->8x16. Calculate `texWidth = tilesX * gridSize`, `texHeight = tilesY * gridSize`. Create an R32F texture with `CLAMP_TO_EDGE` wrapping and `NEAREST` filtering. Bind a framebuffer with `tiledTexture` as `COLOR_ATTACHMENT0`. Set viewport to `texWidth`, `texHeight`. Pass blob parameters as a `Float32Array` of size `numBlobs * 4`. Compute blob params on CPU: `pos.x = sin(i + 1.26 * time * (1.03 + 0.5*cos(0.21*i))) * 0.27 + 0.5`, `pos.y = abs(cos(i + 1.12 * time * cos(1.22 + 0.1424*i))) * 0.77`, `pos.z = cos(i + 1.32 * time * 0.1*sin((0.92 + 0.53 * i))) * 0.27 + 0.5`, `strength = 1.2 / ((sqrt(numBlobs)- 1) / 4 + 1)` [S1, unofficial].
- **Parameters**: `tilesX`, `tilesY` (number of tiles in X and Y directions, determined by `gridSize`); `texWidth`, `texHeight` (dimensions of the tiled texture) [S1, unofficial].
- **Usage**: Implemented in `GpuFieldGenerator` class [S1, unofficial].

### Incremental Chunk Re-meshing
- **Definition**: A technique to keep deformation fast and seamless by only re-meshing chunks affected by edits, rather than the entire world [S2, unofficial].
- **Steps**: Divide the world into chunks. When a dig or build operation modifies the density field, mark only the affected chunks as 'dirty'. In the `LateUpdate` phase of the frame, iterate through dirty chunks and re-mesh them using Marching Cubes [S2, unofficial].

## Background
The application of GPU acceleration to Marching Cubes emerged as a response to the computational intensity of processing large volumetric datasets on the CPU. Early implementations utilized fragment shaders to render implicit surfaces directly, bypassing explicit mesh generation [S1, unofficial]. As compute shaders became available, they allowed for more flexible data structures and explicit mesh generation, enabling real-time interaction with dynamic volumes such as fluid simulations and metaballs [S1, S2, unofficial]. The family has evolved to address specific challenges such as buffer overflow (via histogram pyramids and slab-based prefix sums) and dynamic deformation (via geometry shaders) [S1, S2, unofficial].

## Sources
[S1] 3merillon/WebGL2-GPU-Marching-Cubes, https://github.com/3merillon/WebGL2-GPU-Marching-Cubes, fetched 2026-10-28 (unofficial)
[S2] Rabidus4k/UnityVoxelTerrain, https://github.com/Rabidus4k/UnityVoxelTerrain, fetched 2026-10-28 (unofficial)
