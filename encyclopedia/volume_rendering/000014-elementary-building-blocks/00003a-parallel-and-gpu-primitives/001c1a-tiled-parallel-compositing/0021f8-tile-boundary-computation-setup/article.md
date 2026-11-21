# Tile Boundary Computation Setup

Also called: Tile Boundary Buffer Setup, Tile Boundary Generation

## Overview
This method initializes the data structures required for tiled parallel compositing by computing the spatial extent (boundaries) of each tile within the framebuffer. It solves the problem of determining which input primitives or data elements contribute to which output tiles, enabling subsequent parallel processing stages to operate on disjoint subsets of the image without synchronization. It differs from sibling methods like [Coarse Raster](2807-coarse-raster.md) by focusing on the setup of the boundary metadata itself rather than the rasterization of geometry, and from [Dynamic Load Balancing via Tile Queue](1763-dynamic-load-balancing-via-tile-queue.md) by being a static setup phase rather than a runtime scheduling mechanism.

## How it works
**Inputs:**
- Swapchain extent (width $W$, height $H$) in pixels.
- Tile dimensions $T_x = 16$, $T_y = 16$ (fixed in source [S1, unofficial]).
- Sorted key buffer (`sortKBufferEven`) containing pre-sorted data indices.

**Outputs:**
- `tileBoundaryBuffer`: A storage buffer containing boundary indices for each tile.

**Data Structures:**
- `tileBoundaryBuffer`: A storage buffer of size $N_{tiles} \times 2 \times \text{sizeof(uint32_t)}$.
  - Layout: For each tile $(i, j)$, the buffer stores two `uint32_t` values. The stride is 2 uint32s per tile. The exact semantic meaning of the two values (e.g., start/end indices) is implied by the name "boundary" but not explicitly defined in the source beyond the storage size [S1, unofficial].
  - Total size: $T_{tilesX} \times T_{tilesY} \times 8$ bytes.

**Steps:**
1.  **Calculate Tile Grid Dimensions:**
    Compute the number of tiles in the X and Y directions using ceiling division:
    $$
    T_{tilesX} = \left\lceil \frac{W}{16} \right\rceil = \frac{W + 15}{16}
    $$
    $$
    T_{tilesY} = \left\lceil \frac{H}{16} \right\rceil = \frac{H + 15}{16}
    $$
    [S1, unofficial]

2.  **Allocate Buffer:**
    Allocate `tileBoundaryBuffer` with size $T_{tilesX} \times T_{tilesY} \times 2 \times \text{sizeof(uint32_t)}$. [S1, unofficial]

3.  **Initialize Buffer:**
    Zero out `tileBoundaryBuffer` using a fill operation. [S1, unofficial]

4.  **Synchronization:**
    Apply a pipeline barrier transitioning `tileBoundaryBuffer` from `eTransfer` to `eComputeShader` access. [S1, unofficial]

5.  **Bind Resources:**
    Bind `sortKBufferEven` (sorted key buffer) and `tileBoundaryBuffer` to the compute pipeline descriptor set. [S1, unofficial]

6.  **Dispatch Compute Shader:**
    Bind `tileBoundaryPipeline` and dispatch workgroups.
    - Push constant: `numInstances` (likely related to the number of data elements or tiles, source does not specify exact derivation).
    - Workgroup size: 256 threads.
    - Number of workgroups:
      $$
      N_{workgroups} = \left\lceil \frac{N_{instances}}{256} \right\rceil = \frac{N_{instances} + 255}{256}
      $$
      [S1, unofficial]

7.  **Final Synchronization:**
    Apply a compute read/write barrier on `tileBoundaryBuffer` to ensure completion before subsequent reads. [S1, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def setup_tile_boundaries(context, swapchain_width, swapchain_height, num_instances):
    # Step 1: Calculate tile grid dimensions
    tile_x = (swapchain_width + 15) // 16
    tile_y = (swapchain_height + 15) // 16
    
    # Step 2: Allocate buffer
    # Size: tile_x * tile_y * 2 * 4 bytes
    buffer_size = tile_x * tile_y * 2 * 4
    tile_boundary_buffer = context.create_storage_buffer(buffer_size)
    
    # Step 3: Initialize buffer
    context.fill_buffer(tile_boundary_buffer, 0)
    
    # Step 4: Pipeline barrier
    context.apply_pipeline_barrier(
        buffer=tile_boundary_buffer,
        src_access=vk.AccessFlagBits.eTransferWrite,
        dst_access=vk.AccessFlagBits.eShaderWrite,
        src_stage=vk.PipelineStageFlagBits.eTransfer,
        dst_stage=vk.PipelineStageFlagBits.eComputeShader
    )
    
    # Step 5: Bind resources
    descriptor_set = context.create_descriptor_set()
    descriptor_set.bind_buffer(0, vk.DescriptorType.eStorageBuffer, context.sort_k_buffer_even)
    descriptor_set.bind_buffer(1, vk.DescriptorType.eStorageBuffer, tile_boundary_buffer)
    descriptor_set.build()
    
    # Step 6: Dispatch
    # Assuming num_instances is passed as a push constant
    num_workgroups = (num_instances + 255) // 256
    context.bind_pipeline(tile_boundary_pipeline)
    context.bind_descriptor_set(descriptor_set)
    context.push_constant(num_instances)
    context.dispatch(num_workgroups, 1, 1)
    
    # Step 7: Final barrier
    context.apply_pipeline_barrier(
        buffer=tile_boundary_buffer,
        src_access=vk.AccessFlagBits.eShaderWrite,
        dst_access=vk.AccessFlagBits.eShaderRead,
        src_stage=vk.PipelineStageFlagBits.eComputeShader,
        dst_stage=vk.PipelineStageFlagBits.eComputeShader
    )
    
    return tile_boundary_buffer
```

**Edge Cases and Pitfalls:**
- **Precision:** Integer division for tile count calculation must use ceiling logic to cover partial tiles at the edges. [S1, unofficial]
- **Boundaries:** If the swapchain dimensions are not multiples of 16, the last tiles in X and Y will be partially filled. The boundary computation must handle these partial tiles correctly, though the source does not specify the exact handling logic within the shader. [S1, unofficial]
- **Synchronization:** Failure to apply the correct pipeline barriers between transfer, compute, and subsequent stages will lead to undefined behavior or race conditions. [S1, unofficial]

## Subtopics

### [Coarse Raster](2807-coarse-raster.md)
The Coarse Rasterizer performs triangle setup and emits a conservative list of all triangles touching each tile in a tiled volume rendering pipeline. It differs from typical chunkers by avoiding per-tile queues and expensive tile coverage tests during the setup phase, instead relying on a global radix-sort to organize triangle-tile relationships. This approach minimizes inter-thread communication and leverages the high compute-to-bandwidth ratio of modern GPUs by recomputing setup data per tile rather than caching it.

### [Tile-Based Neural Material Classification and Sorting](16512-tile-based-neural-material-classification-and-sorting.md)
This method is a two-pass compute shader pipeline that classifies 8x4 pixel tiles based on the neural material (MLP) index used by their pixels, separating them into uniform tiles (single MLP) and mixed tiles (multiple MLPs), then sorts mixed tiles to ensure uniform MLP usage per tile for efficient hardware matrix multiplication. It solves the problem of thread divergence in cooperative vector matrix multiplication by ensuring all pixels in a same workgroup use the same base matrix. Unlike [Coarse Raster](2807-coarse-raster.md), which focuses on early visibility culling, this method operates on the output of the Visibility pass to reorganize pixel data for the Lighting pass.

## Operating characteristics
- **Time Complexity:** The setup phase is dominated by the compute shader dispatch. The complexity is $O(N_{instances})$ for the dispatch and $O(1)$ for buffer allocation and initialization. [S1, unofficial]
- **Memory Complexity:** $O(T_{tilesX} \times T_{tilesY})$ for the `tileBoundaryBuffer`. [S1, unofficial]
- **Performance:** The method is designed for high performance on GPUs, leveraging compute shaders and efficient buffer management. The use of fixed tile sizes (16x16) simplifies the calculation and ensures uniform work distribution. [S1, unofficial]
- **Strengths:** Simple setup, low memory overhead for boundary data, efficient use of GPU compute units. [S1, unofficial]
- **Limitations:** Static tile size may not be optimal for all scene complexities. The source does not provide detailed performance metrics or scaling analysis. [S1, unofficial]
- **When to use:** When using tiled parallel compositing with fixed tile sizes and needing to quickly establish tile boundaries for subsequent processing. [S1, unofficial]
- **When not to use:** When adaptive tile sizing is required or when the scene complexity varies significantly across the image, making fixed tiles inefficient. [S1, unofficial]

## Background
This method is part of the Vulkan-based Gaussian Splatting implementation described in the source [S1, unofficial]. It is a specific step in the rendering pipeline that prepares the data structures for tiled processing. The use of Vulkan compute shaders and specific buffer management techniques reflects modern GPU programming practices for high-performance rendering. [S1, unofficial]

## Sources
[S1] Show HN: 3DGS.cpp – performant, cross platform Gaussian Splatting with Vulkan, https://github.com/shg8/3DGS.cpp, fetched 2026-10-26 (unofficial)
