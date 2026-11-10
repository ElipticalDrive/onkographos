# Dynamic Load Balancing via Tile Queue

Also called: Tile Queue Load Balancing

## Overview
Dynamic Load Balancing via Tile Queue is a parallel processing scheme that splits screen space into tiles and assigns them to processors via a shared queue to ensure high processor utilization over a wide range of views [S1, book]. It addresses the unpredictability of isosurface extraction complexity per pixel, which static partitioning cannot handle efficiently [S1, book]. It sets itself apart from static contiguous partitioning by adapting to workload variations and from centralized dispatching by distributing the queue management logic to avoid a single point of control bottleneck [S1, book][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A volumetric dataset, a camera view, and a set of $P$ processors.
*   **Output:** A rendered image (framebuffer).

**Data Structures:**
*   **Tile:** A rectangular region of the image. In the reference implementation, tiles are 32 pixels wide by 4 pixels high [S1, book]. The width (32 pixels) ensures that tiles do not share a cache line with neighboring tiles (assuming standard cache line sizes, e.g., 128 bytes for 4 pixels of float data) [S1, book].
*   **Queue:** A shared data structure containing tile assignments. The queue supports atomic "pull" operations.

**Algorithm Steps:**
1.  **Initialization:** Split the screen space into tiles in image space [S1, book]. At the beginning of a frame, each tile becomes an assignment in the shared queue [S1, book].
2.  **Work Pulling:** Each processor pulls a range of assignments from the queue [S1, book].
3.  **Processing:** The processor performs the assigned work (ray-tracing/isosurface extraction) for the tiles in the pulled range [S1, book].
4.  **Iteration:** After completing the work, the processor returns to the queue to pull more assignments [S1, book].
5.  **Granularity Adjustment:** Assignments are initially doled out in large chunks. As the frame nears completion, the chunk size gets smaller [S1, book]. This large granularity early reduces contention, while smaller granularity late balances load efficiently [S1, book].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Global shared state
tile_queue = Queue() # Contains all tiles for the current frame

def initialize_frame(tiles):
    """Populate the queue with all tiles at the start of a frame."""
    for tile in tiles:
        tile_queue.enqueue(tile)

def processor_work_loop(processor_id):
    """Main loop for a single processor."""
    while True:
        # Pull a chunk of tiles from the queue
        # The size of the chunk depends on the frame progress
        chunk_size = get_dynamic_chunk_size()
        assigned_tiles = []
        
        for _ in range(chunk_size):
            tile = tile_queue.try_dequeue()
            if tile is None:
                break
            assigned_tiles.append(tile)
            
        if not assigned_tiles:
            # No more work available
            break
            
        # Perform rendering for the assigned tiles
        for tile in assigned_tiles:
            render_tile(tile)

def get_dynamic_chunk_size():
    """
    Returns the number of tiles to pull in one atomic operation.
    Large at start, small at end.
    Implementation detail: [G]
    """
    # Not given in the sources. 
    # Typical implementation: start with N tiles, decrement N as 
    # queue empties or based on time elapsed.
    pass
```

**Parameters:**
*   **Tile Dimensions:** 32 pixels wide by 4 pixels high [S1, book].
*   **Chunk Size Strategy:** Starts large, decreases as the frame completes [S1, book]. Exact formula for size reduction is not given in the sources [G].

**Edge Cases and Pitfalls:**
*   **Contention:** High contention for the queue can occur if chunk sizes are too small early in the frame. The large initial chunk size mitigates this [S1, book].
*   **Load Imbalance:** If the workload is extremely uneven, small processors may finish early and idle. The decreasing chunk size helps balance this late in the frame [S1, book].
*   **Synchronization Overhead:** Synchronizing processors at the required frame rate can reduce efficiency, especially with large numbers of processors (> 64) [S1, book].

## Subtopics

### [Scanline Interleaving](2757-scanline-interleaving.md)
Scanline Interleaving is an image-space partitioning strategy that distributes pixels across processors in a striped, round-robin pattern to balance the computational load of ray-merging operations. It addresses variations in depth complexity caused by local variations in cell sizes by dispersing ray-merging computations evenly across the screen. Unlike Dynamic Load Balancing via Tile Queue, which uses a queue of tiles, Scanline Interleaving uses a static pattern.

### [Load-Balanced Cell Assignment](4194-load-balanced-cell-assignment.md)
Load-Balanced Cell Assignment (PA) distributes the ray-segment merging workload in tiled parallel volume rendering by assigning coarse 2D mesh-cells to processors. It solves the problem of balancing computational load during the Local Pixel Merging (LPM) phase, where ray-segments generated by different processors for the same pixel must be combined into an atomic task per mesh-cell. Unlike Dynamic Load Balancing via Tile Queue, which dynamically assigns tiles, PA explicitly models each mesh-cell's computational load based on its global ray-segment count to achieve a balanced distribution.

### [Farm on Demand (FD)](7455-farm-on-demand-fd.md)
Farm on Demand (FD) is a parallel isosurface extraction algorithm that achieves dynamic load balancing by subdividing a volumetric dataset into a large number of fine-grained tasks and distributing them to worker processes on demand. It builds on Dynamic Load Balancing via Tile Queue by applying the processor farm paradigm to volume data, where the master assigns small intervals of slices to workers as they become available, rather than using static partitioning. Unlike its sibling Load Balanced Cell Assignment (LBC), which uses static contiguous partitions, FD provides near-perfect computational load balancing for uneven isosurface distributions but probabilistically produces a significantly higher number of shared vertices due to non-contiguous task assignment.

### [Hybrid Image Partitioning](7675-hybrid-image-partitioning.md)
Hybrid Image Partitioning is a dynamic load balancing scheme for parallel volume rendering that combines static block assignment with dynamic tile grabbing to balance load distribution and minimize overlapping voxel accesses. It sets itself apart from static contiguous partitioning by avoiding poor load distribution caused by varying image complexity, and from static interleaved partitioning by minimizing excessive overlapping voxel accesses through spatial locality. It builds on Dynamic Load Balancing via Tile Queue by distributing the queue management across processors to avoid the bottleneck of a centralized point of control.

### Central Queue Load Balancing
Central Queue Load Balancing introduces a central queue of work where processors grab pending loads as soon as they become available [S2, unofficial]. Processors use a shared queue containing a front-to-back sorted list of bricks, rendering corresponding bricks to a shared frame buffer [S2, unofficial]. Race conditions are solved by having each processor render each brick to a private buffer, then moving to a complete work queue, blending from the private buffer onto the shared buffer only once all dependent bricks are in the completed work queue [S2, unofficial]. However, choosing a central dispatching mechanism serializes the program, creating a single point of control that becomes a bottleneck as the number of processors increases [S2, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
*   **Time:** The algorithm is dynamic, so time complexity depends on the workload distribution. The overhead of queue operations is $O(1)$ per tile pull, but contention can increase this.
*   **Memory:** Memory usage is proportional to the number of tiles in the queue and the private buffers used in variants like Central Queue Load Balancing.

**Measured Performance:**
*   **Hardware:** 128 CPUs [S1, book].
*   **Scene:** Bone isosurface in the Visible Human dataset [S1, book].
*   **Resolution:** 512x512 image [S1, book].
*   **Results:**
    *   **View 1 (nearly 100% pixel coverage):** 20.49 FPS, 91% efficiency [S1, book].
    *   **View 2 (approx 75% pixel coverage):** 39.98 FPS, 80% efficiency [S1, book].
*   **Scalability:** Efficiency drops with larger numbers of processors (> 64) due to load imbalances and synchronization time [S1, book].

**Strengths and Limitations:**
*   **Strengths:** High processor utilization over a wide range of views; adapts to unpredictable isosurface complexity; reduces contention with large initial chunks [S1, book].
*   **Limitations:** Synchronization overhead at high processor counts; requires a shared queue mechanism which can become a bottleneck if not distributed [S1, book][S2, unofficial].

**When to Use:**
*   Use when the workload per pixel is unpredictable (e.g., complex isosurfaces).
*   Use when static partitioning leads to poor load balance.

**When Not to Use:**
*   Use when the workload is uniform and predictable; static partitioning is simpler and has less overhead.
*   Use when the number of processors is very large and synchronization overhead becomes prohibitive.

## Background
Dynamic Load Balancing via Tile Queue was introduced in the context of large-scale isosurface extraction methods for large image data sets [S1, book]. It was developed to address the challenges of real-time ray-tracing on large datasets, specifically the Visible Human dataset [S1, book]. The method was part of a broader effort to optimize parallel volume rendering on CPU clusters, leveraging cache locality through bricking and spatial hierarchies [S1, book].

## Sources
[S1] Fast Isosurface Extraction Methods for Large Image Data sets, https://publications.sci.utah.edu/publications/livnat08/Large-scale-isosurface08.pdf, fetched 2026-10-28 (book)
[S2] Volume Visualization and Compositing on Large-Scale Displays Using Handheld Touchscreen Interaction, https://doi.org/10.25781/kaust-d5m7l, fetched 2026-10-26 (unofficial)
