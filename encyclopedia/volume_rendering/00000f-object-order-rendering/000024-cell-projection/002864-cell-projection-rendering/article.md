# Cell-Projection Rendering
Also called: Cell-Projected Volume Rendering, Parallel Cell-Projection Volume Rendering, Cell Integrating Method

## Overview
Cell-projection rendering is an object-space volume rendering technique that projects 3D data cells onto the 2D image plane to generate ray segments, which are then merged in image space to compose the final image [S1, S2, unofficial][S4]. Unlike ray-casting, it eliminates the need for pre-sorting cells in depth order, offering greater flexibility in data distribution and higher accuracy for unstructured grids [S1, unofficial][S4]. It sets itself apart from slice- or vertex-projected methods by processing entire cells independently, avoiding the aliasing problems associated with point sampling [S2, unofficial][S4].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of unstructured mesh cells (e.g., tetrahedra), each defined by node indices and associated scalar values.
*   **Output:** A 2D image composed of pixel colors and opacities.

**Data Structures:**
*   **Cell Data:** 16 bytes per cell. Contains four node indices [S1, unofficial].
*   **Node Data:** 16 bytes per node. Contains three spatial coordinates ($x, y, z$) and one scalar value ($s$) [S1, unofficial].
*   **Ray Segment:** 16 bytes per segment. Contains:
    *   Sample value (scalar/color contribution) [S1, unofficial][S4].
    *   Pixel coordinates ($u, v$) [S1, unofficial][S4].
    *   Starting depth value ($z_{start}$) [S1, unofficial][S4].
    *   Ending depth value ($z_{end}$) [S1, unofficial][S4].
    *   *Note:* Depth values are required for sorting and merging because segments arrive in unpredictable order [S1, unofficial][S4].

**Algorithm Steps:**

1.  **Data Distribution:** Distribute $n$ cells to $P$ processors using round-robin assignment. Each processor stores local cell and node data. In the worst case (no node sharing), this requires $80n$ bytes of transfer for $n$ cells [S1, unofficial].
2.  **Image Partitioning:** Assign successive scanlines to processors in round-robin fashion (scanline interleaving) to disperse ray-merging operations [S1, unofficial].
3.  **Space Partitioning Tree Construction:** Cooperatively construct a $k$-d tree on all processors to restore spatial coherence. The tree must be identical on every processor [S1, unofficial].
    *   *Parallel Partitioning Algorithm:*
        1.  Each processor examines local cells and establishes a cutting position to split sub-regions into equal cell counts. Cut direction alternates at each tree level [S1, unofficial].
        2.  Local positions are sent to a host node, which averages them to a global position and broadcasts it back, along with the next subregion choice [S1, unofficial].
        3.  Cells intersecting a cut are assigned to the region containing their centroid [S1, unofficial].
        4.  Repeat until desired regions are generated [S1, unofficial].
4.  **Rendering Loop:** Processors follow the same path through the spatial partitioning tree. At each leaf node, process all local cells [S1, unofficial][S4].
5.  **Cell Projection and Scan Conversion:** For each local cell:
    *   Project the 3D vertices of the cell onto the 2D image screen [S1, S2, unofficial].
    *   Perform scan conversion to generate ray segments corresponding to the cell's projection [S1, S2, unofficial].
    *   Identify and process every pixel affected by the projection [S2, unofficial].
    *   *Interpolation:* Use piecewise constant or piecewise linear interpolation (based on Nelson Max's work) to determine scalar values for the ray segments [S1, S3, unofficial].
6.  **Routing:** Route the generated ray segments to the processor owning the corresponding image scanline [S1, unofficial][S4].
7.  **Ray Segment Merging:** At each pixel, maintain a linked list of ray segments. Merge segments using the standard Porter-Duff over operator in back-to-front order, using the stored depth values for sorting [S1, unofficial][S4].
    *   Formula: $\alpha = 1 - \exp(-\tau_{avg} l)$ [S1, unofficial][S4].
8.  **Task Management (Polling):** Interleave scan conversion and merging using a polling strategy [S1, unofficial]:
    1.  Scan convert one or more cells.
    2.  Check for incoming ray segments.
    3.  If available, switch to merging task (sort and merge) until no input is pending.
    4.  Repeat [S1, unofficial].
9.  **Asynchronous Communication:** Buffer ray segments locally. Use asynchronous send/receive to overlap computation and communication. Provide two or more buffers per destination. If a send is pending for a full buffer, place new segments in the companion buffer. If both buffers fill, switch to merging to prevent deadlock [S1, unofficial].
10. **Termination:** Use a designated node to coordinate termination. Collect local termination messages and broadcast a global termination signal. Perform a final global synchronization [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cell_projection_rendering(cells, nodes, image_width, image_height, num_processors):
    # 1. Data Distribution
    local_cells = distribute_round_robin(cells, num_processors)
    local_nodes = extract_nodes(local_cells, nodes)
    
    # 2. Image Partitioning (Scanline Interleaving)
    # Each processor owns specific scanlines
    processor_scanlines = assign_scanlines_round_robin(image_height, num_processors)
    
    # 3. Space Partitioning Tree Construction
    # All processors cooperatively build the same k-d tree
    kdtree = build_parallel_kdtree(local_cells, num_processors)
    
    # 4. Rendering Loop
    for leaf_node in kdtree.traverse():
        local_leaf_cells = get_cells_in_leaf(leaf_node, local_cells)
        
        for cell in local_leaf_cells:
            # 5. Cell Projection
            projected_vertices = project_to_screen(cell.vertices, camera_matrix)
            ray_segments = scan_convert_cell(projected_vertices, cell.node_indices, local_nodes)
            
            # 6. Routing
            for segment in ray_segments:
                dest_processor = get_processor_for_scanline(segment.pixel_y, num_processors)
                send_asynchronous(segment, dest_processor, buffer_pool)
                
    # 7. Ray Segment Merging & 8. Task Management
    while not termination_signal_received():
        # Check for incoming segments
        incoming_segments = receive_asynchronous(buffer_pool)
        
        if incoming_segments:
            # Merge incoming segments
            for pixel in processor_scanlines:
                segments_at_pixel = get_segments_for_pixel(pixel, incoming_segments)
                sort_segments_by_depth(segments_at_pixel)
                composite_porter_duff(segments_at_pixel)
                
        # Polling: Scan convert more cells if buffers are not full
        if not buffers_full():
            process_more_cells(local_cells)
            
    # 9. Termination
    send_local_completion_message()
    if is_designated_node():
        wait_for_all_messages()
        broadcast_global_termination()
        
    return final_image
```

**Parameters:**
*   **Polling Interval:** Number of cells processed before checking for incoming ray segments. High frequency prevents buffer clogging but adds overhead; low frequency causes poor asynchronous performance [S1, unofficial].
*   **Buffer Depth:** Number of ray segments accumulated before posting an asynchronous send. Small buffers cause excessive overhead; large buffers cause delays for waiting processors and reduce bandwidth spreading effectiveness [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Deadlock:** If both buffers for a destination fill up before the first send completes, the processor must switch to the ray merging task to process incoming segments and clear congestion [S1, unofficial].
*   **Unbalanced Tree:** The implementation does not guarantee a well-balanced $k$-d tree, but searching overhead is insignificant compared to rendering time [S1, unofficial].
*   **Communication Volume:** Expected $10^6$ to $10^8$ ray segments per image. At 16 bytes per segment, aggregate communication is $10^7$ to $10^9$ bytes per frame [S1, unofficial][S4].

## Subtopics

### [Parallel Cell Projection](875-parallel-cell-projection.md)
A distributed-memory strategy that decomposes the workload into an object-space projection phase and an image-space composition phase, allowing asynchronous communication to hide data transfer overheads.

### [Cell Projection with Texture Mapping for Opacity](993-cell-projection-with-texture-mapping-for-opacity.md)
Uses texture mapping to handle opacity during cell projection.

### [Topological Sort for Cell Projection](999-topological-sort-for-cell-projection.md)
Uses topological sorting to determine rendering order.

### [GPU-based Cell Projection for Structured Data](1864-gpu-based-cell-projection-for-structured-data.md)
Adapts cell projection for GPU architectures and structured data.

### [Cell Projection with Scan-line Conversion](1968-cell-projection-with-scan-line-conversion.md)
Uses scan-line conversion techniques for cell projection.

### [Tetrahedral representation and traversal](4396-tetrahedral-representation-and-traversal.md)
Focuses on tetrahedral cell representation and traversal.

### [Exact AMR Rendering](4399-exact-amr-rendering.md)
Applies cell projection to Adaptive Mesh Refinement (AMR) data.

### [3D Cube Projection and Rasterization](6678-3d-cube-projection-and-rasterization.md)
Projects 3D cubes and rasterizes them.

### [Incoherent Projection](7541-incoherent-projection.md)
Handles incoherent projection scenarios.

## Operating characteristics
*   **Time Complexity:** High computational cost due to the "tremendous number of calculations" required to project every cell and process every affected pixel [S2, unofficial].
*   **Memory Complexity:** Depends on the number of ray segments generated. Each pixel maintains a linked list of segments [S1, unofficial].
*   **Measured Performance:** Rendering 0.5 million tetrahedral cells into a 400x400 pixel image takes 0.5 seconds (2 frames per second) on 128 IBM SP2 nodes (excluding display time) [S1, unofficial].
*   **Scaling:** Scales well on parallel architectures because cells and their projected pixels can be computed independently [S2, unofficial]. Asynchronous communication helps overlap computation and communication [S1, unofficial].
*   **Strengths:** Higher image quality than point sampling methods (no aliasing) [S2, unofficial]; flexibility in data distribution [S1, unofficial][S4]; accuracy over ray-casting or slice/vertex projection [S1, unofficial][S4]; no need for pre-sorting cells in depth order [S1, unofficial].
*   **Limitations:** High communication volume ($10^7$ to $10^9$ bytes per frame) [S1, unofficial][S4]; requires efficient task management to avoid deadlock and buffer clogging [S1, unofficial]; tremendous number of calculations [S2, unofficial].
*   **When to use:** Unstructured data on distributed-memory systems where flexibility and accuracy are prioritized over raw speed [S1, unofficial][S4].
*   **When not to use:** When memory bandwidth is severely constrained or when pre-sorting is feasible and efficient [S1, unofficial].

## Background
Cell-projection rendering was developed as a scalable parallel volume rendering algorithm for three-dimensional unstructured data [S1, unofficial]. It was initially described in the context of supercomputers, using piecewise constant interpolation, with plans to incorporate higher-quality piecewise linear interpolation based on Nelson Max's work [S3, unofficial]. The method is designed to handle the complexity of unstructured grids by projecting cells directly to the image plane, avoiding the need for global depth sorting [S1, unofficial][S4]. It is particularly suited for parallel architectures due to the independent nature of cell processing [S2, unofficial].

## Sources
[S1] DTIC ADA329328: A Scalable Parallel Cell-Projection Volume Rendering Algorithm for Three-Dimensional Unstructured Data, https://archive.org/details/DTIC_ADA329328, fetched 2026-10-26 (unofficial)
[S2] DTIC ADA215533: Sampling Scattered Data Onto Rectangular Grids for Volume Visualization, https://archive.org/details/DTIC_ADA215533, fetched 2026-10-26 (unofficial)
[S3] Volume Visualization of AMR Data, https://graphics.cs.ucdavis.edu/~okreylos/Research/AMRVis/MainPage.html, fetched 2026-10-26 (unofficial)
[S4] PDF A Scalable Parallel Cell-Projection Volume Rendering Algorithm for ..., https://web.cs.ucdavis.edu/~ma/papers/PRS97.pdf, fetched 2026-10-27
