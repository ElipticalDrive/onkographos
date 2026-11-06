# WebSplatter


## Overview
WebSplatter is a WebGPU-based renderer for 3D Gaussian Splatting (3DGS) designed to achieve high-performance, cross-device compatibility in web browsers [S1, unofficial]. It solves the problem of unstable and memory-intensive Gaussian Splatting implementations on the web by replacing CPU-based and spin-wait-based sorting with a wait-free GPU radix sort and optimizing memory access patterns [S1, unofficial]. Unlike sibling methods that rely on WebGL or naive WebGPU ports, WebSplatter ensures stability on low-end mobile hardware and avoids the busy-waiting pitfalls of Apple M-series chips, maintaining visual fidelity comparable to native CUDA implementations [S1, unofficial].

## How it works
WebSplatter processes 3D Gaussian Splatting data through three main stages: pre-processing, sorting, and rendering. The method relies on the standard 3DGS representation where each Gaussian is defined by a mean position $\mu$, a covariance matrix $\Sigma$ (parameterized by a 6-element rotation/scaling vector and a scale), an opacity $\alpha$, and spherical harmonic coefficients for color [S1, unofficial].

### Inputs and Outputs
- **Input**: A set of $N$ 3D Gaussians, each defined by $\mu \in \mathbb{R}^3$, $\Sigma \in \mathbb{R}^{3 \times 3}$ (or its parameterization), $\alpha \in [0, 1]$, and color coefficients. Camera parameters (view and projection matrices) are also provided.
- **Output**: A 2D image rendered via WebGL/WebGPU rasterization.

### Data Structures
The implementation maps volumetric data processing onto parallel execution units, adhering to the shared principles of GPU web renderers [S1, unofficial].
1.  **Gaussian Attributes**: Stored in structured buffers or textures optimized for coalesced memory access. The covariance matrix is typically transformed into screen space using the Jacobian of the projection matrix.
2.  **Sorting Keys**: For alpha-blended rendering, splats must be sorted by depth. WebSplatter uses a GPU-based radix sort. The key is the depth of the Gaussian center projected into screen space.
3.  **Atomic Counters**: Used in the pre-processing stage to handle variable numbers of splats per tile or bin, avoiding fixed-size buffers that waste memory [S1, unofficial].

### Numbered Steps

1.  **Pre-processing (Screen-space Projection and Tiling)**:
    -   For each Gaussian $i$, compute the screen-space center $c_i$ and the projected covariance matrix $\Sigma'_i$ using the view-projection matrix $J$ and the 3D-to-2D Jacobian [S1, unofficial].
    -   Determine the bounding box of each Gaussian in screen space.
    -   Assign each Gaussian to a tile (e.g., $16 \times 16$ pixels) based on its center.
    -   Use atomic operations to count the number of splats per tile and compute offsets for a compacted output buffer. This step is critical for low-end GPUs where memory throughput is limited [S1, unofficial].

2.  **Sorting (Wait-free GPU Radix Sort)**:
    -   Sort the splats within each tile by depth. WebSplatter implements a wait-free radix sort to avoid the performance penalties of spin-waiting on architectures like Apple M1 [S1, unofficial].
    -   The sort is performed entirely on the GPU using parallel reduction and scan operations.
    -   This step replaces CPU-based sorting (e.g., in `antimatter15/splat`) and spin-wait-based sorts (e.g., in `KeKsBoTer/web-splat`) [S1, unofficial].

3.  **Rendering (Rasterization)**:
    -   Iterate through the sorted splats.
    -   For each splat, compute its contribution to the pixels within its bounding box.
    -   Perform alpha blending in screen space. The standard 3DGS compositing equation is used:
        $$ C = \sum_{i=1}^{N} c_i \alpha_i \prod_{j=1}^{i-1} (1 - \alpha_j) $$
        where $c_i$ is the color and $\alpha_i$ is the opacity of the $i$-th splat in sorted order [S1, unofficial].
    -   Write the final color to the framebuffer.

### Pseudocode
The following pseudocode outlines the core logic, focusing on the wait-free sorting and rendering pipeline. It assumes standard WebGPU compute shader capabilities.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pre-processing: Compute screen-space attributes and tile assignment
def preprocess_gaussians(gaussians, view_proj_matrix, tile_size):
    # Output: sorted_gaussians, tile_offsets
    n = len(gaussians)
    tile_counts = [0] * (num_tiles)
    
    # Step 1: Project and assign to tiles
    for i in range(n):
        g = gaussians[i]
        # Project 3D mean to screen space
        screen_center = project(g.mean, view_proj_matrix)
        # Compute projected covariance
        jacobian = compute_jacobian(g.mean, view_proj_matrix)
        projected_cov = jacobian @ g.covariance @ jacobian.T
        
        # Determine tile index
        tile_idx = (screen_center.y // tile_size) * num_tiles_per_row + (screen_center.x // tile_size)
        
        # Atomic increment for tile count
        tile_counts[tile_idx] += 1
        
        # Store projected attributes for sorting
        g.screen_center = screen_center
        g.projected_cov = projected_cov
        g.depth = screen_center.z # Depth for sorting
        
    # Step 2: Compute tile offsets using prefix sum (scan)
    tile_offsets = prefix_sum(tile_counts)
    
    # Step 3: Compact gaussians into tiles
    # Use atomic counters to assign positions in the compacted buffer
    current_positions = [0] * num_tiles
    compacted_gaussians = [None] * n
    
    for i in range(n):
        g = gaussians[i]
        tile_idx = get_tile_index(g.screen_center, tile_size)
        pos = atomic_add(current_positions[tile_idx], 1)
        compacted_gaussians[tile_idx * max_splats_per_tile + pos] = g
        
    return compacted_gaussians, tile_offsets

# Sorting: Wait-free Radix Sort
def wait_free_radix_sort(compacted_gaussians, tile_offsets):
    # WebSplatter uses a GPU-based radix sort that does not rely on spin-waiting
    # This avoids busy-waiting on Apple M1 chips where workgroup execution order is not guaranteed
    # Implementation details of the radix sort are abstracted here as a standard GPU primitive
    # The sort key is the depth of the gaussian center
    for tile in range(num_tiles):
        start = tile_offsets[tile]
        end = tile_offsets[tile + 1] if tile + 1 < num_tiles else len(compacted_gaussians)
        if start < end:
            # Sort gaussians[start:end] by depth using wait-free radix sort
            sort_by_depth(compacted_gaussians[start:end])
    return compacted_gaussians

# Rendering: Alpha-blended Rasterization
def render(compacted_gaussians, tile_offsets, framebuffer, tile_size):
    for tile in range(num_tiles):
        start = tile_offsets[tile]
        end = tile_offsets[tile + 1] if tile + 1 < num_tiles else len(compacted_gaussians)
        
        # Initialize framebuffer pixels in this tile to transparent
        clear_tile(framebuffer, tile, tile_size)
        
        # Iterate through sorted splats in this tile
        for i in range(start, end):
            g = compacted_gaussians[i]
            # Compute contribution to pixels in the tile
            # Use the projected covariance to determine the ellipse
            # Perform alpha blending
            blend_splat(framebuffer, g, tile, tile_size)
            
    return framebuffer
```

### Parameters and Typical Values
-   **Tile Size**: $16 \times 16$ pixels is a common choice, balancing overhead and coalesced access [S1, unofficial].
-   **Radix Sort Bits**: Typically 8-bit or 10-bit radix sort passes are used for depth sorting [S1, unofficial].
-   **Scene Complexity**: Tested on scenes ranging from 0.3M splats (van gogh) to 5.8M splats (garden) [S1, unofficial].

### Edge Cases and Pitfalls
-   **Apple M1 Spin-waiting**: Spin-wait-based sorting (e.g., in `KeKsBoTer/web-splat`) performs poorly on Apple M1 chips due to lack of guarantees for workgroup execution order, leading to significant busy-waiting. WebSplatter's wait-free design avoids this [S1, unofficial].
-   **Memory Limits on Mobile**: Baselines like `mkkellogg/GaussianSplatsD` crash on iPhone 15 Pro Max due to excessive memory requirements. WebSplatter's optimized memory access prevents this [S1, unofficial].
-   **Firefox WebGPU Instability**: Firefox may block WebGPU or report unreliable timestamp queries. WebSplatter is tested on Chrome/Edge where WebGPU is stable [S1, unofficial].
-   **Pre-processing Bottleneck on Low-end GPUs**: On low-end GPUs (e.g., Nvidia MX350), pre-processing can consume up to 47.2% of frame time due to limited memory throughput. WebSplatter minimizes this by efficient tiling and atomic operations [S1, unofficial].

## Operating characteristics
-   **Time Complexity**: The sorting stage is $O(N \log N)$ or $O(N)$ depending on the radix sort implementation, but WebSplatter's GPU-based sort is highly optimized. The rendering stage is $O(N)$ per pixel, but with early-z culling and tiling, it is efficient.
-   **Memory Complexity**: $O(N)$ for storing Gaussian attributes and intermediate buffers. WebSplatter's memory usage is lower than baselines, allowing it to run on devices with limited VRAM [S1, unofficial].
-   **Performance**:
    -   On Nvidia RTX 3070, WebSplatter achieves a median total frame time of 9.497 ms on the garden scene, a 1.52× speedup over the fastest competing WebGPU implementation [S1, unofficial].
    -   On MacBook Air M4, it is 1.06× faster; on Redmi K70 Pro, 1.18× faster; and on Intel NUC iGPU, 2.26× faster [S1, unofficial].
    -   On MacBook Pro M1, it is over 4.5× faster than `KeKsBoTer/web-splat` due to the avoidance of spin-waiting [S1, unofficial].
-   **Stability**: WebSplatter shows low variance in frame times. On MacBook Air M4 rendering the garden scene (5.8M splats), the standard deviation is 22.303 ms and P99 latency is 116.851 ms, mitigating micro-stuttering [S1, unofficial].
-   **Scalability**: On Nvidia MX350, scaling from 0.3M to 5.8M splats increases frame time by ~143 ms, compared to >280 ms for `KeKsBoTer/web-splat`, yielding a 2.1× speedup on the largest scene [S1, unofficial].
-   **Bottlenecks**:
    -   High-end GPUs (RTX 3070, M-series, Redmi K70 Pro) are render-bound, with rasterization consuming 51.8% to 62.6% of frame time [S1, unofficial].
    -   Low-end GPUs (MX350, Intel NUC iGPU) are pre-process-bound, consuming up to 47.2% due to limited memory throughput [S1, unofficial].
    -   The system never becomes sort-bound across tested hardware [S1, unofficial].
-   **Strengths**: Cross-device compatibility, stability on low-end hardware, avoidance of spin-waiting pitfalls, and visual fidelity comparable to native CUDA [S1, unofficial].
-   **Limitations**: Requires WebGPU support, which is not universally available or stable across all browsers (e.g., Firefox issues) [S1, unofficial].

## Background
WebSplatter was introduced in the paper "WebSplatter: Enabling Cross-Device Efficient Gaussian Splatting in Web Browsers via WebGPU" [S1, unofficial]. It addresses the limitations of earlier web-based 3DGS renderers, which suffered from poor performance on mobile devices, instability on Apple Silicon, and high memory usage. By leveraging WebGPU's parallel compute capabilities and implementing a wait-free sorting algorithm, WebSplatter enables interactive 3DGS rendering in web browsers across a wide range of hardware [S1, unofficial].

## Sources
[S1] WebSplatter: Enabling Cross-Device Efficient Gaussian Splatting in Web Browsers via WebGPU, https://doi.org/10.48550/arxiv.2602.03207, fetched 2026-10-26 (unofficial)
