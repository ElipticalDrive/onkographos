# Dynamic Load Balancing via Tile Queue

Also called: Parallel Volume Rendering (Screen Space Partitioning), Tile Queue Load Balancing

## Overview
Dynamic Load Balancing via Tile Queue is a parallel processing scheme that splits screen space into tiles and assigns them to processors via a shared queue to ensure high processor utilization over a wide range of views [S2, book]. It addresses the unpredictability of isosurface extraction complexity per pixel, which static partitioning cannot handle efficiently [S1, unofficial]. It sets itself apart from static partitioning by dynamically adjusting task granularity to balance load and reduce contention [S2, book].

## How it works
**Inputs and Outputs:**
*   **Input:** A volumetric dataset, a camera view, and a set of $P$ processors.
*   **Output:** A rendered image (framebuffer).

**Data Structures:**
*   **Tile:** A rectangular region of the image. In the reference implementation, tiles are 32 pixels wide by 4 pixels high [S2, book]. The width ensures tiles do not share a cache line with neighboring tiles [S2, book].
*   **Queue:** A shared data structure containing tile assignments. The queue supports atomic "pull" operations [G].

**Algorithm Steps:**
1.  **Initialization:** Split the screen space into tiles in image space [S1, unofficial]. At the beginning of a frame, each tile becomes an assignment in the shared queue [S2, book].
2.  **Work Pulling:** Each processor pulls a range of assignments from the queue [S2, book].
3.  **Processing:** The processor performs the assigned work (ray-tracing/isosurface extraction) for the tiles in the pulled range [S1, unofficial].
4.  **Iteration & Granularity Adjustment:** After completing the work, the processor returns to the queue to pull more assignments [S2, book]. Assignments are initially doled out in large chunks, getting smaller as the frame nears completion to reduce contention early and balance load efficiently late [S2, book].
5.  **Task Redistribution:** If a node cannot finish its task before a time-out stamp, it takes the remnant, re-partitions it, and re-distributes it [S1, unofficial]. Nodes may also steal smaller tasks from other nodes once they have completed their own tasks [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import time
import threading

tile_queue = []
queue_lock = threading.Lock()
TIMEOUT = 1.0  # seconds

def initialize_frame(tiles):
    # Order tasks in decreasing size for better load balancing [S1, unofficial]
    tiles.sort(key=lambda t: t.complexity, reverse=True)
    tile_queue.extend(tiles)

def pull_work(max_count):
    with queue_lock:
        chunk = tile_queue[:max_count]
        del tile_queue[:max_count]
    return chunk

def render_tile(tile):
    # Perform ray-marching/compositing for the tile
    pass

def processor_loop():
    while True:
        # Granularity: large chunks early, small chunks late [S2, book]
        chunk_size = max(1, len(tile_queue) // 2)
        tiles = pull_work(chunk_size)
        if not tiles:
            break

        start = time.time()
        for i, tile in enumerate(tiles):
            render_tile(tile)
            # Time-out: re-partition and redistribute remnant [S1, unofficial]
            if time.time() - start > TIMEOUT:
                with queue_lock:
                    tile_queue.extend(tiles[i:])
                break
```

**Parameters:**
*   **Tile Dimensions:** 32 pixels wide by 4 pixels high [S2, book].
*   **Queue Ordering:** Tasks ordered in decreasing size/complexity to achieve better load balancing [S1, unofficial].
*   **Granularity Strategy:** Starts large, decreases as the frame completes [S2, book]. Exact formula for size reduction is not given in the sources [G].

**Edge Cases and Pitfalls:**
*   **Contention:** High contention for the queue can occur if chunk sizes are too small early in the frame. The large initial chunk size mitigates this [S2, book].
*   **Load Imbalance:** If the workload is extremely uneven, processors may finish early and idle. The decreasing chunk size helps balance this late in the frame [S2, book].
*   **Synchronization Overhead:** Synchronizing processors at the required frame rate can reduce efficiency, especially with large numbers of processors (> 64) [S2, book].

## Subtopics

### [Scanline Interleaving](2757-scanline-interleaving.md)
Scanline Interleaving is an image-space partitioning strategy that distributes pixels across processors in a striped, round-robin pattern to balance the computational load of ray-merging operations. It addresses variations in depth complexity caused by local variations in cell sizes by dispersing ray-merging computations evenly across the screen.

### [Load-Balanced Cell Assignment](4194-load-balanced-cell-assignment.md)
Load-Balanced Cell Assignment (PA) distributes the ray-segment merging workload in tiled parallel volume rendering by assigning coarse 2D mesh-cells to processors. It solves the problem of balancing computational load during the Local Pixel Merging (LPM) phase, where ray-segments generated by different processors for the same pixel must be combined into an atomic task per mesh-cell. Unlike [Scanline Interleaving](2757-scanline-interleaving.md), which assumes neighbor mesh-cells have equal workload, PA explicitly models each mesh-cell's computational load based on its global ray-segment count to achieve a balanced distribution.

### [Farm on Demand (FD)](7455-farm-on-demand-fd.md)
Farm on Demand (FD) is a parallel isosurface extraction algorithm that achieves dynamic load balancing by subdividing a volumetric dataset into a large number of fine-grained tasks and distributing them to worker processes on demand. It builds on [Dynamic Load Balancing via Tile Queue](1763-dynamic-load-balancing-via-tile-queue.md) by applying the processor farm paradigm to volume data, where the master assigns small intervals of slices to workers as they become available, rather than using static partitioning. Unlike its sibling [Load Balanced Cell Assignment](4194-load-balanced-cell-assignment.md) (LBC), which uses static contiguous partitions, FD provides near-perfect computational load balancing for uneven isosurface distributions but probabilistically produces a significantly higher number of shared vertices due to non-contiguous task assignment.

### [Hybrid Image Partitioning](7675-hybrid-image-partitioning.md)
Hybrid Image Partitioning is a dynamic load balancing scheme for parallel volume rendering that combines static block assignment with dynamic tile grabbing to balance load distribution and minimize overlapping voxel accesses. It sets itself apart from static contiguous partitioning by avoiding poor load distribution caused by varying image complexity, and from static interleaved partitioning by minimizing excessive overlapping voxel accesses through spatial locality. It builds on [Dynamic Load Balancing via Tile Queue](1763-dynamic-load-balancing-via-tile-queue.md) by distributing the queue management across processors to avoid the bottleneck of a centralized point of control.

### Central Queue Load Balancing
Central Queue Load Balancing introduces a central queue of work where processors grab pending loads as soon as they become available [S3, unofficial]. Processors use a shared queue containing a front-to-back sorted list of bricks, rendering corresponding bricks to a shared frame buffer [S3, unofficial]. Race conditions are solved by having each processor render each brick to a private buffer, then moving to a complete work queue, blending from the private buffer onto the shared buffer only once all dependent bricks are in the completed work queue [S3, unofficial]. However, choosing a central dispatching mechanism serializes the program, creating a single point of control that becomes a bottleneck as the number of processors increases [S3, unofficial].

### Dynamic Mapping Problem
The Dynamic Mapping Problem is the problem of mapping image sources to display tiles through an ordered sequence of image compositing operations [S4]. It is formulated as embedding an interdependency graph (logical pipelines) into a fixed host graph (network topology) without exhausting bandwidth [S4]. The general problem is NP-complete, and solving it dynamically per frame requires assumptions allowing efficient polynomial time solution [S4].

## Operating characteristics
**Time and Memory Complexity:**
*   **Time:** The algorithm is dynamic, so time complexity depends on the workload distribution and queue contention.
*   **Memory:** Memory usage is proportional to the number of tiles in the queue and any private buffers used in variants.

**Measured Performance:**
*   **Hardware:** 128 CPUs [S2, book].
*   **Scene:** Bone isosurface in the Visible Human dataset [S2, book].
*   **Resolution:** 512x512 image [S2, book].
*   **Results:**
    *   **View 1 (nearly 100% pixel coverage):** 20.49 FPS, 91% efficiency [S2, book].
    *   **View 2 (approx 75% pixel coverage):** 39.98 FPS, 80% efficiency [S2, book].
*   **Scalability:** Efficiency drops with larger numbers of processors (> 64) due to load imbalances and synchronization time [S2, book].

**Strengths and Limitations:**
*   **Strengths:** High processor utilization over a wide range of views; adapts to unpredictable isosurface complexity; reduces contention with large initial chunks [S2, book].
*   **Limitations:** Synchronization overhead at high processor counts; requires a shared queue mechanism which can become a bottleneck if not distributed [S2, book][S3, unofficial].

## Background
Dynamic Load Balancing via Tile Queue was introduced in the context of large-scale isosurface extraction methods for large image data sets [S2, book]. It was developed to address the challenges of real-time ray-tracing on large datasets, specifically the Visible Human dataset [S2, book]. The method was part of a broader effort to optimize parallel volume rendering on CPU clusters, leveraging cache locality through bricking and spatial hierarchies [S2, book].

## Sources
[S1] Overview of Volume Rendering - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/papers/volvisOverview.pdf, fetched 2026-10-27 (unofficial)
[S2] Fast Isosurface Extraction Methods for Large Image Data sets, https://publications.sci.utah.edu/publications/livnat08/Large-scale-isosurface08.pdf, fetched 2026-10-28 (book)
[S3] Volume Visualization and Compositing on Large-Scale Displays Using Handheld Touchscreen Interaction, https://doi.org/10.25781/kaust-d5m7l, fetched 2026-10-26 (unofficial)
[S4] ``Scalable interactive volume rendering using off-the-shelf components,'', http://www.cacr.caltech.edu/~slombey/viz/pvr/pvr-paper.pdf, fetched 2026-10-28
