# GPU Acceleration and Parallel Pipelines


## Overview
This family encompasses methods that accelerate Marching Cubes isosurface extraction by offloading voxel processing to the GPU or distributing it across parallel CPU workers [G]. It solves the performance bottleneck of CPU-based extraction for large volumetric datasets by leveraging massive parallelism [G]. It differs from its siblings by focusing on the execution model (compute shaders, job scheduling, histogram pyramids) rather than the topological logic or mesh simplification [G]. It builds on [Marching Cubes Lookup and Configuration](2518-marching-cubes-lookup-and-configuration.md) for the underlying topology.

## Shared principles
The family relies on the precomputed data structures from [Marching Cubes Lookup and Configuration](2518-marching-cubes-lookup-and-configuration.md):
1.  **Edge Intersection Table (`edgeTable`)**: 256-entry bitmask array [G].
2.  **Triangle Connection Table (`triangleTable`)**: 256x16 array of edge indices [G].

**Parallel Execution Model**:
*   **Thread Mapping**: Typically, one thread (or thread group) processes one voxel cube [G].
*   **Output Buffering**: The variable number of triangles per voxel (0 to 5) requires a strategy for writing to a shared output buffer [G]. Common strategies include:
    *   **Atomic Counters**: Using global atomics to allocate slots in a fixed-capacity buffer [G].
    *   **Histogram/Prefix-Sum**: Computing exact offsets beforehand to allow lock-free writes [G].
    *   **Append Buffers**: GPU-specific structures that handle atomic allocation internally [G].

## Subtopics

### [GPU Marching Cubes with Histogram Pyramid](2317-gpu-marching-cubes-with-histogram-pyramid.md)
GPU Marching Cubes with Histogram Pyramid is a parallel isosurface extraction method that uses a Histogram Pyramid (HP) to count and locate surface triangles without atomic operations or sorting. It builds on [Marching Cubes Lookup and Configuration](2518-marching-cubes-lookup-and-configuration.md) and [GPU Acceleration and Parallel Pipelines](4136-gpu-acceleration-and-parallel-pipelines.md). Unlike sibling methods that rely on global atomics or append buffers, this method precomputes exact output offsets via a hierarchical histogram, enabling lock-free writes to the final vertex buffer.

### [Marching Cubes Job Scheduling](4446-marching-cubes-job-scheduling.md)
Marching Cubes Job Scheduling is a parallel execution strategy for isosurface extraction that decomposes a 3D volume into spatial chunks (sub-volumes) and assigns each chunk to an independent processing unit (job or worker process) for concurrent mesh generation. It solves the scalability problem of processing large volumetric datasets by distributing the computational load of voxel classification and triangle generation across multiple CPU cores or GPU threads. Unlike monolithic extraction methods, it enables incremental processing and memory management by handling one chunk at a time, though it requires synchronization mechanisms to aggregate results from independent outputs.

### [Capacity-safe slab-based Marching Cubes](4595-capacity-safe-slab-based-marching-cubes.md)
A GPU-accelerated isosurface extraction pipeline that uses inclusive slab partitioning and a two-pass counting/exact allocation strategy to prevent slab-boundary omissions and output-buffer overflow for large scalar grids. It differs from standard Marching Cubes by resolving topological discontinuities and holes via boundary-consistent slab processing, and differs from HistoPyramids/Flying Edges by specializing counting/compaction for tightly coupled extraction, rendering, and engine integration.

### [Capacity-Safe Marching Cubes Pipeline](4596-capacity-safe-marching-cubes-pipeline.md)
A GPU-based isosurface extraction method that avoids buffer overflow by determining the exact vertex count before geometry generation, using a two-pass approach: first computing counts/offsets, then allocating exact buffers and generating triangles. It sets itself apart from naive parallel Marching Cubes by avoiding buffer overflow and dynamic memory allocation during geometry generation through a two-pass counting and compaction strategy, building on [GPU Acceleration and Parallel Pipelines](4136-gpu-acceleration-and-parallel-pipelines.md). Unlike fixed-capacity baselines that reserve static buffers, this method determines the exact vertex count before generation, ensuring stability and preventing buffer overflow without excessive static reservation.

### [GPU Marching Cubes (Compute Shader)](4687-gpu-marching-cubes-compute-shader.md)
This method implements the Marching Cubes isosurface extraction algorithm using a GPU compute shader within the Unity engine environment. It offloads voxel processing and triangle generation to the GPU while generating the density field on the CPU in its basic form.

### [Parallel Marching Cubes](12211-parallel-marching-cubes.md)
Parallel Marching Cubes is a parallel isosurface extraction technique designed to compute surfaces of huge datasets that cannot fit entirely into a single machine's physical memory. It solves the memory bottleneck of traditional marching cubes by using a piecewise slice strategy along the k-direction, requiring only that at least two slices fit into memory. Unlike siblings that assume full volume residency, this method partitions the workload across multiple processes (using `fork`, `sproc`, or `pthreads`) to process grid slices concurrently.

### [GPU Marching Cubes](16977-gpu-marching-cubes.md)
GPU Marching Cubes is a method for extracting an isosurface from a 3D scalar field by executing the marching cubes algorithm within a GPU compute shader. It differs from CPU-based implementations by leveraging massive parallelism to process grid cells simultaneously, and from traditional graphics pipeline approaches by using a compute shader to generate geometry into a storage buffer, which is then rendered via an indirect draw call. It is particularly suited for dynamic scalar fields, such as fluid simulations using metaballs, where the surface topology changes frequently.

#### Vectorized Marching Cubes
- **Step**: Execute each operation on all cubes (or cubes with the same properties) simultaneously on the condition that there is no dependency among cubes. Vectorize the determination of cases for each cube and vertices of triangles (bilinear interpolation). Vectorize computation of table lookup index values. For example, for basic case 6 with face and internal ambiguities, assign logical variables index1 and index2 for each ambiguity and compute combined index as index = index1 + 2 * index2 [S1].
- **Parameter**: Prerequisite for vectorization: no dependency between successive inputs and outputs [S1].

#### GPU Scalar Field Generation (Tiled Texture)
- **Definition**: Computes the scalar field values on the GPU using a fragment shader over a full-screen quad, outputting to a tiled 2D texture [S2, unofficial].
- **Step**: Determine tiling based on gridSize: 16->4x4, 32->8x4, 64->8x8, 128->8x16. Calculate `texWidth = tilesX * gridSize`, `texHeight = tilesY * gridSize` [G]. Create an R32F texture with `CLAMP_TO_EDGE` wrapping and `NEAREST` filtering [G]. Bind a framebuffer with the tiled texture as `COLOR_ATTACHMENT0` and set the viewport to `texWidth`, `texHeight` [G]. Pass blob parameters as a `Float32Array` of size `numBlobs * 4` [G]. Compute blob parameters on CPU: `pos.x = sin(i + 1.26 * time * (1.03 + 0.5*cos(0.21*i))) * 0.27 + 0.5`, `pos.y = abs(cos(i + 1.12 * time * cos(1.22 + 0.1424*i))) * 0.77`, `pos.z = cos(i + 1.32 * time * 0.1*sin((0.92 + 0.53 * i))) * 0.27 + 0.5`, `strength = 1.2 / ((sqrt(numBlobs)- 1) / 4 + 1)` [S2, unofficial].
- **Parameter**: `tilesX`, `tilesY` (determined by `gridSize`); `texWidth`, `texHeight` [G].
- **Usage**: Implemented in `GpuFieldGenerator` class [S2, unofficial].

#### Incremental Chunk Re-meshing
- **Definition**: A technique to keep deformation fast and seamless by only re-meshing chunks affected by edits, rather than the entire world [S3, unofficial].
- **Step**: Divide the world into chunks. When a dig or build operation modifies the density field, mark only the affected chunks as 'dirty' [G]. In the `LateUpdate` phase of the frame, iterate through dirty chunks and re-mesh them using Marching Cubes [S3, unofficial].

#### Adaptive Hull Subdivision for Linearization Error
- **Step**: Approximate the error $E_T$ for a given triangle $T$ via numerical integration over its surface. Subdivide $T$ based on the magnitude of $E_T$ to ensure error stays below a given threshold. This is done via hardware tessellation when rendering the hull mesh [S4].
- **Formula**: The linearization error at a point $p$ on the deformed surface is $E(p) = \| \mathbf{p} - D(\widetilde{D^{-1}}(\mathbf{p})) \|$, where $\widetilde{D^{-1}}$ is the approximate inverse from the hull. The error over the surface of a deformed space triangle $T$ is $E_T = \int_T E(\mathbf{p}) \, d\mathbf{p}$ [S4].
- **Pitfall**: Increasing grid resolution in Marching Cubes to reduce triangle size scales poorly (8x more cells for double resolution) and requires regenerating the hull every frame [S4].
- **Comparison**: Unlike methods that linearize deformation over volumetric space, this method only linearizes on a 2D manifold (the hull surface), allowing easier quantification of error [S4].

## Background
The transition from CPU to GPU for Marching Cubes began with early implementations using OpenGL shaders and later compute shaders [G]. The primary challenge addressed is the variable-length output of the MC algorithm, which is inefficient for parallel architectures that prefer fixed-size workloads or require atomic operations for dynamic allocation [G]. Methods like Histogram Pyramids and Slab-based partitioning were developed to mitigate atomic contention and buffer overflow issues inherent in naive parallel implementations [G].

## Sources
[S1] Computing and Displaying Isosurfaces inR, https://doi.org/10.18637/jss.v028.i01, fetched 2026-10-27
[S2] 3merillon/WebGL2-GPU-Marching-Cubes, https://github.com/3merillon/WebGL2-GPU-Marching-Cubes, fetched 2026-10-28 (unofficial)
[S3] Rabidus4k/UnityVoxelTerrain, https://github.com/Rabidus4k/UnityVoxelTerrain, fetched 2026-10-28 (unofficial)
[S4] Non-linear sphere tracing for rendering deformed signed distance fields, https://doi.org/10.1145/3355089.3356502, fetched 2026-10-29
