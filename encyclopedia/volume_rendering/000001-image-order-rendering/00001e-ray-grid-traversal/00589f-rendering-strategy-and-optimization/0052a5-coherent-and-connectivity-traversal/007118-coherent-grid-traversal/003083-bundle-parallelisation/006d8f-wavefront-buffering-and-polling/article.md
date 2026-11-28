# Wavefront Buffering and Polling

Also called: Asynchronous Ray-Segment Communication, Parallel Volume Rendering with Wavefront Buffering and Polling, Wavefront Cache-friendly Algorithm

## Overview
Wavefront Buffering and Polling is a communication strategy for parallel volume rendering that manages the transfer of ray segments between processors to enable ray merging and early ray termination. It solves the problem of synchronizing asynchronous ray traversal across distributed memory by buffering outgoing segments and periodically polling for incoming data. It sets itself apart from synchronous bulk-transfer methods by spreading communication load over the frame time and allowing processors to revert to receiving mode if blocked from sending, thereby avoiding deadlock [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A volumetric dataset (e.g., tetrahedral grid), viewing parameters, opacity mapping, and processor topology.
*   **Output:** A rendered image frame.
*   **Internal State:** Per-processor buffers for outgoing ray segments and incoming ray segments.

**Data Structures:**
*   **Ray Segment Buffer:** A queue or array holding ray segments. Each segment is 24 bytes [S1, unofficial]. The buffer has a maximum depth (size) $B$.
*   **Polling Interval:** An integer $P$ representing the number of cells to scan-convert before checking for communication.

**Parameters:**
*   **Buffer Size ($B$):** The maximum number of ray segments buffered per processor before forced communication. Typical values range from 25 up to $1.25 \times M$, where $M$ is the expected useful maximum [S1, unofficial].
*   **Expected Useful Maximum ($M$):** The average number of ray segments communicated from each processor to every other processor per frame. This is highly problem-dependent (varies with input data, viewing parameters, opacity mapping, and processor count) [S1, unofficial]. For the tested dataset, $M \approx 1200$ [S1, unofficial].
*   **Polling Interval ($P$):** The number of tetrahedral cells scan-converted between communication checks. Strategies include:
    *   Fixed value (e.g., 160 or 200) [S1, unofficial].
    *   Dynamic: $P = 2 \times B$ [S1, unofficial].

**Algorithm Steps:**
1.  **Initialization:** Initialize ray segment buffers for each processor. Set buffer size $B$ and polling interval $P$.
2.  **Scan Conversion Loop:** For each cell assigned to the processor:
    a.  Scan-convert the cell to generate ray segments.
    b.  Add generated ray segments to the outgoing buffer.
    c.  Increment cell counter.
    d.  If cell counter reaches $P$, proceed to Step 3.
3.  **Polling and Communication:**
    a.  Call termination detection routine to check for global termination [S1, unofficial].
    b.  Check for incoming ray segments from other processors.
    c.  If incoming segments exist, merge them with local ray segments (ray merging) [S1, unofficial].
    d.  If the outgoing buffer is full or if the processor is blocked from sending, flush the buffer to other processors [S1, unofficial].
    e.  Reset cell counter to 0.
4.  **Deadlock Avoidance:** If a processor is blocked from sending (e.g., network congestion), it automatically reverts to a receiving mode, regardless of whether the polling interval $P$ has been reached [S1, unofficial].
5.  **Termination:** When all cells are processed and no more ray segments are expected, finalize the frame.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def wavefront_buffering_and_polling(dataset, processors, buffer_size_B, polling_interval_P):
    # Initialize buffers
    outgoing_buffers = [Buffer(max_size=buffer_size_B) for _ in processors]
    incoming_buffers = [Buffer() for _ in processors]
    
    # Assign cells to processors
    cell_assignments = assign_cells_to_processors(dataset, processors)
    
    for proc_id, cells in cell_assignments.items():
        cell_count = 0
        for cell in cells:
            # Scan-convert cell
            new_segments = scan_convert_cell(cell, dataset)
            
            # Add to outgoing buffer
            outgoing_buffers[proc_id].add(new_segments)
            
            cell_count += 1
            
            # Check polling interval
            if cell_count >= polling_interval_P:
                # Poll for incoming data and termination
                is_terminated = check_termination_detection()
                if is_terminated:
                    break
                
                # Receive incoming segments
                received_segments = receive_from_neighbors(proc_id, incoming_buffers)
                
                # Merge incoming segments with local ray segments
                merge_ray_segments(outgoing_buffers[proc_id], received_segments)
                
                # Send buffered segments if not blocked
                # If blocked, revert to receiving mode (handled by receive logic)
                send_to_neighbors(proc_id, outgoing_buffers[proc_id])
                
                cell_count = 0
        
        # Final flush if needed
        if not outgoing_buffers[proc_id].is_empty():
            send_to_neighbors(proc_id, outgoing_buffers[proc_id])
```

**Edge Cases and Pitfalls:**
*   **Excessively Large Buffer Size:** Eliminates the benefit of spreading communication load, defers all communication to frame end, and causes locality loss in ray merging. Larger buffers deliver many ray segments at once over a wider image area, reducing early merge opportunities, lengthening ray segment lists, and increasing insertion time and memory consumption [S1, unofficial].
*   **Precision:** Not given in the sources.
*   **Boundaries:** Not given in the sources.

## Subtopics

### Bundle Parallelisation
The parent method that groups rays into bundles for parallel processing. Wavefront Buffering and Polling is a communication strategy used within this framework. [Bundle Parallelisation](6293-bundle-parallelisation.md)

## Operating characteristics
*   **Time Complexity:** Not explicitly given. Rendering time scales linearly with pixel count [S1, unofficial].
*   **Memory Complexity:** Not explicitly given. Memory consumption increases with buffer size and ray segment list length [S1, unofficial].
*   **Measured Performance:**
    *   On 511 processors with a large dataset, an average of 44.6 million ray segments are communicated per frame. Aggregate volume is ~1.1 GB (24 bytes per segment) [S1, unofficial].
    *   Ray-segment merging accounts for 46.64% of total computation time (0.949170 s out of 2.035252 s total), making it the most expensive operation, costing >4× scan conversion [S1, unofficial].
    *   Overhead breakdown on 511 processors: termination detection dominates overhead at 9.24% (0.187978 s), followed by receive wait at 7.94% (0.161666 s). Total overhead is 25.98% [S1, unofficial].
    *   Performance sensitivity to buffer depth: Total rendering time variation across tested buffer depths is only ~16%, despite degradation at larger sizes [S1, unofficial].
    *   Image resolution scaling: At 1200×1200, overhead drops to 12.4% due to a larger ray segment buffer (1,000 vs 100) enabling more efficient communication [S1, unofficial].
*   **Strengths:** Spreads communication load over time, avoids deadlock, well-suited for high-bandwidth networks (e.g., SGI T3E) [S1, unofficial].
*   **Limitations:** Poor performance on distributed shared memory architectures (e.g., SGI Origin2000) [S1, unofficial]. Ray merging is a dominant cost [S1, unofficial].
*   **When to Use:** High-performance clusters with high network bandwidth (e.g., SGI T3E, Linux PC clusters) [S1, unofficial].
*   **When Not to Use:** Distributed shared memory architectures with poor message passing software or memory management (e.g., SGI Origin2000) [S1, unofficial].

## Background
*   **Origin:** The method was developed and tested on SGI T3E (up to 511 processors), SGI Origin2000, and a 32-node Linux PC cluster [S1, unofficial]. It outperforms the T3E at moderate processor counts on the Linux cluster [S1, unofficial].
*   **History:** The algorithm is part of a wavefront cache-friendly approach for parallel volume rendering. Initial tests on SGI Origin2000 yielded poor results, consistent with findings in other literature [S1, unofficial].
*   **Usage:** Used for direct volume rendering of high-resolution datasets, such as flow density and velocity magnitude in aerodynamic simulations [S1, unofficial].

## Sources
[S1] DTIC ADA370411: Wavefront Cache-friendly Algorithm for Compact Numerical Schemes, https://archive.org/details/DTIC_ADA370411, fetched 2026-10-26 (unofficial)
