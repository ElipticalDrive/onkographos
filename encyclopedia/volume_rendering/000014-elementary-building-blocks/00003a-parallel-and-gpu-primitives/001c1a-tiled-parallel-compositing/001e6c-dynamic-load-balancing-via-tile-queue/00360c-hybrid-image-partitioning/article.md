# Hybrid Image Partitioning

Also called: Task Queue Image Partitioning, distributed task queue

## Overview
Hybrid Image Partitioning is a dynamic load balancing scheme for parallel volume rendering that combines static block assignment with dynamic tile grabbing to balance load distribution and minimize overlapping voxel accesses [S1]. It sets itself apart from static contiguous partitioning by avoiding poor load distribution caused by varying image complexity, and from static interleaved partitioning by minimizing excessive overlapping voxel accesses through spatial locality [S1]. It builds on [Dynamic Load Balancing via Tile Queue](1763-dynamic-load-balancing-via-tile-queue.md) by distributing the queue management across processors to avoid the bottleneck of a centralized point of control [S1].

## How it works
**Inputs:**
- Image plane dimensions.
- Number of processors $P$.
- Volume dataset.
- Ray tracing parameters (e.g., early termination threshold, adaptive sampling region size $w_{max}$) [S1].

**Outputs:**
- Rendered image.

**Data Structures:**
- **Image Plane:** Divided into $P$ contiguous rectangular blocks of comparable size.
- **Image Tiles:** Each block is divided into fixed-size square image tiles.
- **Task Queues:** One queue per processor. Each queue contains the image tiles assigned to that processor. The queues are distributed; processor $i$ manages its own queue and can access the queue of processor $(i+1) \pmod P$ [S1].

**Steps:**
1. **Static Partitioning:** Divide the image plane into $P$ contiguous rectangular blocks of comparable size.
2. **Tiling:** Divide each image block into fixed-size square image tiles.
3. **Assignment:** Statically assign each image block to one processor. Each processor initializes its local task queue with the tiles in its assigned block.
4. **Initial Processing:** Each processor begins ray tracing the tiles in its assigned block in scan line order.
5. **Dynamic Grabbing:** When a processor finishes all tiles in its assigned block, it grabs an unprocessed image tile from the next processor in scan line order (processor $(i+1) \pmod P$) that has not yet finished its block.
6. **Continuation:** Processors continue to grab tiles from other processors' queues until all tiles are completed.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hybrid_image_partitioning(image_plane, volume, P):
    # Step 1: Static Partitioning
    # Divide image plane into P contiguous rectangular blocks
    blocks = partition_image_plane(image_plane, P)
    
    # Step 2: Tiling
    # Divide each block into fixed-size square image tiles
    # Tile size is a parameter, e.g., 16x16 pixels
    tile_size = 16
    queues = []
    for i in range(P):
        tiles = divide_into_tiles(blocks[i], tile_size)
        queues.append(tiles)
        
    # Step 3 & 4: Assignment and Initial Processing
    # Each processor i manages queue i
    def processor_task(i):
        while True:
            # Try to grab a tile from own queue first
            tile = queues[i].pop_front()
            
            # If own queue is empty, try next processor's queue
            if tile is None:
                next_proc = (i + 1) % P
                # Check if next processor is done
                if not is_processor_done(next_proc):
                    tile = queues[next_proc].pop_front()
            
            # If no tile available, check if all work is done
            if tile is None:
                if all_queues_empty():
                    break
                else:
                    continue # Wait or yield
            
            # Ray trace the tile
            ray_trace_tile(tile, volume)
            
            # Mark processor as done if own queue was the last source
            # Note: Logic for 'is_processor_done' depends on tracking
            # which processor owns which tiles. 
            # Simplified: A processor is 'done' if its own queue is empty
            # AND it has not grabbed any tiles from others? 
            # The source says "grabs... from next processor... that has not yet finished its block".
            # This implies a state of 'finished block' per processor.
            
    # Launch P processors
    for i in range(P):
        spawn_thread(processor_task, i)

def is_processor_done(proc_id):
    # A processor is done if its own block is fully processed
    # This requires tracking progress. 
    # In the hybrid scheme, a processor is considered 'done' with its block
    # when its local queue is empty AND it has not taken work from others?
    # Actually, the source says: "grabs... from next processor... that has not yet finished its block".
    # This implies we need to know if the next processor has finished its *own* block.
    # Let's assume a flag or counter.
    return processor_block_finished[proc_id]

def all_queues_empty():
    return all(len(q) == 0 for q in queues)
```

**Parameters:**
- **Tile Size:** Fixed-size square tiles. The size must be large enough to minimize overlapping voxel accesses but small enough to allow dynamic load balancing [S1]. Not given in the sources.
- **$w_{max}$:** Size of adaptive image sampling regions in pixels. Rays are cast only from the four corner pixels of each region [S1]. Not given in the sources.

**Edge Cases and Pitfalls:**
- **Centralized Bottleneck:** Centralized dynamic partitioning becomes a bottleneck as processor count increases. This method avoids that by distributing queue management [S1].
- **Pixel Sharing in Adaptive Sampling:** Adaptive image sampling creates dependencies between adjacent tiles because they share corner pixel values. If adjacent tiles are processed by different processors, synchronization or replicated computation is required. The hybrid scheme minimizes this by keeping spatially adjacent tiles with the same processor initially [S1].
- **Boundary Conditions:** When a processor grabs from the next processor, it must ensure the next processor has not finished its block. If the next processor has finished, it may have already grabbed from the one after it. The source specifies grabbing from the "next processor in scan line order that has not yet finished its block" [S1].

## Subtopics

### [Ray Queue Scheduling](9710-ray-queue-scheduling.md)
Ray Queue Scheduling is a dynamic load balancing method for distributed volume rendering that manages ray processing at the granularity of spatial cells (flex-blocks) rather than static image tiles or bricks. It solves the problem of uneven computational load in distributed ray tracing by maintaining a queue of pending rays for each cell and processing cells one at a time based on availability. It sets itself apart from static partitioning by dynamically assigning work to idle processors and from pure tile-based approaches by focusing on spatial coherence and cell-level traversal, building on [Hybrid Image Partitioning](7675-hybrid-image-partitioning.md).

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Better scalability than centralized dynamic schemes because queue management is distributed (one queue per processor) [S1].
- **Strengths:** Good load distribution compared to static schemes; reduced overlapping voxel accesses compared to interleaved schemes; avoids centralized bottleneck [S1].
- **Limitations:** Synchronization overhead for adaptive image sampling if adjacent tiles are processed by different processors (though minimized) [S1].
- **When to use:** When image complexity varies significantly across the image plane and viewing position, and when overlapping voxel accesses are a concern [S1].
- **When not:** When a centralized queue is feasible and scalable, or when adaptive sampling dependencies are too costly to manage [S1].

## Background
Originated in 1992 by researchers at the University of North Carolina at Chapel Hill, presented at the 1992 Workshop on Volume Visualization [S1]. The method was implemented on the DASH architecture [S1]. It addresses the limitations of static contiguous partitioning (poor load distribution) and static interleaved partitioning (excessive overlapping voxel accesses) [S1].

## Sources
[S1] Volume rendering on scalable shared-memory MIMD architectures, https://doi.org/10.1145/147130.147141, fetched 2026-10-25
