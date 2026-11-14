# Host-Slave Implementation with Interrupt-Driven Communication

Also called: Host-Slave partition, Interrupt-driven host-slave

## Overview
A parallel processing scheme where a host node handles I/O and data preparation while slave nodes handle computation, using asynchronous, interrupt-driven communication to minimize idle time and overhead [S1, unofficial]. It sets itself apart from synchronous host-slave models by allowing the host to process subcubes and handle interrupts concurrently with data transfer, and from double-buffering variants by explicitly employing an interrupt handler to manage background transmission and request processing [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Volume data slices, approximation order (e.g., (1,1,1) or (3,3,3)).
*   **Output:** Rendered image polygons.

**Data Structures:**
*   **C3 Data:** Computed per-pixel data requiring 27 neighboring pixel values [S1, unofficial].
*   **Subcubes:** Blocks of volume data sent to slaves.
*   **Polygons:** Computed geometric primitives sent back to the host.
*   **Buffers:** Two buffers used for asynchronous message packing [S1, unofficial].

**Parameters:**
*   **Preload Slices:** 2 slices for (1,1,1) approximation; 3 slices for (3,3,3) approximation [S1, unofficial].
*   **Chunk Size:** Large quantities at slice start, reducing exponentially near slice end [S1, unofficial].

**Steps:**

1.  **Host Initialization:** Preload the host with the specified number of slices (2 for (1,1,1), 3 for (3,3,3)) to reduce idle time dominance [S1, unofficial].
2.  **Host Step 1 (Read & Compute):** Read slice $i$ and compute its C3 data. This step requires 27 pixel information to compute the C3 data of a pixel [S1, unofficial].
3.  **Host Step 2 (Interrupt Start):** Start the interrupt handler to allow background subcube transmission. This step returns immediately without waiting [S1, unofficial].
4.  **Host Step 3 (Next Slice):** Read new slice $i+1$ and compute its C3 data [S1, unofficial].
5.  **Host Step 4 (Wait & Assist):** Wait for all subcubes of the "old" slice $i$ to be transferred to slave nodes by the interrupt-driven handler. The subcubes prepared by the new slice $i+1$ are transferred in the next iteration. While waiting, the host helps process additional subcubes [S1, unofficial].
6.  **Host Step 5 (Loop):** Go to Step 3 [S1, unofficial].
7.  **Slave Step 1 (Request):** Request and get subcubes from the host. Slaves send a request to the host when almost finished processing current subcubes to process newly arrived subcubes without delay [S1, unofficial].
8.  **Slave Step 2 (Fit):** Fit (3,3,3) functions [S1, unofficial].
9.  **Slave Step 3 (Compute):** Compute polygons [S1, unofficial].
10. **Slave Step 4 (Send):** Send polygons to the host. Slaves store calculated polygons in local memory and empty them to the host when they accumulate to a certain level [S1, unofficial].

**Communication Strategy:**
*   **Asynchronous Communication:** Alternates between two buffers. When `send_message` is called, the node returns immediately and packs data into the other buffer, waiting only if the previous message is not completed [S1, unofficial].
*   **Early Communication:** Slaves request new subcubes before finishing current ones to reduce communication latency [S1, unofficial].
*   **Message Packing:** Pack a couple of rows of subcubes into one transmission packet to reduce the number of transferred packets and total bytes. Shared boundary pixels are sent only once instead of four times (since a pixel can be shared by 4 subcubes) [S1, unofficial].
*   **Interrupt-Driven Host:** Use an interrupt handler to process communication requests, allowing the host to listen to slaves and hand out subcubes/receive polygons even while doing other work [S1, unofficial].
*   **Exponential Reduction:** The host sends large quantities of subcubes at the beginning of a slice and reduces the number sent exponentially when the slice is almost done to avoid long idle time waiting for a slave [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Global state
buffers = [Buffer(), Buffer()]
buffer_index = 0
interrupt_handler_active = False
subcube_queue = []
polygon_buffer = []

def host_algorithm():
    # Preload slices
    preload_slices(2 if approximation == (1,1,1) else 3)
    
    slice_i = 0
    while more_slices():
        # Step 1: Read slice i and compute C3 data
        read_slice(slice_i)
        compute_c3_data(slice_i)
        
        # Step 2: Start interrupt handler for background transmission
        start_interrupt_handler()
        
        # Step 3: Read new slice i+1 and compute C3 data
        read_slice(slice_i + 1)
        compute_c3_data(slice_i + 1)
        
        # Step 4: Wait for old slice i subcubes to be transferred
        # Also help process subcubes while waiting
        wait_for_transfer(slice_i)
        process_additional_subcubes()
        
        # Step 5: Go to step 3 (loop)
        slice_i += 1

def start_interrupt_handler():
    global interrupt_handler_active
    interrupt_handler_active = True
    # Interrupt handler runs in background
    # It handles:
    # 1. Receiving slave requests
    # 2. Sending subcubes to slaves
    # 3. Receiving polygons from slaves
    pass

def slave_algorithm():
    while more_subcubes():
        # Step 1: Request and get subcubes
        if almost_finished():
            send_request_to_host()
        subcubes = get_subcubes_from_host()
        
        # Step 2: Fit (3,3,3) functions
        fit_functions(subcubes)
        
        # Step 3: Compute polygons
        polygons = compute_polygons()
        
        # Step 4: Send polygons to host
        # Store in local memory, empty when accumulated
        polygon_buffer.append(polygons)
        if len(polygon_buffer) > threshold:
            send_polygons_to_host(polygon_buffer)
            polygon_buffer = []

def send_message(data, buffer_idx):
    # Asynchronous send
    # If previous message not completed, wait
    if not is_message_complete(buffer_idx):
        wait_for_completion(buffer_idx)
    
    # Pack data into buffer
    buffers[buffer_idx].pack(data)
    
    # Trigger transmission
    buffers[buffer_idx].send()
    
    # Switch buffer index
    buffer_idx = 1 - buffer_idx
    return buffer_idx

def interrupt_handler():
    # Process communication requests
    # Hand out subcubes
    # Receive polygons
    # Called asynchronously
    pass
```

**Edge Cases and Pitfalls:**
*   **Critical Section Violations:** Interrupt-driven software is tricky to implement; hard-to-trace bugs may occur intermittently due to critical section violations [S1, unofficial].
*   **Idle Time:** If preload slices are not used, idle time becomes a large portion of total processing time, especially with many groups [S1, unofficial].
*   **Boundary Pixels:** Shared boundary pixels must be sent only once to avoid redundant transmission (4 times if sent per subcube) [S1, unofficial].

## Subtopics

### [Multi-Host Partition Scheme](14583-multi-host-partition-scheme.md)
A parallelization strategy where nodes are divided into groups, each with one host and several slaves, processing a subset of slices. It is a hybrid of host-slave and symmetry partitions, allowing load balancing by reassigning finished group nodes as slaves to other groups.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Achieves near linear speed up when preloading hosts with 2 images for (1,1,1) approximation [S1, unofficial]. For (3,3,3) approximation with 65 slices of 192x128 resolution, slave idle time is smaller compared to (1,1,1) because computation time is longer, making communication time less important [S1, unofficial].
*   **Scaling:** Not given in the sources.
*   **Strengths:** Minimizes idle time for both host and slaves; overlaps computation with communication; reduces communication overhead via message packing and early requests.
*   **Limitations:** Complex implementation due to interrupt handling and critical section management; potential for intermittent bugs.
*   **When to Use:** When communication overhead is significant and computation can be effectively partitioned between host and slaves.
*   **When Not to Use:** When implementation complexity is a concern or when computation time is very short relative to communication time.

## Background
Originated in the context of scalar field modeling and visualization on the Intel Delta architecture [S1, unofficial]. The method addresses problems associated with traditional host-slave partitions, such as idle host and idle slave nodes, by introducing asynchronous, interrupt-driven communication and specific data management strategies [S1, unofficial].

## Sources
[S1] Scalar Field Modeling & Visualization on the Intel Delta', https://docs.lib.purdue.edu/cgi/viewcontent.cgi?article=2138&context=cstech, fetched 2026-10-26 (unofficial)
