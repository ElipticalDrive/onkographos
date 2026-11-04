# Tile Boundary Computation Setup

Also called: Tile Boundary Buffer Setup, Tile Boundary Generation

## Overview
This method initializes the data structures required for tiled parallel compositing by computing which triangles (or splats) overlap with each tile in the output framebuffer. It solves the problem of mapping spatial geometry to disjoint processing units without synchronization. It differs from its siblings by relying on a global radix-sort to organize triangle-tile relationships rather than per-tile queues or expensive coverage tests during setup [S1, unofficial].

## How it works
**Inputs:**
- Swapchain width and height (resolution).
- Number of instances (triangles/splats) to process.
- Sorted key buffer (`sortKBufferEven`) containing tile indices for each instance.

**Outputs:**
- `tileBoundaryBuffer`: A storage buffer storing boundary indices for each tile.

**Data Structures:**
- `tileBoundaryBuffer`: A storage buffer of size `tileX * tileY * 2 * sizeof(uint32_t)`. It stores two `uint32_t` values per tile, likely representing start and end indices into the sorted instance list [S1, unofficial].

**Parameters:**
- `tileX`: Number of tiles in the x-direction, calculated as `(width + 15) / 16` [S1, unofficial].
- `tileY`: Number of tiles in the y-direction, calculated as `(height + 15) / 16` [S1, unofficial].
- `numInstances`: The total number of geometry instances (triangles/splats) [S1, unofficial].

**Steps:**
1.  **Calculate Tile Grid:** Compute `tileX` and `tileY` using ceiling division by 16.
    $$ tileX = \lceil \frac{width}{16} \rceil, \quad tileY = \lceil \frac{height}{16} \rceil $$
2.  **Allocate Buffer:** Allocate `tileBoundaryBuffer` with size `tileX * tileY * 2 * sizeof(uint32_t)` [S1, unofficial].
3.  **Initialize Buffer:** Zero out `tileBoundaryBuffer` using a fill operation [S1, unofficial].
4.  **Barrier:** Apply a pipeline barrier transitioning `tileBoundaryBuffer` from `eTransfer` to `eComputeShader` access [S1, unofficial].
5.  **Bind Descriptors:** Bind `sortKBufferEven` and `tileBoundaryBuffer` to the compute pipeline descriptor set [S1, unofficial].
6.  **Dispatch:** Bind `tileBoundaryPipeline` and dispatch compute shaders.
    -   Push constant: `numInstances`.
    -   Workgroup size: 256 threads per workgroup.
    -   Number of workgroups: `(numInstances + 255) / 256` [S1, unofficial].
7.  **Barrier:** Apply a compute read/write barrier on `tileBoundaryBuffer` to ensure completion before subsequent reads [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def setup_tile_boundaries(context, width, height, num_instances, sort_k_buffer_even):
    # 1. Calculate tile grid dimensions
    tile_x = (width + 15) // 16
    tile_y = (height + 15) // 16
    
    # 2. Allocate tile boundary buffer
    # Size: tileX * tileY * 2 * sizeof(uint32_t)
    tile_boundary_buffer = context.create_storage_buffer(
        size=tile_x * tile_y * 2 * 4, 
        name="tileBoundaryBuffer"
    )
    
    # 3. Zero out the buffer
    context.fill_buffer(tile_boundary_buffer, 0)
    
    # 4. Apply pipeline barrier: Transfer -> ComputeShader
    context.apply_pipeline_barrier(
        buffer=tile_boundary_buffer,
        src_access=vk.AccessFlagBits.eTransferWrite,
        dst_access=vk.AccessFlagBits.eShaderWrite,
        src_stage=vk.PipelineStageFlagBits.eTransfer,
        dst_stage=vk.PipelineStageFlagBits.eComputeShader
    )
    
    # 5. Bind descriptors
    descriptor_set = context.create_descriptor_set()
    descriptor_set.bind_buffer(0, vk.DescriptorType.eStorageBuffer, vk.ShaderStageFlagBits.eCompute, sort_k_buffer_even)
    descriptor_set.bind_buffer(1, vk.DescriptorType.eStorageBuffer, vk.ShaderStageFlagBits.eCompute, tile_boundary_buffer)
    descriptor_set.build()
    
    # 6. Dispatch compute pipeline
    # Pipeline: tileBoundaryPipeline
    # Push constant: num_instances
    # Workgroup size: 256
    num_workgroups = (num_instances + 255) // 256
    
    context.bind_pipeline(tile_boundary_pipeline)
    context.bind_descriptor_set(0, descriptor_set)
    context.push_constant(num_instances)
    context.dispatch_compute(num_workgroups, 1, 1)
    
    # 7. Apply compute barrier
    context.apply_pipeline_barrier(
        buffer=tile_boundary_buffer,
        src_access=vk.AccessFlagBits.eShaderWrite,
        dst_access=vk.AccessFlagBits.eShaderRead | vk.AccessFlagBits.eShaderWrite,
        src_stage=vk.PipelineStageFlagBits.eComputeShader,
        dst_stage=vk.PipelineStageFlagBits.eComputeShader
    )
```

**Edge Cases:**
-   **Zero Instances:** If `num_instances` is 0, the dispatch should handle this gracefully or be skipped [G].
-   **Boundary Precision:** Ceiling division ensures full coverage of the swapchain dimensions, even if not perfectly divisible by 16 [S1, unofficial].

## Subtopics

### [Coarse Raster](2807-coarse-raster.md)
Performs triangle setup and emits a conservative list of triangles touching each tile. It differs by using global radix-sort to avoid per-tile queues and expensive coverage tests during setup, minimizing inter-thread communication.

## Operating characteristics
-   **Time Complexity:** Linear with respect to the number of instances for the dispatch, dominated by the preceding radix sort [S1, unofficial].
-   **Memory Complexity:** $O(tileX \times tileY)$ for the boundary buffer [S1, unofficial].
-   **Strengths:** Zero synchronization between tiles during compositing; leverages GPU compute parallelism [S1, unofficial].
-   **Limitations:** Requires a sorted input list; setup cost is non-trivial for very small tile counts or low instance counts [G].
-   **When to Use:** When compositing large numbers of overlapping primitives on GPU where atomic operations are expensive or undesirable [S1, unofficial].

## Background
This method is part of the Vulkan-based Gaussian Splatting implementation in `3DGS.cpp`, utilizing Vulkan compute shaders for parallel processing [S1, unofficial].

## Sources
[S1] Show HN: 3DGS.cpp – performant, cross platform Gaussian Splatting with Vulkan, https://github.com/shg8/3DGS.cpp, fetched 2026-10-26 (unofficial)
