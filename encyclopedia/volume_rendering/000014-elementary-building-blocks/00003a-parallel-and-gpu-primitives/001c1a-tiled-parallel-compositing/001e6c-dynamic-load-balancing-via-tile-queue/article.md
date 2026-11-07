# Dynamic Load Balancing via Tile Queue

Also called: Tile Queue Load Balancing

## Overview
Dynamic Load Balancing via Tile Queue is a parallel processing scheme that splits screen space into tiles and assigns them to processors via a shared queue to ensure high processor utilization across varying views [S1, book]. It addresses the unpredictability of isosurface extraction complexity per pixel, which static partitioning cannot handle efficiently [S1, book]. Unlike its siblings, which often rely on static spatial locality or explicit communication minimization for cell assignment, this method uses a dynamic queue with variable-granularity chunking to balance load without synchronization overhead during the rendering phase [S1, book].

## How it works
**Inputs:**
- Screen dimensions (width $W$, height $H$) [G].
- Number of processors $P$ [G].
- Tile dimensions $T_w \times T_h$ [S1, book].

**Outputs:**
- Rendered image [G].
- Processor utilization metrics [G].

**Data Structures:**
- **Tile:** A rectangular region of the image defined by top-left pixel coordinates $(x, y)$ and dimensions $T_w \times T_h$ [S1, book].
- **Queue:** A shared data structure containing available tile assignments. The queue supports atomic "pull" operations to retrieve a range of assignments [S1, book].

**Parameters:**
- Tile width $T_w = 32$ pixels [S1, book].
- Tile height $T_h = 4$ pixels [S1, book].
- Initial chunk size: Large (number of tiles pulled per request is high) [S1, book].
- Final chunk size: Small (number of tiles pulled per request decreases as the frame completes) [S1, book].

**Steps:**
1. **Tile Partitioning:** Divide the screen space into disjoint tiles of size $32 \times 4$ pixels [S1, book]. The total number of tiles $N_{tiles}$ is calculated as:
   $$ N_{tiles} = \left\lceil \frac{W}{32} \right\rceil \times \left\lceil \frac{H}{4} \right\rceil $$
   The tile width of 32 pixels ensures that tiles do not share a cache line with neighboring tiles, optimizing memory access patterns [S1, book].
2. **Queue Initialization:** At the beginning of each frame, create a queue containing all $N_{tiles}$ assignments [S1, book]. Each assignment represents one tile.
3. **Dynamic Assignment Loop:** For each processor $p$:
   a. Pull a range of assignments (tiles) from the shared queue [S1, book].
   b. The number of tiles pulled depends on the frame's progress:
      - Early in the frame: Pull a large chunk of tiles to reduce contention for the queue lock/atomic operation [S1, book].
      - Late in the frame: Pull smaller chunks to ensure fine-grained load balancing as remaining work diminishes [S1, book].
   c. Process all assigned tiles (ray-tracing/isosurface extraction) [S1, book].
   d. If work remains in the queue, return to step 3a [S1, book].
4. **Completion:** The frame is complete when the queue is empty and all processors have finished their current tasks [S1, book].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import threading
from collections import deque

def dynamic_tile_queue_load_balancing(screen_width, screen_height, num_processors, volume_data):
    tile_w = 32
    tile_h = 4

    # Calculate number of tiles
    tiles_x = math.ceil(screen_width / tile_w)
    tiles_y = math.ceil(screen_height / tile_h)
    total_tiles = tiles_x * tiles_y

    # Initialize queue with all tile indices
    queue = deque(range(total_tiles))
    queue_lock = threading.Lock()

    # Shared state for chunk size adaptation
    current_chunk_size = 16  # Large initial chunk [G]
    threshold = 10           # Threshold to switch to small chunks [G]

    def process_tile(tile_idx):
        tx = tile_idx % tiles_x
        ty = tile_idx // tiles_x
        x_start, x_end = tx * tile_w, min((tx + 1) * tile_w, screen_width)
        y_start, y_end = ty * tile_h, min((ty + 1) * tile_h, screen_height)
        # Raycasting and volume access logic [G]
        # Bug fix: Cast coordinates to int before indexing to prevent TypeError
        for y in range(y_start, y_end):
            for x in range(x_start, x_end):
                ix, iy = int(x), int(y)
                _ = volume_data[0][iy][ix]

    def worker():
        nonlocal current_chunk_size
        while True:
            with queue_lock:
                if not queue:
                    break
                batch = []
                for _ in range(current_chunk_size):
                    if queue:
                        batch.append(queue.popleft())
                    else:
                        break
            if not batch:
                break
            for tile_idx in batch:
                process_tile(tile_idx)
            # Adapt chunk size: reduce as frame progresses [S1, book]
            with queue_lock:
                remaining = len(queue)
            if remaining < threshold:
                current_chunk_size = 1  # Small final chunk [G]

    threads = [threading.Thread(target=worker) for _ in range(num_processors)]
    for t in threads: t.start()
    for t in threads: t.join()
```

**Edge Cases and Pitfalls:**
- **Cache Line Sharing:** Tile width must be chosen to avoid sharing cache lines with neighbors. A width of 32 pixels is specified to ensure this [S1, book].
- **Queue Contention:** Pulling single tiles individually causes high contention. Large initial chunks mitigate this [S1, book].
- **Load Imbalance:** Small remaining work requires small chunks to prevent idle processors while one processor finishes a large remaining chunk [S1, book].
- **Coordinate Indexing:** Volume texture lookups must cast floating-point ray coordinates to integers before array indexing to prevent `TypeError: can't multiply sequence by non-int of type 'float'` [G].

## Subtopics

### [Scanline Interleaving](2757-scanline-interleaving.md)
Distributes pixels in a striped, round-robin pattern to balance load based on depth complexity variations. It sets itself apart by assuming neighbor mesh-cells have equal workload, whereas this method explicitly models per-tile complexity. Choose it when depth complexity varies locally across the screen.

### [Load-Balanced Cell Assignment](4194-load-balanced-cell-assignment.md)
Assigns coarse 2D mesh-cells to processors based on global ray-segment counts to minimize communication during merging. It sets itself apart by explicitly modeling each mesh-cell's computational load rather than assuming uniform neighbor workloads. Choose it during the Local Pixel Merging phase when ray-segments from different processors must be combined.

### [Farm on Demand (FD)](7455-farm-on-demand-fd.md)
A parallel isosurface extraction algorithm based on the processor farm paradigm where balancing is obtained dynamically. It sets itself apart by relying on task granularity rather than image-space tiling. Choose it when enough partitions (tasks) can be generated to sustain dynamic balancing.

### [Hybrid Image Partitioning](7675-hybrid-image-partitioning.md)
Combines static block assignment with dynamic tile grabbing to balance load distribution and minimize overlapping voxel accesses. It sets itself apart by merging static locality with dynamic queue mechanics. Choose it when minimizing redundant voxel fetches is critical alongside load balancing.

## Operating characteristics
- **Efficiency:** On 128 processors, efficiency is 91% for View 1 (nearly 100% pixel coverage) and 80% for View 2 (approx 75% pixel coverage) [S1, book].
- **Performance:** For a 512x512 image of the Visible Human bone isosurface, 128 CPUs achieved 20.49 FPS (View 1) and 39.98 FPS (View 2) [S1, book].
- **Scalability:** Speedup is linear up to 32 processors. Beyond 64 processors, efficiency drops due to load imbalances and synchronization time required to maintain frame rates [S1, book].
- **Strengths:** High utilization across varying views; reduces contention via large initial chunks; balances load efficiently near frame completion via small chunks [S1, book].
- **Limitations:** Efficiency decreases with more processors (>64) due to synchronization overhead and load imbalance; efficiency would be higher for larger images [S1, book].

## Background
The method was developed for fast isosurface extraction from large image datasets, specifically the Visible Human dataset [S1, book]. It was introduced in the context of real-time ray-tracing for volume rendering, addressing the challenge of unpredictable isosurface extraction complexity per pixel [S1, book]. The approach is part of a broader strategy involving data bricking and spatial hierarchies to optimize cache locality and empty space skipping [S1, book].

## Sources
[S1] Fast Isosurface Extraction Methods for Large Image Data sets, https://publications.sci.utah.edu/publications/livnat08/Large-scale-isosurface08.pdf, fetched 2026-10-28 (book)
