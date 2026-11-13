# Bandwidth-based Streaming Interval Calculation

Also called: N_skip calculation, t_delta calculation

## Overview
Bandwidth-based Streaming Interval Calculation is a method for determining the minimum time interval between data fetches and the number of simulation slices to skip, based on system bandwidth limits to ensure real-time streaming from HDD to GPU [S1, unofficial]. It solves the problem of loading large volumetric datasets that exceed available GPU and RAM capacity by calculating a safe skip count ($N_{\text{skip}}$) that guarantees data arrives before it is needed [S1, unofficial]. Unlike [Sectioning](1636-sectioning.md) which partitions spatial data, or [Hardware-Assisted Volume Streaming](2119-hardware-assisted-volume-streaming.md) which relies on specific hardware features, this method is purely bandwidth-driven and operates on a multi-layered ring buffer architecture to hide transfer latency [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs:**
    *   $B_{\text{HDD} \rightarrow \text{RAM}}$: Throughput from hard disk to RAM [S1, unofficial].
    *   $B_{\text{RAM} \rightarrow \text{GPU}}$: Throughput from RAM to GPU memory [S1, unofficial].
    *   $\text{size}(\mathcal{S})$: Size of a single time slice in bytes [S1, unofficial].
    *   $t_0$: Start time of the simulation [S1, unofficial].
    *   $t_1$: End time of the simulation [S1, unofficial].
    *   $N_S$: Number of slices in the simulation [S1, unofficial].
    *   $m_{\text{app}}$: Application time multiplier [S1, unofficial].
    *   $m_{\text{func}}$: Function time multiplier [S1, unofficial].
*   **Outputs:**
    *   $t_\Delta$: Minimum time interval between fetches [S1, unofficial].
    *   $N_{\text{skip}}$: Number of slices to skip when scheduling new loads [S1, unofficial].

**Data Structures**
The method relies on two ring buffers:
1.  **Host Layer:** A ring buffer of 6 items in heap memory. Each item stores one flow slice. Items 2 and 4 are the center items containing slices before and after the current function time $t_{\text{func}}$. A queue manages more than 6 items for smooth buffering [S1, unofficial].
2.  **Device Layer:** A ring buffer of 4 items. Each item is a composition of one host-layer item and a chunk of allocated device memory, managed by a CUDA queue. Items 1 and 2 are the center items corresponding to time slices around $t_{\text{func}}$ [S1, unofficial].

**Formulas**
1.  **Minimum Bandwidth:**
    $$B_{\min} = \min(B_{\text{HDD} \rightarrow \text{RAM}}, B_{\text{RAM} \rightarrow \text{GPU}})$$
    Where $B_{\min}$ is the bottleneck bandwidth [S1, unofficial].

2.  **Minimum Time Interval:**
    $$t_\Delta = \frac{\text{size}(\mathcal{S})}{B_{\min}}$$
    Where $t_\Delta$ is the minimum time required to transfer one slice [S1, unofficial].

3.  **Data Time Spacing:**
    $$t_S = \frac{t_1 - t_0}{N_S - 1}$$
    Where $t_S$ is the time difference between adjacent slices [S1, unofficial].

4.  **Skip Count:**
    $$N_{\text{skip}} = \frac{t_\Delta}{t_S} \cdot m_{\text{app}} \cdot m_{\text{func}}$$
    Where $N_{\text{skip}}$ is the number of slices to skip. This is a floating-point value [S1, unofficial].

**Parameters**
*   $B_{\text{HDD} \rightarrow \text{RAM}}$: Typical value 1.2 GB/s [S1, unofficial].
*   $B_{\text{RAM} \rightarrow \text{GPU}}$: Typical value 1.6 GB/s [S1, unofficial].
*   $m_{\text{app}}$: Range $[0, 1]$ or $\mathbb{R}$ [S1, unofficial].
*   $m_{\text{func}}$: Range $\mathbb{R}$ [S1, unofficial].

**Steps**
1.  Calculate $B_{\min}$ using the minimum of HDD-to-RAM and RAM-to-GPU throughput.
2.  Calculate $t_\Delta$ by dividing the size of a single time slice by $B_{\min}$.
3.  Calculate $t_S$ using the simulation start/end times and number of slices.
4.  Calculate $N_{\text{skip}}$ using $t_\Delta$, $t_S$, $m_{\text{app}}$, and $m_{\text{func}}$.
5.  **Host Layer Update:**
    *   Check if $t_{\text{func}}$ has moved outside the valid interval defined by the center items (IDs 2 and 4) in the positive time direction.
    *   If so, pop item 0 from the left of the host ring buffer and push it back to the right.
    *   Insert a new item (ID 5) into the host buffer.
    *   Schedule this new item to load a future time slice from the hard disk. The ID of the slice to load is determined by $N_{\text{skip}}$ relative to the current position [S1, unofficial].
6.  **Device Layer Update:**
    *   Upon insertion of a new item into the host ring buffer, pop one item from the left of the device ring buffer.
    *   Send the associated memory chunk of the popped item back to the CUDA queue.
    *   Copy the previously last (fully loaded) host-buffer item by reference to the device layer.
    *   Combine this copy with a CUDA memory object and schedule it for asynchronous memory transfer [S1, unofficial].
7.  **Synchronization:**
    *   **Host Layer:** After popping a new item from the host buffer queue, block the host thread if the associated host memory is still subject to device or host layer transfer [S1, unofficial].
    *   **Device Layer:** After popping a new item from the device buffer queue, block the host thread if the associated host memory is still subject to host layer transfer. Device layer transfers are synchronized by assigning a unique CUDA stream object to every device layer item [S1, unofficial].
    *   **CUDA:** After performing all tasks on both layers, synchronize device layer items with IDs 1 and 2 before launching CUDA. This asserts that time slices are fully loaded when accessed [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def calculate_streaming_intervals(B_hdd_ram, B_ram_gpu, size_slice, t0, t1, N_slices, m_app, m_func):
    """
    Calculate t_delta and N_skip for bandwidth-based streaming.
    """
    # 1. Calculate Minimum Bandwidth
    B_min = min(B_hdd_ram, B_ram_gpu)
    
    # 2. Calculate Minimum Time Interval
    t_delta = size_slice / B_min
    
    # 3. Calculate Data Time Spacing
    # Note: Source defines t_S as (t1 - t0) / (N_S - 1)
    t_S = (t1 - t0) / (N_slices - 1)
    
    # 4. Calculate Skip Count
    N_skip = (t_delta / t_S) * m_app * m_func
    
    return t_delta, N_skip

def host_layer_update(ring_buffer, t_func, N_skip, queue):
    """
    Manage the host layer ring buffer (size 6).
    """
    # Check if t_func leaves the valid interval between center items (IDs 2 and 4)
    # in positive time direction.
    if t_func > ring_buffer[3].time: # Assuming item 3 is the upper bound of the valid interval
        # Pop item 0 from the left
        item = ring_buffer.pop(0)
        # Push it back to the right
        ring_buffer.append(item)
        
        # Insert new item (ID 5)
        new_item = create_new_item()
        ring_buffer.append(new_item)
        
        # Schedule load for future time slice determined by N_skip
        target_slice_id = calculate_target_slice_id(new_item.id, N_skip)
        queue.schedule_disk_load(target_slice_id, new_item)

def device_layer_update(host_ring_buffer, device_ring_buffer, cuda_queue):
    """
    Manage the device layer ring buffer (size 4).
    """
    # Upon insertion of new item into host buffer, pop from device buffer
    if len(device_ring_buffer) > 0:
        popped_item = device_ring_buffer.pop(0)
        # Send memory chunk back to CUDA queue
        cuda_queue.release_memory(popped_item.memory_chunk)
        
        # Copy the previously last (fully loaded) host-buffer item by reference
        last_host_item = host_ring_buffer[-1]
        
        # Combine with CUDA memory object
        device_item = DeviceItem(host_item=last_host_item, memory=cuda_queue.allocate_memory())
        
        # Schedule for asynchronous memory transfer
        cuda_queue.schedule_async_transfer(device_item)

def synchronize_layers(host_ring_buffer, device_ring_buffer, cuda_streams):
    """
    Synchronization rules to handle deviations from constant transfer rates.
    """
    # 1. Host Layer Synchronization
    # After popping a new item from host buffer queue, block if memory is still transferring
    for item in host_ring_buffer:
        if item.is_being_transferred():
            block_host_thread()
            
    # 2. Device Layer Synchronization
    # After popping from device buffer, block if host memory is still transferring
    for item in device_ring_buffer:
        if item.host_memory_is_being_transferred():
            block_host_thread()
            
    # Device layer transfers are synchronized via unique CUDA streams per item
    # 3. CUDA Synchronization
    # Synchronize device layer items with IDs 1 and 2 before launching CUDA
    cuda_streams[1].synchronize()
    cuda_streams[2].synchronize()
```

**Edge Cases and Pitfalls**
*   **Assumption Violation:** Using $t_\Delta$ directly for skip calculation assumes constant transfer rates and function time. User input or system load can disturb these values, requiring the synchronization rules to handle deviations [S1, unofficial].
*   **Memory Scarcity:** Queue items in both host and device layers consume allocated memory even when unused. On the device layer, memory is scarce, so the number of items should be kept to a minimum [S1, unofficial].
*   **Transfer Time Discrepancy:** The actual transfer of a single time slice takes roughly twice $t_\Delta$. This is safe because $t_{\text{func}}$ is at least two items away from the insertion point, allowing enough time for completion [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The calculation of $t_\Delta$ and $N_{\text{skip}}$ is $O(1)$. The streaming overhead is amortized over time, aiming for real-time performance.
*   **Memory Complexity:** The method requires fixed-size buffers: 6 items in the host layer and 4 items in the device layer. The total memory usage is proportional to the size of a single time slice $\times 10$ (plus overhead for CUDA queues) [S1, unofficial].
*   **Performance:** The method allows rendering of large simulations (e.g., 100 time slices of $128^3$ points = 2,500 MB) that would otherwise exceed GPU/RAM capacity [S1, unofficial].
*   **Strengths:** It is independent of specific hardware features beyond standard PCIe bandwidth and CUDA support. It handles arbitrary simulation sizes by streaming [S1, unofficial].
*   **Limitations:** It is dependent on system specifications (bandwidth). Performance may degrade if the bottleneck bandwidth is significantly lower than expected or if system load varies [S1, unofficial].
*   **When to use:** When rendering large volumetric datasets that cannot fit in GPU memory and require real-time or near-real-time interaction [S1, unofficial].
*   **When not to use:** When the dataset fits entirely in GPU memory, direct upload is simpler and faster [S1, unofficial].

## Background
The method was described by Dirk Sowinski in his 2012 Master's Thesis "Real-Time Smoke Volume Rendering" [S1, unofficial]. It was developed to enable real-time visualization of large fluid simulation data sets (e.g., Square Cylinder data) that exceed the capacity of modern graphics cards at the time. The approach uses a multi-layered streaming architecture (Host and Device layers) with asynchronous transfers to hide latency [S1, unofficial].

## Sources
[S1] Real-Time Smoke Volume Rendering, http://v22015081501727525.yourvserver.net/webpage/dirkfiles/betreutearbeiten/Sowinski_MA_2012.pdf, fetched 2026-10-31 (unofficial)
