# GPU-Driven Tetrahedral Mesh Rasterization Pipeline

Also called: Differentiable Tetrahedral Volume Rasterization, TetMeshRendererIntersection

## Overview
This method manages the rendering of tetrahedral meshes on the GPU by handling sorting algorithms, buffer creation for triangles and tetrahedra, and indirect draw/dispatch command setup [S1, unofficial]. It serves as the execution engine for the forward intersect raster pass, projecting tetrahedral mesh cells into triangles, sorting them back-to-front, and rasterizing them with back-to-front blending to compute volume color and opacity [S1, unofficial].

## How it works
**Inputs and Outputs:**
* **Input:** A tetrahedral mesh defined by vertices and cell connectivity [G].
* **Output:** A rendered image with accumulated color and opacity [G].

**Data Structures:**
The pipeline relies on several GPU buffers managed via Vulkan Memory Allocator (VMA) with `VMA_MEMORY_USAGE_GPU_ONLY` [S1, unofficial].
1. **UniformDataBuffer:** Created with usage `VK_BUFFER_USAGE_TRANSFER_DST_BIT | VK_BUFFER_USAGE_UNIFORM_BUFFER_BIT` [S1, unofficial].
2. **TriangleCounterBuffer:** Created with usage `VK_BUFFER_USAGE_TRANSFER_DST_BIT | VK_BUFFER_USAGE_STORAGE_BUFFER_BIT | VK_BUFFER_USAGE_UNIFORM_BUFFER_BIT | VK_BUFFER_USAGE_SHADER_DEVICE_ADDRESS_BIT`; adds `VK_BUFFER_USAGE_TRANSFER_SRC_BIT` if `sortingAlgorithm` is `CPU_STD_SORT` [S1, unofficial].
3. **TriangleVertexPositionBuffer:** Created with usage `VK_BUFFER_USAGE_STORAGE_BUFFER_BIT` [S1, unofficial]. Size: `maxNumVertices * sizeof(glm::vec4)`, where `maxNumVertices = maxNumTriangles * 3` [S1, unofficial].
4. **TriangleTetIndexBuffer:** Created with usage `VK_BUFFER_USAGE_STORAGE_BUFFER_BIT` [S1, unofficial]. Size: `maxNumTriangles * sizeof(uint32_t)` [S1, unofficial].
5. **TetIndexBuffer:** Created with usage `VK_BUFFER_USAGE_STORAGE_BUFFER_BIT` [S1, unofficial].
6. **TetVertexPositionBuffer:** Created with usage `VK_BUFFER_USAGE_STORAGE_BUFFER_BIT` [S1, unofficial].
7. **DrawIndirectCommandBuffer / DispatchIndirectCommandBuffer:** Created with usage `VK_BUFFER_USAGE_STORAGE_BUFFER_BIT | VK_BUFFER_USAGE_INDIRECT_BUFFER_BIT` [S1, unofficial].
8. **TriangleKeyValueBuffer:** Created with usage `VK_BUFFER_USAGE_STORAGE_BUFFER_BIT` [S1, unofficial].
9. **TriangleVertexColorBuffer:** Created with usage `VK_BUFFER_USAGE_STORAGE_BUFFER_BIT` [S1, unofficial].

**Algorithm Steps:**

1. **Initialization:**
   * When `setTetMeshData` is called, calculate `maxNumTriangles = tetMesh->getNumCells() * 3` and `maxNumVertices = maxNumTriangles * 3` [S1, unofficial].
   * Create `TriangleVertexPositionBuffer` with size `maxNumVertices * sizeof(glm::vec4)` and `TriangleTetIndexBuffer` with size `maxNumTriangles * sizeof(uint32_t)` [S1, unofficial].
   * Call `recreateSortingBuffers()` to allocate sorting resources [S1, unofficial].
   * Initialize sorting algorithm: if `FUCHSIA_RADIX_SORT` is supported, detect target and create `radixSortVk`; otherwise fallback to `CPU_STD_SORT` [S1, unofficial].

2. **Triangle Generation Compute Pass:**
   * Clear the `TriangleCounterBuffer` to zero using a transfer operation [S1, unofficial].
   * Insert a memory barrier for the `TriangleCounterBuffer` transitioning from `VK_ACCESS_TRANSFER_WRITE_BIT` to `VK_ACCESS_SHADER_READ_BIT | VK_ACCESS_SHADER_WRITE_BIT` across `VK_PIPELINE_STAGE_TRANSFER_BIT` to `VK_PIPELINE_STAGE_COMPUTE_SHADER_BIT` [S1, unofficial].
   * Dispatch the triangle generation compute shader with workgroup count `ceil(numCells / BLOCK_SIZE)` in X, 1 in Y, 1 in Z [S1, unofficial].
   * Insert memory barriers for `TriangleVertexPositionBuffer` and `TriangleTetIndexBuffer` transitioning from `VK_ACCESS_SHADER_WRITE_BIT` to `VK_ACCESS_SHADER_READ_BIT` within `VK_PIPELINE_STAGE_COMPUTE_SHADER_BIT` [S1, unofficial].

3. **Indirect Command Initialization:**
   * Dispatch the indirect command initialization compute shader with 1x1x1 workgroups [S1, unofficial].
   * Insert memory barriers for `DrawIndirectCommandBuffer` and `DispatchIndirectCommandBuffer` transitioning from `VK_ACCESS_SHADER_WRITE_BIT` to `VK_ACCESS_INDIRECT_COMMAND_READ_BIT` across `VK_PIPELINE_STAGE_COMPUTE_SHADER_BIT` to `VK_PIPELINE_STAGE_DRAW_INDIRECT_BIT` [S1, unofficial].

4. **Triangle Depth Computation Pass:**
   * Execute the triangle depth computation pass using indirect dispatch from the `DispatchIndirectCommandBuffer` [S1, unofficial].
   * Insert a memory barrier for `TriangleKeyValueBuffer` transitioning from `VK_ACCESS_SHADER_WRITE_BIT` to `VK_ACCESS_SHADER_READ_BIT | VK_ACCESS_SHADER_WRITE_BIT` within `VK_PIPELINE_STAGE_COMPUTE_SHADER_BIT` [S1, unofficial].

5. **Rasterization Pass (Forward Intersect Raster Pass):**
   * Configure the rasterization pipeline: disable culling, set front face to counter-clockwise, enable color write, disable depth write and depth test, and set blend mode to `BACK_TO_FRONT_PREMUL_ALPHA` [S1, unofficial].
   * Load rasterization shaders with preprocessor defines `PI_SQRT`, `INV_PI_SQRT`, `SHOW_TET_QUALITY`, `USE_SHADING` (conditional), and `BACK_TO_FRONT_BLENDING` [S1, unofficial].
   * Bind static buffers: `UniformDataBuffer`, `TriangleKeyValueBuffer`, `TriangleCounterBuffer` (optional), `TriangleVertexPositionBuffer`, `TriangleTetIndexBuffer`, `TetIndexBuffer`, `TetVertexPositionBuffer` [S1, unofficial].
   * Set indirect draw buffer to `DrawIndirectBuffer` with size `sizeof(VkDrawIndirectCommand)` and indirect draw count 1 [S1, unofficial].
   * Execute the indirect draw command [S1, unofficial].

**Parameters:**
* `BLOCK_SIZE`: 256 threads per workgroup [S1, unofficial].
* `sortingAlgorithm`: Enum determining sorting method; defaults to `CPU_STD_SORT` if `FUCHSIA_RADIX_SORT` is unavailable [S1, unofficial].
* `radixSortVkTarget`: Auto-detected target for `FUCHSIA_RADIX_SORT` based on physical device and subgroup properties [S1, unofficial].
* `blendMode`: `BACK_TO_FRONT_PREMUL_ALPHA` [S1, unofficial].

**Edge Cases and Pitfalls:**
* **Boundaries:** The maximum number of triangles and vertices is pre-allocated based on the maximum possible count (`numCells * 3`) [S1, unofficial].
* **Precision:** The use of `glm::vec4` for vertex positions suggests 32-bit floating point precision [G]. Ensure depth computation handles potential precision issues near tetrahedron boundaries [G].
* **Degenerate Input:** If `tetMesh->getNumCells()` is zero, the pipeline should handle empty dispatches gracefully [G].

## Subtopics

### [Adjoint Intersect Raster Pass](7927-adjoint-intersect-raster-pass.md)
A backward pass that computes gradients for the forward tetrahedral rasterization to enable differentiable volume rendering. It sets apart from the forward pass by operating in a fragment shader with interlock mechanisms to safely accumulate gradient contributions, whereas the forward pass computes color accumulation. Choose it when gradients with respect to input parameters are required for differentiable training [G].

## Operating characteristics

## Background
Originated in the DiffTetVR project, a differentiable volume renderer for tetrahedral meshes [S1, unofficial]. The method leverages Vulkan for low-level GPU control and utilizes libraries like OpenVolumeMesh for mesh handling and Fuchsia Radix Sort for efficient sorting [S1, unofficial].

## Sources
[S1] GitHub - chrismile/DiffTetVR: Differentiable volume renderer for ..., https://github.com/chrismile/DiffTetVR, fetched 2026-10-28 (unofficial)
