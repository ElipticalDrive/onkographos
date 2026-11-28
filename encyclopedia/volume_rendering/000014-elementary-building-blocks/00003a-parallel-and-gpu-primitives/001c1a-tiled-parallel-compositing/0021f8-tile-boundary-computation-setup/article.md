# Tile Boundary Computation Setup

Also called: Tile Boundary Buffer Setup, Tile Boundary Generation

## Overview
This method initializes the data structures required for tiled parallel compositing by computing the spatial extent (boundaries) of each tile within the framebuffer [S1, unofficial]. It allocates and prepares a storage buffer for tile boundary indices, applies necessary GPU pipeline barriers, and dispatches a compute shader to populate the boundaries [S1, unofficial].

## How it works
**Inputs:**
- Swapchain extent (width $W$, height $H$) in pixels [S1, unofficial].
- `numInstances` representing the number of data elements or primitives to process [S1, unofficial].

**Outputs:**
- `tileBoundaryBuffer`: A storage buffer containing boundary indices for each tile [S1, unofficial].

**Data Structures:**
- `tileBoundaryBuffer`: A storage buffer sized $T_{tilesX} \times T_{tilesY} \times 2 \times \text{sizeof(uint32_t)}$ [S1, unofficial].
  - Layout: For each tile $(i, j)$, the buffer stores two `uint32_t` values with a stride of 2 uint32s per tile [S1, unofficial].
  - Total size: $T_{tilesX} \times T_{tilesY} \times 8$ bytes [S1, unofficial].
  - The exact semantic meaning of the two values per tile is implied by the name "boundary" but not explicitly defined in the source beyond the storage size [S1, unofficial].

**Steps:**
1. **Calculate Tile Grid Dimensions:** Compute the number of tiles in the X and Y directions using ceiling division: $T_{tilesX} = (W + 15) / 16$ and $T_{tilesY} = (H + 15) / 16$ [S1, unofficial].
2. **Allocate Buffer:** Allocate `tileBoundaryBuffer` with size $T_{tilesX} \times T_{tilesY} \times 2 \times \text{sizeof(uint32_t)}$ [S1, unofficial].
3. **Initialize Buffer:** Zero out `tileBoundaryBuffer` using `fillBuffer` [S1, unofficial].
4. **Synchronization:** Apply a pipeline barrier transitioning `tileBoundaryBuffer` from `eTransfer` to `eComputeShader` access [S1, unofficial].
5. **Bind Resources:** Bind the sorted key buffer (`sortKBufferEven`) and `tileBoundaryBuffer` to the compute pipeline descriptor set [S1, unofficial].
6. **Dispatch Compute Shader:** Bind `tileBoundaryPipeline` and dispatch with `numInstances` as a push constant [S1, unofficial].
7. **Dispatch Size:** Calculate workgroup count as $(numInstances + 255) / 256$ workgroups [S1, unofficial].
8. **Final Synchronization:** Apply a compute write/read barrier on `tileBoundaryBuffer` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def setup_tile_boundaries(swapchain_width, swapchain_height, num_instances):
    # 1. Calculate tile grid dimensions
    tile_x = (swapchain_width + 15) // 16
    tile_y = (swapchain_height + 15) // 16

    # 2. Allocate tile boundary storage buffer
    buffer_size = tile_x * tile_y * 2 * 4  # 2 uint32_t per tile
    tile_boundary_buffer = [0] * buffer_size

    # 3. Initialize buffer using fillBuffer
    # (Buffer is already zero-initialized in Python list allocation)

    # 4. Apply pipeline barrier from eTransfer to eComputeShader for tileBoundaryBuffer
    # (Synchronization step: ensures transfer writes are visible to compute shaders)

    # 5. Bind sorted key buffer (sortKBufferEven) and tile boundary buffer to compute pipeline descriptor set
    # (Descriptor binding step: maps buffers to shader resource slots)

    # 6. Bind tileBoundaryPipeline and dispatch with numInstances as push constant
    # Workgroup size is 256 threads
    num_workgroups = (num_instances + 255) // 256
    # dispatch(num_workgroups, 1, 1)

    # 7. Apply compute write/read barrier on tileBoundaryBuffer
    # (Synchronization step: ensures compute writes are visible to subsequent reads)

    return tile_boundary_buffer, tile_x, tile_y, num_workgroups
```

**Edge Cases and Pitfalls:**
- If swapchain dimensions are not multiples of 16, the last tiles in X and Y are partially filled, requiring the boundary computation to handle them correctly [S1, unofficial].
- Integer division for tile count calculation must use ceiling logic to cover partial tiles at the edges [S1, unofficial].

## Subtopics

### [Coarse Raster](2807-coarse-raster.md)
The Coarse Rasterizer performs triangle setup and emits a conservative list of all triangles touching each tile in a tiled volume rendering pipeline. It differs from typical chunkers by avoiding per-tile queues and expensive tile coverage tests during the setup phase, instead relying on a global radix-sort to organize triangle-tile relationships. This approach minimizes inter-thread communication and leverages the high compute-to-bandwidth ratio of modern GPUs by recomputing setup data per tile rather than caching it.

### [Tile-Based Neural Material Classification and Sorting](16512-tile-based-neural-material-classification-and-sorting.md)
This method is a two-pass compute shader pipeline that classifies 8x4 pixel tiles based on the neural material (MLP) index used by their pixels, separating them into uniform tiles (single MLP) and mixed tiles (multiple MLPs), then sorts mixed tiles to ensure uniform MLP usage per tile for efficient hardware matrix multiplication. It solves the problem of thread divergence in cooperative vector matrix multiplication by ensuring all pixels in a same workgroup use the same base matrix. Unlike [Coarse Raster](2807-coarse-raster.md), which focuses on early visibility culling, this method operates on the output of the Visibility pass to reorganize pixel data for the Lighting pass.

### Per-Tile Upper Bound Calculation
Fix a tile $t$ and define its pixel space rectangle $\Omega_t = [x_{\min}, x_{\max}) \times [y_{\min}, y_{\max})$ [S2, unofficial]. Index four slab pairs by $i \in \{0, 1, 2, 3\}$ with normals $n_i = \epsilon_{i,0}c_0 + \epsilon_{i,1}c_1 + \epsilon_{i,2}c_2$, where $\epsilon_{i,0}=+1$ and $\epsilon_{i,1}, \epsilon_{i,2} \in \{+1, -1\}$ [S2, unofficial]. For each slab pair with $n_{i,z} \neq 0$, solve $|n_i \cdot (\Delta x, \Delta y, \tau)| \le 1$ for $\tau$ to get bounds $s_{i,lo}(x,y)$ and $s_{i,hi}(x,y)$ [S2, unofficial]. Compute $m_i(x,y)$ and $h_i$ from these bounds [S2, unofficial].

## Operating characteristics
- **Memory Complexity:** $O(T_{tilesX} \times T_{tilesY})$ for the `tileBoundaryBuffer` [S1, unofficial].
- **Time Complexity:** The setup phase is dominated by the compute shader dispatch, which scales proportionally to `numInstances` [S1, unofficial].

## Background
This method is part of a Vulkan-based Gaussian Splatting implementation [S1, unofficial]. It serves as a specific step in the rendering pipeline that prepares the data structures for tiled processing [S1, unofficial]. The use of Vulkan compute shaders and specific buffer management techniques reflects modern GPU programming practices for high-performance rendering [S1, unofficial].

## Sources
[S1] Show HN: 3DGS.cpp – performant, cross platform Gaussian Splatting with Vulkan, https://github.com/shg8/3DGS.cpp, fetched 2026-10-26 (unofficial)
[S2] XPR: An Extensible Cross-Platform Point-Based Differentiable Renderer, https://doi.org/10.48550/arxiv.2606.11529, fetched 2026-10-26 (unofficial)
