# Multi-Threaded Tile-Based Rendering

Also called: threaded rasterization

## Overview
Multi-Threaded Tile-Based Rendering is a CPU-based parallelization strategy that divides the screen height into horizontal tiles, assigning each tile to a separate thread to parallelize the rasterization workload [S1, unofficial]. It differs from its sibling [Tile-Based Alpha Compositing](1215-tile-based-alpha-compositing.md) by focusing on the rasterization phase itself rather than just the compositing of pre-rasterized fragments, and from [Screen Parallel Rendering](5725-screen-parallel-rendering.md) by dividing the screen into tiles processed by threads on a single machine rather than subscreens distributed across multiple processors [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** Screen height ($H$), screen width ($W$), and the number of available hardware threads.
*   **Output:** A fully rasterized framebuffer.

**Data Structures:**
*   **Tile Struct:** Contains four integer fields defining the rectangular region of the screen to render: `x_min`, `x_max`, `y_min`, and `y_max` [S1, unofficial].

**Algorithm Steps:**
1.  **Determine Thread Count:** Calculate the number of threads ($T$) as the maximum of 1 and the system's reported hardware concurrency. If `std::thread::hardware_concurrency()` returns 0, $T$ is set to 1 to prevent division by zero or empty thread vectors [S1, unofficial].
2.  **Calculate Tile Height:** Compute the height of each tile ($h_{tile}$) as $H / T$ [S1, unofficial].
3.  **Initialize Threads:** For each thread index $i$ from 0 to $T-1$:
    *   Calculate the vertical range for the tile.
    *   Set `y_min` = $i \times h_{tile}$.
    *   Set `y_max` = $(i == T-1) ? H-1 : (i+1) \times h_{tile} - 1$.
    *   Set `x_min` = 0 and `x_max` = $W-1$ (assuming full width per tile).
    *   Launch a worker thread executing the `renderTile` function with this tile's coordinates.
4.  **Join Threads:** Wait for all worker threads to complete [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import threading
import os

def render_tile(tile):
    """
    Renders the specific tile region.
    Implementation details of rasterization are assumed to be handled here.
    """
    # Placeholder for actual rasterization logic within the bounds
    # tile.x_min, tile.x_max, tile.y_min, tile.y_max
    pass

def multi_threaded_tile_rendering(screen_height, screen_width):
    # Step 1: Determine number of threads
    hw_concurrency = os.cpu_count()
    if hw_concurrency is None or hw_concurrency == 0:
        threads_count = 1
    else:
        threads_count = hw_concurrency

    # Step 2: Calculate tile height
    tile_height = screen_height // threads_count

    threads = []

    # Step 3: Create and launch threads
    for i in range(threads_count):
        y_min = i * tile_height
        # Last tile extends to the end of the screen to handle non-divisible heights
        if i == threads_count - 1:
            y_max = screen_height - 1
        else:
            y_max = (i + 1) * tile_height - 1
        
        tile = {
            'x_min': 0,
            'x_max': screen_width - 1,
            'y_min': y_min,
            'y_max': y_max
        }
        
        thread = threading.Thread(target=render_tile, args=(tile,))
        threads.append(thread)
        thread.start()

    # Step 4: Join all worker threads
    for thread in threads:
        thread.join()

```

**Parameters:**
*   `threads_count`: Typically equal to `std::thread::hardware_concurrency()` [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Zero Hardware Concurrency:** If the system reports 0 cores, the algorithm defaults to 1 thread to avoid runtime errors [S1, unofficial].
*   **Non-Divisible Heights:** The last tile must explicitly extend to `screen_height - 1` to ensure all pixels are covered when $H$ is not evenly divisible by $T$ [S1, unofficial].

## Subtopics

### Tile-Based Rasterization
A GPU-specific variant where one CUDA thread-block is launched per tile, with one thread per pixel (e.g., 16×16 threads) [S2]. It differs from the CPU approach by utilizing shared memory for splat loading. The process ping-pongs between two phases: 128 threads load 128 splats into shared memory, then all 256 threads iterate through those splats to blend them in front-to-back order [S2].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of pixels and splats, parallelized by $T$ threads.
*   **Memory Complexity:** Low overhead per thread, storing only tile coordinates and local rasterization state.
*   **Strengths:** Zero synchronization between tiles during rasterization; no atomics or locks required; effective on multi-core CPUs [S1, unofficial].
*   **Limitations:** Requires careful handling of tile boundaries; performance depends on load balancing if tiles have varying complexity (though uniform height tiles assume uniform work).
*   **When to use:** CPU-based rendering where parallelism is needed without the complexity of GPU shared memory management.

## Background
This method is described in the context of CPU rasterizers, specifically referenced in the `whudsontosh/CPU-Rasterizer` repository [S1, unofficial]. It serves as a foundational parallelization technique for tiled rendering on multi-core architectures.

## Sources
[S1] whudsontosh/CPU-Rasterizer, https://github.com/whudsontosh/CPU-Rasterizer, fetched 2026-10-27 (unofficial)
[S2] Splatshop: Efficiently Editing Large Gaussian Splat Models, https://doi.org/10.1111/cgf.70214, fetched 2026-10-31
