# Cell-Projection Rendering
Also called: Cell-Projected Volume Rendering, Parallel Cell-Projection Volume Rendering

## Overview
Cell-Projection Rendering is a parallel volume rendering technique for unstructured data that projects mesh cells into image space, generating ray segments that are merged in image space [S1, unofficial][S2]. Unlike ray-casting, it does not require pre-sorting cells in depth order, offering greater flexibility in data distribution and higher accuracy than slice- or vertex-projected methods [S1, unofficial][S2]. It is specifically designed for distributed-memory systems where computational load and communication overhead must be balanced through asynchronous task management [S1, unofficial].

## How it works
The method operates on a distributed-memory architecture where processors collaboratively partition the volume and independently render local cells, routing intermediate results to a global image buffer for final compositing.

### Inputs and Outputs
*   **Input:** Unstructured volume data (mesh cells and nodes).
*   **Output:** Final composited image.

### Data Structures
The implementation relies on specific memory layouts to minimize transfer overhead and support asynchronous communication [S1, unofficial][S2].

**Cell Data Layout (16 bytes total):**
*   Four node indices (4 bytes each).
*   No connectivity data is stored; connectivity is implicit via node indices [S1, unofficial].

**Node Data Layout (16 bytes total):**
*   Three coordinates (x, y, z).
*   One scalar value (e.g., density or temperature) [S1, unofficial].

**Ray Segment Data Layout (16 bytes total):**
*   Sample value (opacity/intensity contribution).
*   Pixel coordinates (x, y).
*   Starting depth value ($z_{start}$).
*   Ending depth value ($z_{end}$).
*   These depth values are critical for sorting and merging segments that arrive in unpredictable order [S1, unofficial][S2].

### Algorithm Steps

1.  **Data Distribution:** Distribute $n$ cells to processors using round-robin assignment. Each cell requires 16 bytes for indices; each node requires 16 bytes for coordinates and scalar. In the worst case (no node sharing), transferring $n$ cells requires $80n$ bytes [S1, unofficial].
2.  **Image Partitioning:** Assign successive scanlines to processors in round-robin fashion (scanline interleaving) to disperse ray-merging operations [S1, unofficial].
3.  **Space Partitioning Tree Construction:** Cooperatively construct a $k$-d tree on all processors to restore spatial coherence. The tree must be identical on every processor [S1, unofficial].
    *   **Parallel Partitioning Algorithm:**
        1.  Each processor examines local cells and establishes a cutting position to split sub-regions into equal cell counts. Cut direction alternates at each tree level [S1, unofficial].
        2.  Local positions are sent to a host node, which averages them to a global position and broadcasts it back, along with the next subregion choice [S1, unofficial].
        3.  Cells intersecting a cut are assigned to the region containing their centroid [S1, unofficial].
        4.  Repeat until desired regions are generated [S1, unofficial].
4.  **Rendering Loop:** Processors follow the same path through the spatial partitioning tree. At each leaf node, process all cells [S1, unofficial][S2].
5.  **Scan Conversion:** For each cell, perform scan conversion to generate ray segments corresponding to the cell's projection in image space. This step is performed independently for each cell [S1, unofficial][S2].
6.  **Routing:** Route the generated ray segments to their final destinations in the global image space (the processor owning the corresponding image scanline) [S1, unofficial].
7.  **Merging:** At each pixel, maintain a linked list of ray segments. Merge segments using the standard Porter-Duff over operator. Segments must be sorted by depth ($z_{start}$ to $z_{end}$) before merging [S1, unofficial][S2].
8.  **Termination:** Use a designated node to coordinate termination. Collect local termination messages and broadcast a global termination signal. Perform a final global synchronization [S1, unofficial].

### Task Management and Asynchronous Communication
To overlap computation and communication, the algorithm uses a polling strategy with double-buffering [S1, unofficial].

*   **Polling Strategy:** Interleave scan conversion and merging.
    1.  Scan convert one or more cells.
    2.  Check for incoming ray segments.
    3.  If available, switch to merging task (sort and merge) until no input is pending.
    4.  Repeat [S1, unofficial].
*   **Asynchronous Communication:**
    *   Buffer ray segments locally.
    *   Use asynchronous send/receive to overlap computation and communication.
    *   Provide two or more buffers per destination.
    *   If a send is pending for a full buffer, place new segments in the companion buffer.
    *   If both buffers fill, switch to merging to prevent deadlock [S1, unofficial].

### Parameters
*   **Polling Interval:** The number of cells to be processed before checking for incoming ray segments. High frequency prevents buffer clogging but adds overhead; low frequency causes poor asynchronous performance [S1, unofficial].
*   **Buffer Depth:** The number of ray segments accumulated before posting an asynchronous send. Small buffers cause excessive overhead; large buffers cause delays for waiting processors and reduce bandwidth spreading effectiveness [S1, unofficial].

### Edge Cases and Pitfalls
*   **Deadlock:** If both buffers for a destination fill up before the first send completes, the processor must switch to the ray merging task to process incoming segments and clear congestion [S1, unofficial].
*   **Unbalanced Tree:** The current implementation does not guarantee a well-balanced $k$-d tree, but the searching overhead is insignificant compared to rendering time [S1, unofficial].
*   **Precision:** Ray segments contain starting and ending depth values for sorting. Since segments arrive in unpredictable order, local sorting within the pixel's linked list is required before compositing [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Global variables assumed:
# - cells: list of local cell indices
# - nodes: dictionary mapping node_id -> (x, y, z, scalar)
# - k_d_tree: global spatial partitioning tree (identical on all processors)
# - image_scanlines: list of processors owning specific scanlines
# - pixel_lists: dictionary mapping (pixel_x, pixel_y) -> list of RaySegment
# - buffers: dictionary mapping destination_processor -> [Buffer1, Buffer2, ...]

class RaySegment:
    def __init__(self, sample_value, pixel_x, pixel_y, z_start, z_end):
        self.sample_value = sample_value
        self.pixel_x = pixel_x
        self.pixel_y = pixel_y
        self.z_start = z_start
        self.z_end = z_end

def main():
    # 1. Data Distribution and Tree Construction (Assumed completed before rendering loop)
    # Data is distributed round-robin. k-d tree is constructed cooperatively.
    
    # 2. Rendering Loop
    for leaf_node in k_d_tree.traverse():
        local_cells = get_cells_in_leaf(leaf_node)
        
        for cell in local_cells:
            # 3. Scan Conversion
            segments = scan_convert_cell(cell)
            
            # 4. Routing and Buffering
            for seg in segments:
                dest_proc = get_scanline_owner(seg.pixel_y)
                buffer = get_buffer(dest_proc)
                
                if buffer.is_full():
                    # If buffer full, try companion buffer or switch to merging
                    if has_companion_buffer(dest_proc):
                        send_to_companion(dest_proc, seg)
                    else:
                        # Deadlock prevention: switch to merging
                        merge_pending_segments()
                else:
                    buffer.push(seg)
            
            # 5. Polling for Incoming Segments
            if should_poll():
                merge_pending_segments()

    # 6. Termination
    send_termination_message()
    wait_for_global_termination()
    final_sync()

def scan_convert_cell(cell):
    # Project cell vertices to 2D screen space
    # Generate ray segments for the projected polygon
    # Each segment contains: sample_value, pixel_x, pixel_y, z_start, z_end
    # Note: Exact scan conversion logic depends on cell type (tetrahedron, etc.)
    # and follows the shared principles of the Cell Projection family.
    segments = []
    # ... implementation details for projecting cell and generating segments ...
    return segments

def merge_pending_segments():
    # Sort incoming segments by depth
    # Merge using Porter-Duff over operator
    for dest_proc, buffers in buffers.items():
        for buffer in buffers:
            if buffer.has_data():
                segments = buffer.pop_all()
                # Sort segments by z_start
                segments.sort(key=lambda s: s.z_start)
                
                for seg in segments:
                    key = (seg.pixel_x, seg.pixel_y)
                    if key not in pixel_lists:
                        pixel_lists[key] = []
                    
                    # Insert into linked list maintaining depth order
                    insert_sorted(pixel_lists[key], seg)
    
    # Perform final compositing for pixels that have completed segments
    for key, seg_list in pixel_lists.items():
        if is_pixel_complete(key):
            result = composite(seg_list)
            set_pixel_color(key, result)

def get_scanline_owner(pixel_y):
    # Round-robin assignment of scanlines to processors
    return pixel_y % num_processors

def get_buffer(dest_proc):
    # Return the current active buffer for the destination
    # ...
    pass

def should_poll():
    # Check if polling interval has been reached
    # ...
    pass

def is_pixel_complete(pixel_key):
    # Determine if all ray segments for this pixel have been received
    # This is complex in parallel; often based on termination signals or depth coverage
    # ...
    pass

def composite(seg_list):
    # Apply Porter-Duff over operator
    # alpha = 1 - exp(-tau_avg * l)
    # C_out = C_in * (1 - alpha) + C_seg * alpha
    # ...
    pass
```

## Operating characteristics
*   **Time Complexity:** Rendering 0.5 million tetrahedral cells into a 400x400 pixel image takes 0.5 seconds (2 frames per second) on 128 IBM SP2 nodes (excluding display time) [S1, unofficial].
*   **Memory Complexity:** Each processor stores local cells and nodes. Worst-case transfer for $n$ cells is $80n$ bytes [S1, unofficial].
*   **Communication Volume:** Expected ray segment count is $10^6$ to $10^8$ per image. At 16 bytes per segment, aggregate communication is $10^7$ to $10^9$ bytes per frame [S1, unofficial][S2].
*   **Strengths:** Flexibility in data distribution; no need for pre-sorting cells in depth order; higher accuracy than slice- or vertex-projected methods [S1, unofficial][S2].
*   **Limitations:** High communication overhead due to large number of ray segments; requires careful tuning of polling interval and buffer depth to avoid deadlock or poor performance [S1, unofficial].
*   **When to use:** Distributed-memory systems rendering unstructured data where data distribution flexibility is prioritized over raw speed [S1, unofficial].
*   **When not to use:** Single-processor systems or scenarios where pre-sorting is feasible and communication bandwidth is severely limited [S1, unofficial].

## Background
Cell-Projection Rendering was developed for parallel volume rendering of unstructured data on distributed-memory systems. It was introduced in the late 1990s as part of research into scalable parallel rendering algorithms [S1, unofficial][S2]. The method is based on the principle that projection methods offer more flexibility in data distribution and accuracy compared to ray-casting [S1, unofficial][S2]. It was tested on the IBM SP2 supercomputer [S1, unofficial].

## Sources
[S1] DTIC ADA329328: A Scalable Parallel Cell-Projection Volume Rendering Algorithm for Three-Dimensional Unstructured Data, https://archive.org/details/DTIC_ADA329328, fetched 2026-10-26 (unofficial)
[S2] PDF A Scalable Parallel Cell-Projection Volume Rendering Algorithm for ..., https://web.cs.ucdavis.edu/~ma/papers/PRS97.pdf, fetched 2026-10-27
