# Parallel Volume Rendering for Unstructured Grids

Also called: Cell-projection rendering with multiplexed asynchronous communication, Parallel volume renderer for 3D unstructured-grid data

## Overview
A parallel volume rendering algorithm for visualizing large-scale unstructured-grid data (e.g., tetrahedral meshes) that uses cell-projection, fine-grained pixel interleaving, and asynchronous communication to achieve scalability and load balancing [S1, unofficial]. It differs from its parent method by replacing scanline image partitioning with fine-grained pixel interleaving to handle high cell density regions, and by employing a logarithmic-time distributed termination detection protocol that eliminates the host processor bottleneck [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Large-scale unstructured grid data (e.g., tetrahedral meshes) with scalar values at grid points; viewing parameters (camera position, orientation, field of view); image resolution.
*   **Output:** Final composited image displayed on a host processor.

**Data Structures:**
*   **Volume Data:** Distributed among processors. Each processor holds a subset of tetrahedra.
*   **k-d Tree:** A hierarchical spatial partitioning structure built in parallel to restore spatial locality lost during data distribution. Used for efficient traversal and ray-segment generation [S1, unofficial].
*   **Ray-Segment Buffer:** A fixed-size buffer on each processor to store incoming ray segments for merging. Size depends on dataset scale (e.g., 80 for small, 100 for large) [S1, unofficial].
*   **Subimage:** Each processor maintains a portion of the final image corresponding to its assigned pixels.

**Parameters:**
*   **Number of Processors ($P$):** Tests performed with up to 511 rendering processors (plus one coordinator) [S1, unofficial].
*   **Image Resolution:** 400 x 400 pixels in tests [S1, unofficial].
*   **Ray-Segment Buffer Size:** 80 (small dataset), 100 (large dataset) [S1, unofficial].
*   **Polling Interval:** 160 (small dataset), 200 (large dataset) [S1, unofficial].

**Steps:**

1.  **Data Distribution:** Distribute volume data cells among processors in a round-robin fashion. This disperses nearby cells widely to average out load imbalances caused by varying cell sizes and shapes [S1, unofficial].
2.  **Spatial Partitioning:** Perform a parallel, synchronized preprocessing step to partition the volume data into a hierarchical representation using a k-d tree. This restores spatial locality lost during distribution, improving image compositing efficiency and reducing runtime memory requirements [S1, unofficial].
3.  **Image Partitioning:** Partition the image using a fine-grained pixel interleaving scheme. This distributes high-density cell regions over more processors, mitigating load imbalance that occurs with scanline interleaving as the number of processors approaches the number of scanlines [S1, unofficial].
4.  **Ray Segment Generation (Scan Conversion):** Each processor traverses the k-d tree in a fixed order. It scan converts its local cells to produce ray segments without pre-sorting cells in depth order. The ray segments are routed to their final destinations in image space [S1, unofficial].
5.  **Multiplexed Execution:** Interleave scan conversion and ray-segment merging using a polling strategy:
    *   Scan convert one or more cells.
    *   Periodically check (poll) for incoming ray segments.
    *   If segments are available, switch to merging (sorting and merging) until no input is pending.
    *   Use a double-buffering scheme with asynchronous send and receive operations to amortize communication overheads and overlap communication with rendering computations [S1, unofficial].
6.  **Subimage Completion:** After scan conversion and merging are finished for all assigned pixels, each processor has a completed subimage.
7.  **Image Assembly:** Each processor sends its completed subimage to a host processor for assembly and display [S1, unofficial].
8.  **Termination Detection:** Employ a distributed termination detection protocol using a binary merging algorithm based on ray-segment counts. This determines frame completion in logarithmic time $O(\log P)$ without involving the host processor [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Assumptions:
# - Processors are indexed 0 to P-1.
# - Processor 0 is the coordinator/host.
# - Data is distributed round-robin.
# - k-d tree is built and available on all processors.
# - Image is partitioned by fine-grained pixel interleaving.

def parallel_volume_rendering():
    # Step 1: Data Distribution (Handled by Coordinator, assumed done)
    # Each processor has local_tetrahedra and local_pixels
    
    # Step 2: Spatial Partitioning (Parallel Preprocessing)
    # Build k-d tree on distributed data
    kd_tree = build_kd_tree_parallel(local_tetrahedra)
    
    # Step 3: Image Partitioning (Handled by Coordinator, assumed done)
    # Each processor knows its assigned pixels
    
    # Step 4 & 5: Multiplexed Scan Conversion and Merging
    # Initialize buffers
    ray_segment_buffer = []
    subimage = initialize_subimage(local_pixels)
    
    # Polling interval and buffer size are parameters
    polling_interval = get_polling_interval()
    buffer_size = get_buffer_size()
    
    # Main loop: Process assigned pixels
    for pixel in local_pixels:
        # Generate ray for pixel
        ray = create_ray(pixel)
        
        # Scan convert local tetrahedra using k-d tree
        # Traverse k-d tree, find intersections with local tetrahedra
        local_segments = scan_convert_ray(ray, kd_tree, local_tetrahedra)
        
        # Add local segments to buffer
        ray_segment_buffer.extend(local_segments)
        
        # Check if buffer is full or polling interval reached
        if len(ray_segment_buffer) >= buffer_size or (pixel_index % polling_interval == 0):
            # Merge incoming ray segments
            # Sort segments by depth
            sorted_segments = sort_by_depth(ray_segment_buffer)
            
            # Composite segments into subimage
            subimage = composite_segments(subimage, sorted_segments, ray)
            
            # Clear buffer
            ray_segment_buffer = []
            
            # Asynchronously receive incoming segments from other processors
            # This overlaps communication with computation
            incoming_segments = async_receive_segments()
            if incoming_segments:
                ray_segment_buffer.extend(incoming_segments)
                # Merge incoming segments immediately if available
                sorted_incoming = sort_by_depth(ray_segment_buffer)
                subimage = composite_segments(subimage, sorted_incoming, ray)
                ray_segment_buffer = []
    
    # Step 6: Final Merge for remaining segments
    if ray_segment_buffer:
        sorted_segments = sort_by_depth(ray_segment_buffer)
        subimage = composite_segments(subimage, sorted_segments, ray)
        
    # Step 7: Send subimage to host
    if processor_id != 0:
        send_subimage_to_host(subimage, host_id=0)
    else:
        # Host collects subimages
        all_subimages = [subimage]
        for p in range(1, P):
            all_subimages.append(receive_subimage_from_processor(p))
        final_image = assemble_image(all_subimages)
        display(final_image)
        
    # Step 8: Distributed Termination Detection
    # Binary merging based on ray-segment counts
    terminate = distributed_termination_detection()
    if terminate:
        shutdown()

def distributed_termination_detection():
    # Binary merging algorithm
    # Each processor reports ray-segment count
    # Counts are merged in a binary tree fashion
    # Result is known in O(log P) time
    # No host involvement
    pass
```

**Edge Cases and Pitfalls:**
*   **Load Imbalance:** Using scanline interleaving with high cell density regions causes load imbalance as the number of processors approaches the number of scanlines. Mitigated by fine-grained pixel interleaving [S1, unofficial].
*   **Memory Requirements:** Minimum 42 processors are required to render the large dataset due to memory constraints [S1, unofficial].
*   **Precision:** Data is reduced from double precision to single precision to fit in memory [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Termination detection is $O(\log P)$, compared to $O(P)$ in the previous version [S1, unofficial].
*   **Memory Complexity:** Reduced runtime memory requirements due to k-d tree partitioning [S1, unofficial].
*   **Performance:**
    *   Rendering rate: Almost 9 million tetrahedra/second using 511 processors on SGI/Cray T3E [S1, unofficial].
    *   Parallel overhead: 26% at 511 processors [S1, unofficial].
    *   Parallel efficiency: 70% or better for up to 128 processors on IBM SP2 (previous study) [S1, unofficial].
*   **Scalability:** Performance with the large dataset increases steadily through 511 processors. Performance with the small dataset peaks around 320 processors and deteriorates beyond that point [S1, unofficial].
*   **Strengths:** Handles large-scale unstructured grids; good load balancing with fine-grained pixel interleaving; efficient termination detection; overlaps communication with computation.
*   **Limitations:** Requires minimum number of processors for large datasets; performance may degrade for small datasets with very high processor counts [S1, unofficial].
*   **When to Use:** Large-scale unstructured grid visualization where load balancing and scalability are critical.
*   **When Not to Use:** Small datasets with very high processor counts may see performance degradation; structured grids may not benefit from the overhead of k-d tree construction and cell-projection [S1, unofficial].

## Background
*   **Origin:** Developed by Kwan-Liu Ma and Thomas W. Crockett at NASA Langley Research Center and ICASE [S1, unofficial].
*   **History:** Presented in 1999. An earlier version was tested on IBM SP2 [S1, unofficial]. This version improved upon the previous algorithm by using fine-grained pixel interleaving and a logarithmic-time termination detection protocol [S1, unofficial].
*   **Usage:** Used on the SGI/Cray T3E operated by NASA’s Goddard Space Flight Center for visualizing large-scale aerodynamics calculations [S1, unofficial].

## Sources
[S1] DTIC ADA370411: Wavefront Cache-friendly Algorithm for Compact Numerical Schemes, https://archive.org/details/DTIC_ADA370411, fetched 2026-10-26 (unofficial)
