# Multi-Threaded Tile-Based Rendering
Also called: threaded rasterization

## Overview
Multi-Threaded Tile-Based Rendering is a CPU-based parallelization strategy that divides the screen height into disjoint horizontal tiles, assigning each tile to a separate thread to parallelize the rasterization workload [S1, unofficial]. It builds on [Tiled Parallel Compositing](1192-tiled-parallel-compositing.md) by providing the thread management and tile decomposition logic, allowing for lock-free, zero-synchronization processing of screen regions [S1, unofficial]. Unlike [Screen Parallel Rendering](#screen-parallel-rendering), which requires every processor to hold the entire volume in memory, this method allows for more flexible memory management as tiles are processed independently [S2, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Screen height ($H$), screen width ($W$), number of available hardware threads ($N_{hw}$).
- Output: A fully rendered image (framebuffer) where each tile has been rasterized by its assigned thread.

**Data Structures:**
The core data structure is a `Tile` struct, which defines the rectangular region of the screen to be rendered by a specific thread. It contains four integer fields representing the boundaries:
- `x_min`: Left boundary (inclusive).
- `x_max`: Right boundary (exclusive or inclusive depending on implementation, typically exclusive for iteration).
- `y_min`: Top boundary (inclusive).
- `y_max`: Bottom boundary (inclusive).

**Algorithm Steps:**
1. Determine the number of threads to use. If `std::thread::hardware_concurrency()` returns 0 or is unavailable, default to 1 thread to prevent division by zero or empty thread vectors [S1, unofficial].
   $$ THREADS = \max(1, \text{std::thread::hardware\_concurrency}()) $$
2. Calculate the height of each tile. The screen height $H$ is divided by the number of threads. The last tile may handle any remainder to ensure full coverage [S1, unofficial].
   $$ tileH = \lfloor H / THREADS \rfloor $$
3. Create and launch worker threads. For each thread index $i$ from $0$ to $THREADS - 1$:
   - Define the y-range for the tile.
     $$ y_{min} = i \times tileH $$
     $$ y_{max} = \begin{cases} H - 1 & \text{if } i == THREADS - 1 \\ (i + 1) \times tileH - 1 & \text{otherwise} \end{cases} $$
   - Define the x-range. Typically, the tile spans the full width of the screen, so $x_{min} = 0$ and $x_{max} = W - 1$ [S1, unofficial].
   - Instantiate a `Tile` struct with these coordinates.
   - Launch a new thread executing the `renderTile` function, passing the `Tile` struct as an argument.
4. Join all worker threads to ensure completion before proceeding [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import threading
import math

class Tile:
    def __init__(self, x_min, x_max, y_min, y_max):
        self.x_min = x_min
        self.x_max = x_max
        self.y_min = y_min
        self.y_max = y_max

def render_tile(tile, framebuffer, scene_data):
    """
    Renders a single tile into the shared framebuffer.
    Assumes framebuffer is pre-allocated and thread-safe for writes 
    (e.g., using atomic operations or disjoint memory regions per tile).
    """
    for y in range(tile.y_min, tile.y_max + 1):
        for x in range(tile.x_min, tile.x_max + 1):
            # Perform rasterization/compositing for pixel (x, y)
            # This step is specific to the volume rendering method being used
            pixel_color = compute_pixel_color(x, y, scene_data)
            framebuffer[y][x] = pixel_color

def multi_threaded_tile_rendering(width, height, scene_data):
    # Determine thread count
    try:
        num_hw_threads = threading.active_count() # Placeholder for hardware_concurrency check
        # In real C++ std::thread::hardware_concurrency() returns unsigned int
        # If 0, treat as 1. Here we simulate the logic.
        threads = max(1, 4) # Example: assuming 4 cores if detection fails
    except:
        threads = 1

    # Calculate tile height
    tile_h = height // threads
    
    threads_list = []
    
    for i in range(threads):
        y_min = i * tile_h
        if i == threads - 1:
            y_max = height - 1
        else:
            y_max = (i + 1) * tile_h - 1
            
        # Full width for each tile
        x_min = 0
        x_max = width - 1
        
        tile = Tile(x_min, x_max, y_min, y_max)
        
        # Create and start thread
        t = threading.Thread(target=render_tile, args=(tile, framebuffer, scene_data))
        threads_list.append(t)
        t.start()
        
    # Join all threads
    for t in threads_list:
        t.join()

    return framebuffer
```

**Parameters:**
- `THREADS`: Number of parallel workers. Typical value is `std::thread::hardware_concurrency()` [S1, unofficial].
- `tileH`: Height of each tile. Determined by $H / THREADS$ [S1, unofficial].

**Edge Cases and Pitfalls:**
- If `hardware_concurrency()` returns 0, the thread count must be clamped to 1 to avoid division by zero errors when calculating `tileH` or creating an empty thread vector [S1, unofficial].
- The last tile must explicitly handle the remainder of the height division to ensure the entire screen is covered without gaps or overlaps [S1, unofficial].

## Subtopics

### Screen Parallel Rendering
A parallel scheme that divides the screen into $p$ subscreens for $p$ processors, where each processor takes responsibility for the entire ray for its subscreen [S2, unofficial]. Tasks are assigned in a cyclic manner to statically balance workloads [S2, unofficial]. It is suitable for small datasets that do not require data decomposition but requires large main memory because every processor must load the entire volume into memory [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N / T)$, where $N$ is the total number of pixels and $T$ is the number of threads, assuming uniform workload distribution.
- **Memory Complexity:** $O(W \times H)$ for the framebuffer, plus $O(1)$ per thread for tile state. Unlike object-parallel schemes, it does not require distributing the volume data across processors if the volume is shared read-only [S2, unofficial].
- **Strengths:** Zero synchronization between tiles during rendering (no atomics or locks needed for pixel writes if tiles are disjoint), simple implementation, effective on CPUs [S1, unofficial].
- **Limitations:** Load imbalance if the scene has varying complexity across the screen (e.g., occlusion varies by tile). Requires careful handling of the last tile's boundaries [S1, unofficial].
- **When to use:** CPU-based volume rendering where memory bandwidth is not the primary bottleneck and the scene fits in memory.
- **When not to use:** Very large datasets that cannot fit in memory on each processor (see [Screen Parallel Rendering](#screen-parallel-rendering) for memory constraints) or when GPU acceleration is available.

## Background
The concept of dividing the screen into subscreens for parallel processing was introduced in early parallel volume rendering schemes, specifically in the context of medical volume rendering [S2, unofficial]. The specific implementation using `std::thread` and hardware concurrency detection is documented in the `CPU-Rasterizer` repository [S1, unofficial].

## Sources
[S1] whudsontosh/CPU-Rasterizer, https://github.com/whudsontosh/CPU-Rasterizer, fetched 2026-10-27 (unofficial)
[S2] Medical Volume Rendering Techniques, https://arxiv.org/pdf/1802.07710v1, fetched 2026-10-27 (unofficial)
