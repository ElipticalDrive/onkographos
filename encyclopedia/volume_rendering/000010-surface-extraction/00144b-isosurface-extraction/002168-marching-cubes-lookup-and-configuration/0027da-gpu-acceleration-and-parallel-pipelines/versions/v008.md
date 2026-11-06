# GPU Acceleration and Parallel Pipelines

Also called: GPU Marching Cubes, Compute Shader MC, Parallel Isosurface Extraction

## Overview
This family encompasses techniques that execute the Marching Cubes isosurface extraction algorithm on a Graphics Processing Unit (GPU) to achieve real-time performance on large volumetric datasets [S1, S2, unofficial]. It solves the computational bottleneck of CPU-based extraction by mapping voxel processing to parallel thread groups (compute shaders) or shader stages (geometry shaders) [S1, S2, unofficial]. What sets this family apart from its siblings is the diversity of output management strategies: methods range from fixed-capacity buffers with atomic counters [S1, unofficial], to histogram pyramids for exact allocation [S1, unofficial], to capacity-safe two-pass pipelines that guarantee no buffer overflow [S1, unofficial], and to indirect rendering pipelines that eliminate CPU-GPU synchronization stalls [S1, unofficial].

## Shared principles
The family relies on the precomputed data structures and case index logic defined in [Marching Cubes Lookup and Configuration](2518-marching-cubes-lookup-and-configuration.md). Specifically, every method in this family utilizes:
1.  **Edge Intersection Table (`edgeTable`)**: A 256-entry lookup mapping the 8-bit case index to a bitmask of intersected edges [S4, S12, unofficial].
2.  **Triangle Connection Table (`triangleTable`)**: A 256x16 lookup mapping the case index to a sequence of edge indices forming triangles, terminated by -1 [S8, S12, unofficial].
3.  **Case Index Calculation**: For each voxel, the 8-bit index $C$ is derived from the scalar field values $V$ at the 8 vertices relative to an isovalue $I$ [S7, S8, S12, unofficial].

All methods offload the classification, interpolation, and triangle emission steps to the GPU, typically using one thread or shader invocation per voxel [S1, S2, unofficial].

## Subtopics

### [Primal Grid Marching Cubes](1211-primal-grid-marching-cubes.md)
A GPU-accelerated technique using compute shaders to output a triangle mesh into fixed-size buffers managed by a global atomic counter. It differs from other GPU variants by relying on a fixed-size output buffer and a subsequent cleanup pass to handle overflow, rather than hierarchical reduction or exact allocation. It builds on [GPU Acceleration and Parallel Pipelines](4136-gpu-acceleration-and-parallel-pipelines.md) by mapping the primal grid directly to parallel thread groups.

### [GPU Marching Cubes with Histogram Pyramid](2317-gpu-marching-cubes-with-histogram-pyramid.md)
This method extracts isosurfaces using OpenCL and OpenGL, employing a Histogram Pyramid (HP) to count triangles per voxel and compactly store vertex data in a Vertex Buffer Object (VBO) without atomic operations or sorting. It builds on the lookup tables and parallel dispatch principles defined in [Marching Cubes Lookup and Configuration](2518-marching-cubes-lookup-and-configuration.md) and [GPU Acceleration and Parallel Pipelines](4136-gpu-acceleration-and-parallel-pipelines.md).

### [Marching Cubes with Gaussian Deformation](3769-marching-cubes-with-gaussian-deformation.md)
This method extracts a triangle mesh using a GPU geometry shader, applying a Gaussian-based deformation to generated vertices along a user-defined force vector. It differs from standard implementations by integrating deformation directly into the geometry shader's vertex emission phase, allowing real-time, procedural modification of the isosurface shape without modifying the underlying scalar field data. It builds on [GPU Acceleration and Parallel Pipelines](4136-gpu-acceleration-and-parallel-pipelines.md).

### [GPU Marching Cubes Terrain Pipeline with Indirect Rendering](3803-gpu-marching-cubes-terrain-pipeline-with-indirect-rendering.md)
This method extracts an implicit terrain surface from voxel data on the GPU, managing multiple Level of Detail (LOD) levels and rendering via indirect draw calls. It builds on [GPU Acceleration and Parallel Pipelines](4136-gpu-acceleration-and-parallel-pipelines.md) by replacing CPU-GPU synchronization stalls with GPU-generated draw arguments, allowing the GPU to render the mesh directly without readback. Unlike [GPU Marching Cubes with Atomic Counter Emission](4082-gpu-marching-cubes-with-atomic-counter-emission.md) which may require CPU-side buffer management, this approach keeps RenderThread time low (~1.5ms) by keeping all draw argument generation on the GPU.

### [GPU Marching Cubes with Atomic Counter Emission](4082-gpu-marching-cubes-with-atomic-counter-emission.md)
This method performs real-time isosurface meshing by mapping one compute thread per grid cell to classify density, reserve vertex slots via atomic addition on indirect draw arguments, and write vertices directly into a capped pool, thereby avoiding the overhead of prefix sums. It is distinguished from siblings like [GPU Marching Cubes with Histogram Pyramid](2317-gpu-marching-cubes-with-histogram-pyramid.md) and [GPU Marching Cubes (Compute Shader)](4687-gpu-marching-cubes-compute-shader.md) by its use of atomic counters on the indirect-draw buffer for output management rather than histogram pyramids or fixed-size buffers with complex synchronization.

### [Capacity-safe slab-based Marching Cubes](4595-capacity-safe-slab-based-marching-cubes.md)
This method is a GPU-accelerated isosurface extraction pipeline that partitions the z-axis into slabs to compute exact vertex counts per slab via a two-pass strategy (classify/scan then generate), allowing for exact buffer allocation and crack-free surfaces at slab boundaries. It differs from standard Marching Cubes by resolving topological discontinuities and holes via boundary-consistent slab processing, and differs from HistoPyramids/Flying Edges by specializing counting/compaction for tightly coupled extraction, rendering, and engine integration. Unlike implicit isosurface raycasting, this method generates explicit geometric assets (vertices/normals) suitable for mesh-based interaction tools and OBJ export, whereas raycasting is more memory-efficient for direct visualization of single isovalues.

### [Capacity-Safe Marching Cubes Pipeline](4596-capacity-safe-marching-cubes-pipeline.md)
The Capacity-Safe Marching Cubes Pipeline is a GPU-accelerated isosurface extraction method that prevents buffer overflow by determining the exact vertex count before geometry generation. Unlike fixed-capacity baselines that reserve static buffers and risk failure at high resolutions, this method uses a two-pass approach: first computing counts and offsets, then allocating exact buffers and generating triangles. It builds on [GPU Acceleration and Parallel Pipelines](4136-gpu-acceleration-and-parallel-pipelines.md) by replacing dynamic or fixed-size output emission with a deterministic allocation strategy.

### [GPU Marching Cubes (Compute Shader)](4687-gpu-marching-cubes-compute-shader.md)
This method implements the Marching Cubes isosurface extraction algorithm using a GPU Compute Shader, specifically designed for integration within the Unity engine. It solves the problem of real-time surface generation from volumetric data by offloading the voxel processing and triangle generation to the GPU, while maintaining a CPU-based density field generation for demonstration purposes. It differs from its siblings primarily in its use of an `AppendStructuredBuffer` for dynamic vertex emission and `Graphics.DrawProceduralIndirect` for rendering, relying on the parallel dispatch and lookup logic defined in [GPU Acceleration and Parallel Pipelines](4136-gpu-acceleration-and-parallel-pipelines.md) and [Marching Cubes Lookup and Configuration](2518-marching-cubes-lookup-and-configuration.md).

#### GPU Marching Cubes (Potential Triangles)
- **Definition**: Renders the isosurface on the GPU by enumerating all possible triangles per voxel in a single draw call, letting the vertex shader compute and discard inactive triangles [S1, unofficial].
- **Steps**: Calculate `numVoxels = (gridSize - 1)^3`. Set `maxTrisPerVoxel = 5`, `vertsPerTri = 3`. Compute `vertexCount = numVoxels * maxTrisPerVoxel * vertsPerTri`. Allocate `Uint32Array` of size `vertexCount * 3`. Fill indices with nested loops: for each voxel $v$, triangle $t$ (0-4), vertex $k$ (0-2), store $(v, t, k)$. Create VAO and buffer data with `gl.STATIC_DRAW`. Enable vertex attributes 0, 1, 2. Set `vertexAttribIPointer` with stride 12, offsets 0, 4, 8, type `UNSIGNED_INT` [S1, unofficial].
- **Parameters**: `isoLevel`: isosurface threshold, default 80.0 [S1, unofficial].
- **Layout**: Index buffer layout: 3 `UNSIGNED_INT` components per entry, stride 12 bytes. Component 0: voxel index. Component 1: triangle index (0-4). Component 2: vertex index (0-2) [S1, unofficial].
- **Comparison**: Uses a single static index buffer enumerating all potential triangles instead of dynamic per-frame triangle lists, reducing CPU-GPU transfer overhead [S1, unofficial].
- **Usage**: Implemented in `GpuMarchingCubesRenderer` class [S1, unofficial].

#### GPU Scalar Field Generation (Tiled Texture)
- **Definition**: Computes the scalar field values on the GPU using a fragment shader over a full-screen quad, outputting to a tiled 2D texture [S1, unofficial].
- **Steps**: Determine tiling based on `gridSize`: 16->4x4, 32->8x4, 64->8x8, 128->8x16. Calculate `texWidth = tilesX * gridSize`, `texHeight = tilesY * gridSize`. Create `R32F` texture with `CLAMP_TO_EDGE` wrapping and `NEAREST` filtering. Bind framebuffer with `tiledTexture` as `COLOR_ATTACHMENT0`. Set viewport to `texWidth`, `texHeight`. Pass blob parameters as `Float32Array` of size `numBlobs * 4`. Compute blob params on CPU: `pos.x = sin(i + 1.26 * time * (1.03 + 0.5*cos(0.21*i))) * 0.27 + 0.5`, `pos.y = abs(cos(i + 1.12 * time * cos(1.22 + 0.1424*i))) * 0.77`, `pos.z = cos(i + 1.32 * time * 0.1*sin((0.92 + 0.53 * i))) * 0.27 + 0.5`, `strength = 1.2 / ((sqrt(numBlobs)- 1) / 4 + 1)` [S1, unofficial].
- **Parameters**: `tilesX`, `tilesY`: number of tiles in X and Y directions, determined by `gridSize`. `texWidth`, `texHeight`: dimensions of the tiled texture [S1, unofficial].
- **Usage**: Implemented in `GpuFieldGenerator` class [S1, unofficial].

#### Incremental Chunk Re-meshing
- **Definition**: A technique to keep deformation fast and seamless by only re-meshing chunks affected by edits, rather than the entire world [S2, unofficial].
- **Steps**: Divide the world into chunks. When a dig or build operation modifies the density field, mark only the affected chunks as 'dirty'. In the `LateUpdate` phase of the frame, iterate through dirty chunks and re-mesh them using Marching Cubes [S2, unofficial].

## Background
The application of GPU acceleration to Marching Cubes emerged as volumetric data sizes grew beyond the capabilities of CPU-based extraction for real-time applications. Early implementations utilized OpenGL geometry shaders to generate triangles on-the-fly, leveraging the fixed-function pipeline's ability to emit primitives [S1, S2, unofficial]. With the advent of compute shaders (OpenGL 4.3+, DirectX 11+), more flexible parallel processing models became available, allowing for complex output management strategies such as atomic counters, histogram pyramids, and indirect rendering [S1, S2, unofficial]. These methods are widely used in real-time volume visualization, medical imaging, and voxel-based game engines (e.g., Minecraft-style terrain generation) where dynamic isosurface extraction is required [S1, S2, unofficial].

## Sources
[S1] 3merillon/WebGL2-GPU-Marching-Cubes, https://github.com/3merillon/WebGL2-GPU-Marching-Cubes, fetched 2026-10-28 (unofficial)
[S2] Rabidus4k/UnityVoxelTerrain, https://github.com/Rabidus4k/UnityVoxelTerrain, fetched 2026-10-28 (unofficial)
