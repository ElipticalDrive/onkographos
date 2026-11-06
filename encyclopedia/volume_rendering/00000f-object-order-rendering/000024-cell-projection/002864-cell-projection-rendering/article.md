# Cell-Projection Rendering

Also called: Cell-Projected Volume Rendering, Parallel Cell-Projection Volume Rendering

## Overview
Cell-projection rendering is a parallel volume rendering technique for unstructured data that projects mesh cells into image space, generating ray segments that are merged in image space [S1, unofficial][S2]. Unlike ray-casting, it does not require pre-sorting cells in depth order, offering greater flexibility in data distribution and higher accuracy than slice- or vertex-projected methods [S1, unofficial][S2]. It builds on [Cell projection](26-cell-projection.md) by utilizing hardware-accelerated scan-conversion of geometric primitives [S1, unofficial][S2].

## How it works
**Inputs:** Unstructured volumetric data (cells and nodes), camera parameters (Model-View-Projection matrix), image dimensions, and parallel configuration (number of processors).
**Outputs:** Final composited image.

**Data Structures:**
*   **Cell Data:** 16 bytes total. Contains four node indices [S1, unofficial].
*   **Node Data:** 16 bytes total. Contains three coordinates (x, y, z) and one scalar value [S1, unofficial].
*   **Ray Segment:** 16 bytes total. Contains:
    *   Sample value (color/opacity)
    *   Pixel coordinates (x, y)
    *   Starting depth value ($z_{start}$)
    *   Ending depth value ($z_{end}$) [S1, unofficial][S2]
*   **Pixel List:** A linked list at each pixel location storing ray segments [S1, unofficial].

**Algorithm Steps:**

1.  **Data Distribution:** Distribute $n$ cells to $P$ processors using round-robin assignment [S1, unofficial]. Each cell requires 16 bytes; each node requires 16 bytes. In the worst case (no node sharing), transferring $n$ cells requires $80n$ bytes [S1, unofficial].
2.  **Image Partitioning:** Assign successive scanlines to processors in round-robin fashion (scanline interleaving) to disperse ray-merging operations [S1, unofficial].
3.  **Space Partitioning Tree Construction:** Cooperatively construct a $k$-d tree on all processors to restore spatial coherence. The tree must be identical on every processor [S1, unofficial].
    *   Each processor examines local cells and establishes a cutting position to split sub-regions into equal cell counts. Cut direction alternates at each tree level [S1, unofficial].
    *   Local positions are sent to a host node, which averages them to a global position and broadcasts it back, along with the next subregion choice [S1, unofficial].
    *   Cells intersecting a cut are assigned to the region containing their centroid [S1, unofficial].
    *   Repeat until desired regions are generated [S1, unofficial].
4.  **Rendering Loop:** Processors follow the same path through the spatial partitioning tree. At each leaf node, process all cells [S1, unofficial][S2].
5.  **Scan Conversion:** For each cell, perform scan conversion to generate ray segments corresponding to the cell's projection in image space [S1, unofficial].
6.  **Routing:** Route the generated ray segments to their final destinations in the global image space (the processor owning the corresponding image scanline) [S1, unofficial].
7.  **Ray Segment Merging:** At each pixel, maintain a linked list of ray segments. Merge segments using the standard Porter-Duff over operator [S1, unofficial][S2]. Segments must be sorted by depth ($z_{start}$) before merging because they arrive in unpredictable order [S1, unofficial][S2].
8.  **Task Management:** Interleave scan conversion and merging using a polling strategy [S1, unofficial].
9.  **Termination:** Use a designated node to coordinate termination. Collect local termination messages and broadcast a global termination signal. Perform a final global synchronization [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Global constants
CELL_SIZE_BYTES = 16
NODE_SIZE_BYTES = 16
SEGMENT_SIZE_BYTES = 16
NUM_PROCESSORS = P

# Data Structures
class Cell:
    node_indices: list[int] # 4 indices

class Node:
    coords: tuple[float, float, float]
    scalar: float

class RaySegment:
    sample_value: float
    pixel_x: int
    pixel_y: int
    z_start: float
    z_end: float

class PixelList:
    segments: list[RaySegment]

# 1. Data Distribution
# Round-robin assignment of cells to processors
# Worst-case transfer: 80n bytes for n cells
cells = distribute_round_robin(all_cells, NUM_PROCESSORS)
nodes = distribute_round_robin(all_nodes, NUM_PROCESSORS)

# 2. Image Partitioning
# Round-robin scanline assignment
scanline_owner = [i % NUM_PROCESSORS for i in range(image_height)]

# 3. Space Partitioning Tree Construction
# Cooperative k-d tree construction
# Each processor determines local cut positions
# Host averages and broadcasts global cut
tree = build_kd_tree_cooperative(cells, nodes, NUM_PROCESSORS)

# 4. Rendering Loop
# All processors follow the same path through the tree
def render_leaf_node(leaf_cells):
    for cell in leaf_cells:
        # 5. Scan Conversion
        segments = scan_convert_cell(cell, mvp_matrix)
        
        # 6. Routing
        for seg in segments:
            owner = scanline_owner[seg.pixel_y]
            send_async(seg, owner)

# 7. Ray Segment Merging
def merge_segments(pixel_list):
    # Sort segments by z_start
    pixel_list.segments.sort(key=lambda s: s.z_start)
    
    # Merge using Porter-Duff over operator
    result = empty_pixel
    for seg in pixel_list.segments:
        result = porter_duff_over(result, seg)
    return result

# 8. Task Management
polling_interval = 10 # Example parameter
buffer_depth = 100    # Example parameter
buffers = {dest: [] for dest in range(NUM_PROCESSORS)}

def task_loop():
    cells_processed = 0
    while not termination_signal:
        # Scan convert cells
        if cells_processed < polling_interval:
            process_next_cell()
            cells_processed += 1
        else:
            cells_processed = 0
            
        # Check for incoming segments
        incoming = check_incoming_segments()
        if incoming:
            # Switch to merging task
            for seg in incoming:
                dest = seg.pixel_y % NUM_PROCESSORS
                buffers[dest].append(seg)
                
                # Buffer management to prevent deadlock
                if len(buffers[dest]) >= buffer_depth:
                    send_async(buffers[dest], dest)
                    buffers[dest] = []
            
            # Merge available segments
            for dest in range(NUM_PROCESSORS):
                if buffers[dest]:
                    # Merge locally if owned, or route if not
                    # In this implementation, merging happens on the owner
                    pass
            
            # If buffers are full and send pending, switch to merging to clear congestion
            if all(len(b) >= buffer_depth for b in buffers.values()):
                merge_all_pixels()

# 9. Termination
# Designated node collects messages and broadcasts global signal
```

**Parameters:**
*   **Polling Interval:** Number of cells processed before checking for incoming ray segments. High frequency prevents buffer clogging but adds overhead; low frequency causes poor asynchronous performance [S1, unofficial].
*   **Buffer Depth:** Number of ray segments accumulated before posting an asynchronous send. Small buffers cause excessive overhead; large buffers cause delays for waiting processors and reduce bandwidth spreading effectiveness [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Deadlock:** If both buffers for a destination fill up before the first send completes, the processor must switch to the ray merging task to process incoming segments and clear congestion [S1, unofficial].
*   **Unbalanced Tree:** The implementation does not guarantee a well-balanced $k$-d tree, but the searching overhead is insignificant compared to rendering time [S1, unofficial].
*   **Precision:** Ray segments contain starting and ending depth values for sorting. Precision issues in depth comparison may affect correct blending order [S1, unofficial][S2].

## Operating characteristics
*   **Time Complexity:** Rendering time scales with the number of cells and image resolution. On 128 IBM SP2 nodes, rendering 0.5 million tetrahedral cells into a 400x400 pixel image takes 0.5 seconds (2 frames per second), excluding display time [S1, unofficial].
*   **Memory Complexity:** Memory usage is proportional to the number of cells and the number of ray segments. Expected ray segment count is $10^6$ to $10^8$ per image [S1, unofficial][S2].
*   **Communication Volume:** Aggregate communication is $10^7$ to $10^9$ bytes per frame (at 16 bytes per segment) [S1, unofficial][S2].
*   **Strengths:** Flexibility in data distribution; no need for pre-sorting cells in depth order; more accurate than slice- or vertex-projected methods [S1, unofficial][S2].
*   **Limitations:** High communication volume; requires careful management of asynchronous communication to avoid deadlock; dependent on efficient scan-conversion hardware [S1, unofficial][S2].
*   **When to use:** Parallel volume rendering of unstructured data on distributed-memory systems where data distribution flexibility is prioritized [S1, unofficial].
*   **When not to use:** When memory bandwidth is severely constrained and communication overhead cannot be amortized; when real-time performance is critical and communication latency is high [S1, unofficial].

## Background
Cell-projection rendering was introduced as a scalable parallel volume rendering algorithm for three-dimensional unstructured data [S1, unofficial][S2]. It was developed to address the limitations of ray-casting, which requires depth-sorted cells, and slice- or vertex-projected methods, which offer less flexibility and accuracy [S1, unofficial][S2]. The method leverages parallel processing and asynchronous communication to achieve scalability on distributed-memory systems [S1, unofficial].

## Sources
[S1] DTIC ADA329328: A Scalable Parallel Cell-Projection Volume Rendering Algorithm for Three-Dimensional Unstructured Data, https://archive.org/details/DTIC_ADA329328, fetched 2026-10-26 (unofficial)
[S2] PDF A Scalable Parallel Cell-Projection Volume Rendering Algorithm for ..., https://web.cs.ucdavis.edu/~ma/papers/PRS97.pdf, fetched 2026-10-27
