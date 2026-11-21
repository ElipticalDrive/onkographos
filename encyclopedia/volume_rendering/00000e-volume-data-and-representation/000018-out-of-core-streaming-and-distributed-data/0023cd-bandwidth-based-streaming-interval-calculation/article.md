# Bandwidth-based Streaming Interval Calculation

Also called: N_skip calculation, t_delta calculation

## Overview
Bandwidth-based Streaming Interval Calculation determines the minimum time interval between data fetches and the number of simulation slices to skip based on system bandwidth limits to ensure real-time streaming from HDD to GPU [S1, unofficial]. It solves the problem of loading large volumetric datasets that exceed available GPU and RAM capacity by calculating a safe skip count that guarantees data arrives before it is needed [S1, unofficial]. Unlike [Sectioning](1636-sectioning.md) which partitions spatial data, or [Hardware-Assisted Volume Streaming](2119-hardware-assisted-volume-streaming.md) which relies on specific hardware features, this method is purely bandwidth-driven and operates on a multi-layered ring buffer architecture to hide transfer latency [S1, unofficial].

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
1.  **Host Layer:** A ring buffer of 6 items in heap memory, where each item stores one flow slice [S1, unofficial]. Items 2 and 4 are the center items containing slices before and after the current function time $t_{\text{func}}$ [S1, unofficial]. A queue manages more than 6 items for smooth buffering [S1, unofficial].
2.  **Device Layer:** A ring buffer of 4 items, where each item is a composition of one host-layer item and a chunk of allocated device memory [S1, unofficial]. The buffer is managed by a CUDA queue, and items 1 and 2 are the center items corresponding to time slices around $t_{\text{func}}$ [S1, unofficial].

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
    Where $N_{\text{skip}}$ is the number of slices to skip, defined as a floating-point value [S1, unofficial].

**Parameters**
*   $B_{\text{HDD} \rightarrow \text{RAM}}$: Typical value 1.2 GB/s [S1, unofficial].
*   $B_{\text{RAM} \rightarrow \text{GPU}}$: Typical value 1.6 GB/s [S1, unofficial].
*   $m_{\text{app}}$: Range $[0, 1]$ or $\mathbb{R}$ [S1, unofficial].
*   $m_{\text{func}}$: Range $\mathbb{R}$ [S1, unofficial].

**Steps**
1.  Calculate $B_{\min}$ using the minimum of HDD-to-RAM and RAM-to-GPU throughput [S1, unofficial].
2.  Calculate $t_\Delta$ by dividing the size of a single time slice by $B_{\min}$ [S1, unofficial].
3.  Calculate $t_S$ using the simulation start/end times and number of slices [S1, unofficial].
4.  Calculate $N_{\text{skip}}$ using $t_\Delta$, $t_S$, $m_{\text{app}}$, and $m_{\text{func}}$ [S1, unofficial].
5.  **Host Layer Update:** Check if $t_{\text{func}}$ has moved outside the valid interval defined by the center items in the positive time direction [S1, unofficial]. If so, pop item 0 from the left of the host ring buffer and push it back to the right [S1, unofficial]. Insert a new item (ID 5) into the host buffer [S1, unofficial]. Schedule this new item to load a future time slice from the hard disk, where the slice ID is determined by $N_{\text{skip}}$ relative to the current position [S1, unofficial].
6.  **Device Layer Update:** Upon insertion of a new item into the host ring buffer, pop one item from the left of the device ring buffer [S1, unofficial]. Send the associated memory chunk of the popped item back to the CUDA queue [S1, unofficial]. Copy the previously last (fully loaded) host-buffer item by reference to the device layer [S1, unofficial]. Combine this copy with a CUDA memory object and schedule it for asynchronous memory transfer [S1, unofficial].
7.  **Synchronization:** After popping a new item from the host buffer queue, block the host thread if the associated host memory is still subject to device or host layer transfer [S1, unofficial]. After popping a new item from the device buffer queue, block the host thread if the associated host memory is still subject to host layer transfer, while device layer transfers are synchronized by assigning a unique CUDA stream object to every device layer item [S1, unofficial]. After performing all tasks on both layers, synchronize device layer items with IDs 1 and 2 before launching CUDA to assert that time slices are fully loaded when accessed [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class StreamItem:
    """Represents a single time slice buffer item."""
    def __init__(self, item_id, time, data_ptr=None):
        self.id = item_id
        self.time = time
        self.data_ptr = data_ptr
        self.transferring = False

def create_new_item(item_id, time):
    """Standard allocation of a new stream buffer item."""
    return StreamItem(item_id, time)

def calculate_target_slice_id(current_id, n_skip):
    """Standard offset calculation for the next slice to load."""
    return current_id + int(n_skip)

def block_host_thread():
    """Standard synchronization wait until transfer completes."""
    pass

def calculate_streaming_intervals(B_hdd_ram, B_ram_gpu, size_slice, t0, t1, N_slices, m_app, m_func):
    """Calculate t_delta and N_skip for bandwidth-based streaming."""
    B_min = min(B_hdd_ram, B_ram_gpu)
    t_delta = size_slice / B_min
    t_S = (t1 - t0) / (N_slices - 1)
    N_skip = (t_delta / t_S) * m_app * m_func
    return t_delta, N_skip

def host_layer_update(ring_buffer, t_func, N_skip, queue, current_slice_id):
    """Manage the host layer ring buffer (size 6)."""
    # Check if t_func leaves the valid interval between items 2 and 3 in positive time direction
    if t_func > ring_buffer[3].time:
        # Pop item 0 from the left and push it back to the right
        item = ring_buffer.pop(0)
        ring_buffer.append(item)
        
        # Insert new item (ID 5)
        new_item = create_new_item(5, t_func)
        ring_buffer.append(new_item)
        
        # Schedule load for future time slice determined by N_skip
        target_slice_id = calculate_target_slice_id(current_slice_id, N_skip)
        queue.schedule_disk_load(target_slice_id, new_item)

def device_layer_update(host_ring_buffer, device_ring_buffer, cuda_queue):
    """Manage the device layer ring buffer (size 4)."""
    # Upon insertion of new item into host buffer, pop from device buffer
    if len(device_ring_buffer) > 0:
        popped_item = device_ring_buffer.pop(0)
        # Send memory chunk back to CUDA queue
        cuda_queue.release_memory(popped_item.data_ptr)
        
        # Copy the previously last (fully loaded) host-buffer item by reference
        last_host_item = host_ring_buffer[-1]
        
        # Combine with CUDA memory object
        device_item = StreamItem(last_host_item.id, last_host_item.time)
        device_item.data_ptr = cuda_queue.allocate_memory()
        
        # Schedule for asynchronous memory transfer
        cuda_queue.launch_async_copy(last_host_item.data_ptr, device_item.data_ptr, size_slice)

def synchronize_layers(host_ring_buffer, device_ring_buffer, cuda_streams):
    """Synchronization rules to handle deviations from constant transfer rates."""
    # 1. Host Layer Synchronization
    for item in host_ring_buffer:
        if item.transferring:
            block_host_thread()
            
    # 2. Device Layer Synchronization
    for item in device_ring_buffer:
        if item.transferring:
            block_host_thread()
            
    # 3. CUDA Synchronization
    # Synchronize device layer items with IDs 1 and 2 before launching CUDA
    cuda_streams[1].synchronize()
    cuda_streams[2].synchronize()
```

**Edge Cases and Pitfalls**
*   **Assumption Violation:** Using $t_\Delta$ directly for skip calculation assumes constant transfer rates and function time [S1, unofficial]. User input or system load can disturb these values, requiring the synchronization rules to handle deviations [S1, unofficial].
*   **Memory Scarcity:** Queue items in both host and device layers consume allocated memory even when unused [S1, unofficial]. On the device layer, memory is scarce, so the number of items should be kept to a minimum [S1, unofficial].
*   **Transfer Time Discrepancy:** The actual transfer of a single time slice takes roughly twice $t_\Delta$ [S1, unofficial]. This is safe because $t_{\text{func}}$ is at least two items away from the insertion point, allowing enough time for completion [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The calculation of $t_\Delta$ and $N_{\text{skip}}$ is $O(1)$ [S1, unofficial]. The streaming overhead is amortized over time, aiming for real-time performance [S1, unofficial].
*   **Memory Complexity:** The method uses a fixed number of buffer items: 6 in the host layer and 4 in the device layer, consuming memory proportional to the slice size plus CUDA queue overhead [S1, unofficial].
*   **Performance:** The method allows rendering of large simulations (e.g., 100 time slices of $128^3$ points = 2,500 MB) that would otherwise exceed GPU/RAM capacity [S1, unofficial].
*   **Strengths:** It is independent of specific hardware features beyond standard PCIe bandwidth and CUDA support [S1, unofficial]. It handles arbitrary simulation sizes by streaming [S1, unofficial].
*   **Limitations:** It is dependent on system specifications (bandwidth) [S1, unofficial]. Performance may degrade if the bottleneck bandwidth is significantly lower than expected or if system load varies [S1, unofficial].
*   **When to use:** When rendering large volumetric datasets that cannot fit in GPU memory and require real-time or near-real-time interaction [S1, unofficial].
*   **When not to use:** When the dataset fits entirely in GPU memory, direct upload is simpler and faster [S1, unofficial].

## Background
The method was described by Dirk Sowinski in his 2012 Master's Thesis "Real-Time Smoke Volume Rendering" [S1, unofficial]. It was developed to enable real-time visualization of large fluid simulation data sets (e.g., Square Cylinder data) that exceed the capacity of modern graphics cards at the time [S1, unofficial]. The approach uses a multi-layered streaming architecture (Host and Device layers) with asynchronous transfers to hide latency [S1, unofficial].

## Sources
[S1] Real-Time Smoke Volume Rendering, http://v22015081501727525.yourvserver.net/webpage/dirkfiles/betreutearbeiten/Sowinski_MA_2012.pdf, fetched 2026-10-31 (unofficial)
