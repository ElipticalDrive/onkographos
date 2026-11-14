# Host-Slave Implementation with Interrupt-Driven Communication

Also called: Host-Slave partition, Interrupt-driven host-slave

## Overview
This method is a parallel processing scheme for volume rendering where a host node handles I/O and data preparation, while slave nodes handle the computationally intensive fitting and polygon generation. It solves the idle time and communication overhead problems of standard host-slave partitions by using asynchronous, interrupt-driven communication to overlap computation with data transfer [S1, unofficial]. It sets itself apart from its sibling [Asynchronous Communication with Double-Buffering](4280-asynchronous-communication-with-double-buffering.md) by specifically employing an interrupt handler on the host to manage background subcube transmission and by implementing specific strategies like exponential subcube reduction and early communication requests to minimize latency [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Volume data slices, rendering parameters (approximation order, e.g., (3,3,3)).
*   **Output:** Rendered image composed of polygons sent from slaves to the host.

**Data Structures:**
*   **C3 Data:** Computed by the host. Requires 27 pixel information to compute the C3 data of a single pixel [S1, unofficial].
*   **Subcubes:** Chunks of volume data transferred from host to slaves.
*   **Polygons:** Computed by slaves, stored in local memory, and sent to the host.
*   **Communication Buffers:** Two buffers are used for asynchronous message passing.

**Parameters:**
*   **Preload Slices:** For (1,1,1) approximation, preload 2 slices. For (3,3,3) approximation, preload 3 slices [S1, unofficial].
*   **Buffer Count:** 2 buffers for asynchronous communication [S1, unofficial].

**Numbered Steps:**

1.  **Host Initialization:**
    *   Read the first slice $i$ and compute its C3 data [S1, unofficial].
    *   Start the interrupt handler to enable background subcube transmission [S1, unofficial].

2.  **Host Main Loop (for each slice):**
    *   Read the next slice $i+1$ and compute its C3 data [S1, unofficial].
    *   Wait for all subcubes of the "old" slice $i$ to be transferred to slave nodes by the interrupt handler [S1, unofficial].
    *   While waiting, the host may help process additional subcubes to utilize idle time [S1, unofficial].
    *   Increment slice index and repeat.

3.  **Slave Processing (Concurrent with Host):**
    *   Request and get subcubes from the host [S1, unofficial].
    *   Fit (3,3,3) functions to the subcube data [S1, unofficial].
    *   Compute polygons based on the fitted functions [S1, unofficial].
    *   Send polygons to the host [S1, unofficial].

**Communication Strategies:**
*   **Asynchronous Communication:** Alternates between two buffers. When `send_message` is called, the node returns immediately and packs data into the other buffer, waiting only if the previous message is not completed [S1, unofficial].
*   **Early Communication:** When a slave is almost finished processing subcubes, it sends a request to the host to process newly arrived subcubes without delay, reducing communication latency [S1, unofficial].
*   **Message Packing:** The host packs a couple of rows of subcubes into one transmission packet. Shared boundary pixels are sent only once instead of four times (since a pixel can be shared by 4 subcubes) [S1, unofficial].
*   **Exponential Subcube Reduction:** The host sends large quantities of subcubes at the beginning of a slice and reduces the number sent exponentially when the slice is almost done to avoid long idle time waiting for a slave [S1, unofficial].
*   **Local Memory Management:** Slaves store calculated polygons in local memory and empty them to the host when they accumulate to a certain level [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Global variables
buffers = [Buffer(), Buffer()]
buffer_index = 0
interrupt_handler_active = False
subcube_queue = []
polygon_buffer = []

def send_message(data, buffer_idx):
    """
    Asynchronous send. Returns immediately.
    If previous message in buffer_idx is not done, wait.
    """
    if not buffers[buffer_idx].is_ready():
        wait_for_completion(buffer_idx)
    buffers[buffer_idx].pack(data)
    buffers[buffer_idx].send()

def interrupt_handler():
    """
    Runs in background. Handles incoming requests and sends subcubes.
    """
    global buffer_index
    while interrupt_handler_active:
        if slave_request_received():
            # Pack subcubes into the other buffer
            next_buffer = 1 - buffer_index
            subcubes = get_next_subcubes()
            send_message(subcubes, next_buffer)
            buffer_index = next_buffer
        if polygon_data_available():
            polygons = get_polygons()
            send_message(polygons, buffer_index)

def host_algorithm():
    """
    Host Node's Interrupt Driven Algorithm
    """
    global interrupt_handler_active
    
    # Step 1: Read slice i and compute its C3 data
    slice_i = read_slice(0)
    c3_data_i = compute_c3_data(slice_i)
    
    # Step 2: Start interrupt handler
    interrupt_handler_active = True
    start_interrupt_handler(interrupt_handler)
    
    slice_idx = 0
    
    while more_slices():
        # Step 3: Read new slice i+1 and compute its C3 data
        slice_idx += 1
        slice_next = read_slice(slice_idx)
        c3_data_next = compute_c3_data(slice_next)
        
        # Step 4: Wait for all subcubes of the "old" slice i to be transferred
        # The subcubes prepared by the new slice i+1 is transferred at the next iteration
        wait_for_subcube_transfer(slice_idx - 1)
        
        # Help process subcubes while waiting
        process_additional_subcubes()
        
        # Prepare subcubes for next iteration
        # Note: In a real implementation, subcubes would be queued here
        # based on c3_data_next and sent via interrupt handler
        queue_subcubes(c3_data_next)

def slave_algorithm():
    """
    Slave Node's Algorithm
    """
    while True:
        # Step 1: Request and get subcubes from the host
        subcubes = request_and_get_subcubes()
        
        # Step 2: Fit (3,3,3) functions
        fitted_functions = fit_3x3x3_functions(subcubes)
        
        # Step 3: Compute polygons
        polygons = compute_polygons(fitted_functions)
        
        # Store in local memory
        polygon_buffer.append(polygons)
        
        # Empty polygons to host when accumulated
        if len(polygon_buffer) > threshold:
            send_message(polygon_buffer, buffer_index)
            polygon_buffer.clear()
            
        # Step 4: Send polygons to the host (handled by send_message above)
        
        # Early communication: Request next subcubes if almost finished
        if almost_finished():
            send_request_to_host()
```

## Subtopics

### [Multi-Host Partition Scheme](14583-multi-host-partition-scheme.md)
A parallel volume rendering strategy that divides processing nodes into independent groups, each containing one host and multiple slaves, to process subsets of volume slices. It solves the load imbalance and I/O bottlenecks of pure symmetry partitions and the single-host computing bottleneck of pure host-slave schemes by allowing dynamically reassigning idle nodes from finished groups to busy ones.

## Operating characteristics
*   **Time Complexity:** The host performs I/O and C3 data computation, which is relatively fast compared to polygon generation. The slaves perform the heavy computation. The asynchronous nature overlaps communication with computation, reducing total time.
*   **Memory Complexity:** Requires local memory on slaves to store polygons until a threshold is reached. Two buffers are used for communication.
*   **Performance:** Preloading slices reduces idle time dominance. For (3,3,3) approximation, communication time becomes less important compared to computation time. Near linear speed up is achieved with preloading [S1, unofficial].
*   **Strengths:** Minimizes idle time for both host and slaves. Reduces communication overhead through message packing and asynchronous transfer.
*   **Limitations:** Complex implementation due to interrupt-driven software. Critical section violations can cause hard-to-trace bugs [S1, unofficial].
*   **When to use:** When communication overhead is significant and the host can be utilized for data preparation and background communication management.
*   **When not to use:** When the implementation complexity is prohibitive or when the communication latency is negligible compared to computation time.

## Background
This method originates from the work on Scalar Field Modeling & Visualization on the Intel Delta [S1, unofficial]. It was developed to alleviate problems associated with the host-slave partition in parallel volume rendering, specifically idle time and communication overhead. The approach uses an interrupt handler to manage background subcube transmission, allowing the host to remain productive while waiting for slaves.

## Sources
[S1] Scalar Field Modeling & Visualization on the Intel Delta', https://docs.lib.purdue.edu/cgi/viewcontent.cgi?article=2138&context=cstech, fetched 2026-10-26 (unofficial)
