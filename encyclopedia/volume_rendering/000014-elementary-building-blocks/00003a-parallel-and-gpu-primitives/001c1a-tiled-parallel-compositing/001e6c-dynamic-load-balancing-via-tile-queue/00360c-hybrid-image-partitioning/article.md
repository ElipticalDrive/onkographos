# Hybrid Image Partitioning

Also called: Task Queue Image Partitioning, distributed task queue

## Overview
Hybrid Image Partitioning is a parallel volume rendering scheme that combines static block assignment with dynamic tile grabbing to balance load distribution and minimize overlapping voxel accesses [S1]. Unlike static contiguous partitioning, which suffers from poor load distribution due to varying image complexity, and static interleaved partitioning, which causes excessive overlapping voxel accesses due to small tile sizes, this method provides better load distribution while minimizing redundant computation by assigning spatially adjacent tiles to the same processor initially [S1].

## How it works
**Inputs:**
- Image plane dimensions.
- Number of processors $P$.
- Volume dataset.
- Ray tracing parameters (isovalue, transfer function, etc.).

**Outputs:**
- Rendered image.

**Data Structures:**
- **Image Plane:** Divided into $P$ contiguous rectangular blocks of comparable size.
- **Tiles:** Each block is divided into fixed-size square image tiles. The tile size is chosen to balance load granularity against the overhead of tile management and the cost of overlapping voxel accesses [S1].
- **Task Queues:** There are $P$ queues, one per processor. Each queue manages the tiles within the processor's statically assigned block. The management is distributed, avoiding a centralized bottleneck [S1].

**Steps:**
1. **Static Partitioning:** Divide the image plane into $P$ contiguous rectangular blocks of comparable size.
2. **Tile Division:** Divide each image block into fixed-size square image tiles.
3. **Initial Assignment:** Statically assign each image block to one processor. Each processor begins ray tracing the tiles in its assigned block in scan line order.
4. **Dynamic Grabbing:** When a processor finishes its assigned block, it grabs an unprocessed image tile from the next processor in scan line order that has not yet finished its block.
5. **Completion:** Processors continue to grab tiles from other processors' queues until all tiles are completed.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Initialization
num_processors = P
image_width, image_height = W, H
tile_size = T # Fixed square tile dimension

# 1. Partition image plane into P contiguous blocks
# Blocks are arranged in scan-line order across the image
blocks = []
for i in range(num_processors):
    # Calculate block boundaries to ensure comparable size
    # Assuming a simple row-major split for P processors
    # Exact geometry depends on aspect ratio, but blocks are contiguous
    block_start_x = (i * image_width) // num_processors
    block_end_x = ((i + 1) * image_width) // num_processors
    blocks.append({
        'id': i,
        'x_start': block_start_x,
        'x_end': block_end_x,
        'y_start': 0,
        'y_end': image_height,
        'tiles': []
    })

# 2. Divide each block into tiles
for block in blocks:
    x = block['x_start']
    while x < block['x_end']:
        y = block['y_start']
        while y < block['y_end']:
            tile = {
                'x': x,
                'y': y,
                'width': min(T, block['x_end'] - x),
                'height': min(T, block['y_end'] - y),
                'owner': block['id'],
                'processed': False
            }
            block['tiles'].append(tile)
            x += T
        y += T

# 3. Assign blocks to processors
# Processor i is responsible for blocks[i] initially
processor_queues = [block['tiles'] for block in blocks]

# 4. Ray Tracing Loop
# Each processor runs this logic concurrently
def process_processor(my_id):
    queue = processor_queues[my_id]
    
    # Process own tiles in scan line order
    while queue:
        tile = queue.pop(0)
        if not tile['processed']:
            ray_trace_tile(tile)
            tile['processed'] = True
            
    # When own block is done, grab from next processor in scan line order
    next_id = (my_id + 1) % num_processors
    while next_id != my_id:
        # Check if next processor has finished its block
        # In a real implementation, this requires synchronization or
        # checking if the queue is empty and the processor is idle.
        # The source implies a simple linear scan for the next available tile.
        
        # Try to grab a tile from the next processor's queue
        if processor_queues[next_id]:
            tile = processor_queues[next_id].pop(0)
            if not tile['processed']:
                ray_trace_tile(tile)
                tile['processed'] = True
                continue # Stay at this processor if it has more work
        
        # If next processor's queue is empty or we can't grab, move to next
        next_id = (next_id + 1) % num_processors
        
        # Safety break if all processors are done
        if all(len(q) == 0 for q in processor_queues):
            break

# Edge Cases and Pitfalls:
# - Adaptive Image Sampling: If adaptive sampling is used, adjacent tiles may share corner pixels.
#   This creates dependencies. If adjacent tiles are processed by different processors,
#   synchronization or replicated computation is required [S1].
#   The method minimizes this by keeping adjacent tiles in the same block initially.
#   If a tile is grabbed from another processor, the synchronization overhead is small
#   because it only affects the boundary between the two processors' regions [S1].
# - Centralized Bottleneck: This method avoids the bottleneck of centralized dynamic partitioning
#   by distributing queue management across processors (one queue per processor) [S1].
```

## Subtopics

### [Ray Queue Scheduling](9710-ray-queue-scheduling.md)
A hybrid scheme for load balancing in distributed ray tracing where ray-task scheduling is updated dynamically within the local partition of each render-node.

## Operating characteristics
**Time and Memory Complexity:**
- Time complexity is generally $O(N)$ where $N$ is the number of tiles, with load balancing improving the constant factor by reducing idle time.
- Memory complexity is proportional to the number of tiles and the volume data distribution.

**Measured Performance:**
- Provides better load distribution than static partitioning schemes [S1].
- Reduces overlapping voxel accesses compared to interleaved partitioning [S1].
- Avoids the bottleneck of centralized dynamic partitioning as processor count increases [S1].

**Strengths and Limitations:**
- **Strengths:** Good load distribution, reduced overlapping voxel accesses, scalable due to distributed queue management.
- **Limitations:** Synchronization overhead may arise in adaptive image sampling if adjacent tiles are processed by different processors, though this is minimized by the initial static assignment [S1].

**When to Use:**
- Use when image complexity varies significantly across the image plane and viewing position.
- Use when scaling to a large number of processors where centralized load balancing becomes a bottleneck.

**When Not to Use:**
- Use static partitioning if image complexity is uniform and predictable.
- Use interleaved partitioning if the overhead of tile management is prohibitive and overlapping accesses are acceptable.

## Background
Hybrid Image Partitioning was introduced in the context of volume rendering on scalable shared-memory MIMD architectures [S1]. It addresses the limitations of static contiguous and interleaved partitioning schemes, as well as the scalability issues of centralized dynamic partitioning. The method was implemented on the DASH architecture [S1].

## Sources
[S1] Volume rendering on scalable shared-memory MIMD architectures, https://doi.org/10.1145/147130.147141, fetched 2026-10-25
