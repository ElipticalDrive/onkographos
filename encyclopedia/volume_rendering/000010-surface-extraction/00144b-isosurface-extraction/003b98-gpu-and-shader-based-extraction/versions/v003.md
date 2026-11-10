# GPU and Shader-Based Extraction

Also called: GPU Isosurface Extraction, Shader-Based Surface Extraction

## Overview
This family encompasses methods that perform isosurface extraction entirely or partially on the GPU using programmable shaders (Compute, Geometry, Fragment) to bypass CPU bottlenecks and reduce data transfer latency. Unlike CPU-based methods that require reading back geometry or managing complex synchronization, GPU-based approaches leverage parallel processing for cell classification, vertex interpolation, and primitive generation. They differ from their siblings by utilizing the graphics pipeline's parallelism (e.g., one thread per voxel or fragment) and hardware features like atomic counters, append buffers, or render-to-vertex-array to construct meshes in real-time.

## Shared principles
All methods in this family operate on a discretized domain, typically a regular grid (Cartesian, octree, or tetrahedral) where each cell contains scalar values or occupancy states at its vertices or centers. The core mechanism involves traversing the grid cells and evaluating the data to determine which parts of the domain constitute the surface [G].

1.  **Domain Traversal**: The algorithm iterates over the volume data structure [G]. For regular grids, this often involves checking each cell or voxel [G].
2.  **Surface Determination**: A criterion is applied to identify surface locations. In isosurface methods, this involves comparing scalar values against a threshold (isovalue) to detect sign changes or equality [30-isosurface-extraction.md]. In blocky meshing, this involves checking occupancy states or material IDs to identify exposed boundaries [31-blocky-meshing.md].
3.  **Primitive Generation**: Geometric primitives (vertices, edges, faces) are generated based on the determined surface locations [G].
4.  **Mesh Coherence**: Adjacent primitives are merged or connected to form a coherent mesh, often requiring vertex merging to avoid duplicates and ensure watertightness [30-isosurface-extraction.md].

## Subtopics

### [Marching Cubes with Gaussian Deformation](3769-marching-cubes-with-gaussian-deformation.md)
This method extracts an isosurface from a scalar field using a GPU geometry shader, applying a Gaussian-based deformation to the resulting mesh vertices along a user-defined force vector. It differs from standard Marching Cubes by integrating the deformation directly into the primitive generation stage, allowing for real-time, procedural modification of the surface topology without pre-computing a deformed scalar field. It builds on [GPU and Shader-Based Extraction](9036-gpu-and-shader-based-extraction.md) by utilizing the geometry shader's ability to generate new primitives and modify vertex attributes on the fly.

### [GPU Marching Cubes Terrain Pipeline with Indirect Rendering](3803-gpu-marching-cubes-terrain-pipeline-with-indirect-rendering.md)
This method extracts an implicit terrain surface from voxel data on the GPU using Marching Cubes, manages multiple Level of Detail (LOD) levels, and renders the generated triangles via indirect draw calls. It builds on [GPU and Shader-Based Extraction](9036-gpu-and-shader-based-extraction.md) by replacing CPU-GPU synchronization stalls with GPU-generated draw arguments, allowing the GPU to render the mesh directly without readback. Unlike [GPU Marching Cubes with Atomic Counter Emission](4082-gpu-marching-cubes-with-atomic-counter-emission.md) which may require CPU-side buffer management or synchronization, this approach keeps the RenderThread time low (~1.5ms) by keeping all draw argument generation on the GPU.

### [GPU-accelerated Marching Cubes with Distance Extraction](3818-gpu-accelerated-marching-cubes-with-distance-extraction.md)
This method extracts a triangulated surface mesh (STL) from a Signed Distance Function (SDF) by offloading the scalar field evaluation to the GPU while performing the topological mesh generation on the CPU. It differs from pure GPU approaches by extracting vertex distances via fragment shader output first, thereby avoiding the high latency and cost of reading large volumes of geometry data back from GPU buffers. It is distinct from sibling methods that perform the entire marching cubes logic in shaders or use indirect rendering pipelines for terrain, as it relies on a hybrid CPU-GPU workflow.

### [GPU Marching Cubes with Atomic Counter Emission](4082-gpu-marching-cubes-with-atomic-counter-emission.md)
This method is a real-time GPU-based isosurface extraction technique that eliminates the need for prefix sums or histogram pyramids by using atomic operations to reserve vertex slots directly within indirect draw arguments. It assigns one compute thread per grid cell to classify density, emit interpolated vertices into a capped pool buffer, and update a global vertex counter atomically, enabling immediate rendering via `drawIndirect`. Unlike sibling methods that rely on pre-computed output sizes or hierarchical data structures, this approach dynamically manages memory allocation on the GPU with minimal synchronization overhead, making it suitable for real-time fluid simulation visualizations where mesh topology changes rapidly.

### [Transvoxel](6277-transvoxel.md)
Transvoxel is a level-of-detail (LOD) aware isosurface extraction method that extends Marching Cubes to handle voxel grids with varying resolutions. It solves the problem of mesh discontinuities at boundaries where adjacent voxels have different LODs by introducing specialized "transition cells" that generate connecting geometry. Unlike standard Marching Cubes, which assumes a uniform grid, Transvoxel ensures a watertight surface by dynamically adjusting vertex positions and normals in transition regions.

### [GPU Isosurface Extraction from Molecular Orbital Densities](6862-gpu-isosurface-extraction-from-molecular-orbital-densities.md)
An optimized GPU-based pipeline for extracting isosurfaces from molecular orbital densities (Gaussian Type Orbitals, GTOs) without blocking the host thread, utilizing an octree for spatial culling, shared corner computation, and Newton projection. It distinguishes itself from sibling methods like [Marching Cubes with Gaussian Deformation](3769-marching-cubes-with-gaussian-deformation.md) by operating on a "gridless" basis where the mesh is generated directly from atom-centered basis functions rather than a uniform voxel grid, and from [Transvoxel](6277-transvoxel.md) by its specific focus on quantum chemical density fields and non-blocking job management.

### [Fragment Shader Isosurface Extraction with MRTs](7507-fragment-shader-isosurface-extraction-with-mrts.md)
This method computes isosurfaces from unstructured tetrahedral grids using fragment shaders and Multiple Render Targets (MRTs), enabling the generated geometry to be reused via render-to-vertex-arrays. It distinguishes itself from sibling methods by decoupling surface extraction from immediate rasterization; instead of emitting triangles directly to the screen, it writes vertex positions to texture buffers, allowing for flexible post-processing and view-independent extraction.

### GPU Marching Cubes (Potential Triangles)
**Definition**: Renders the isosurface on the GPU by enumerating all possible triangles per voxel in a single draw call, letting the vertex shader compute and discard inactive triangles [S1, unofficial].
**Steps**: Calculate `numVoxels = (gridSize - 1)^3`. Set `maxTrisPerVoxel = 5`, `vertsPerTri = 3`. Compute `vertexCount = numVoxels * maxTrisPerVoxel * vertsPerTri`. Allocate `Uint32Array` of size `vertexCount * 3`. Fill indices with nested loops: for each voxel `v`, triangle `t` (0-4), vertex `k` (0-2), store `(v, t, k)`. Create VAO and buffer data with `gl.STATIC_DRAW`. Enable vertex attributes 0, 1, 2. Set `vertexAttribIPointer` with stride 12, offsets 0, 4, 8, type `UNSIGNED_INT` [S1, unofficial].
**Parameters**: `isoLevel`: isosurface threshold, default 80.0 [S1, unofficial].
**Layout**: Index buffer layout: 3 `UNSIGNED_INT` components per entry, stride 12 bytes. Component 0: voxel index. Component 1: triangle index (0-4). Component 2: vertex index (0-2) [S1, unofficial].
**Comparison**: Uses a single static index buffer enumerating all potential triangles instead of dynamic per-frame triangle lists, reducing CPU-GPU transfer overhead [S1, unofficial].
**Usage**: Implemented in `GpuMarchingCubesRenderer` class [S1, unofficial].

## Background
GPU-based extraction emerged with the advent of programmable shaders (OpenGL 2.0+, DirectX 10+) allowing general-purpose computation on the GPU (GPGPU). Early implementations used fragment shaders to evaluate scalar fields and render points or lines, evolving to geometry shaders for primitive generation. The shift to Compute Shaders (OpenGL 4.3+, DirectX 11+) enabled more flexible data structures and atomic operations, critical for dynamic mesh generation without fixed-function pipeline limitations. Transvoxel (2007) was a seminal work in adaptive LOD on the GPU. Recent trends focus on indirect rendering and atomic counters to eliminate CPU-GPU synchronization stalls, enabling real-time extraction for fluid simulations and terrain generation.

## Sources
[S1] 3merillon/WebGL2-GPU-Marching-Cubes, https://github.com/3merillon/WebGL2-GPU-Marching-Cubes, fetched 2026-10-28 (unofficial)
[S2] Transvoxel: Adaptive Isosurface Extraction for Voxel Data, https://www.iquilezles.org/www/articles/transvoxel/transvoxel.htm, fetched 2026-10-28 (unofficial)
