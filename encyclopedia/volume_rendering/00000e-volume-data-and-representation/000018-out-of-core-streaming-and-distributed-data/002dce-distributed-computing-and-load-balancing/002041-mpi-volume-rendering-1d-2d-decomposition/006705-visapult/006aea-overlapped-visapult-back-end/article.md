# Overlapped Visapult Back End

Also called: Overlapped Visapult Back End Implementation, Visapult Back End with Overlapped Data Loading and Rendering

## Overview
The Overlapped Visapult Back End is a method for overlapping volume data loading and rendering within a single distributed processing element (PE) to hide I/O latency. It solves the problem of network-I/O bottlenecks in remote visualization by decoupling the data transfer phase from the rendering phase using a detached reader thread and SystemV shared memory semaphores. It differs from pure MPI-only implementations by avoiding the transmission of large scientific data blocks between separate MPI processes, instead relying on shared memory within a single node, though this introduces potential CPU contention on single-CPU nodes [S1].

## How it works
**Inputs and Outputs:**
- **Input:** A time-varying volume dataset distributed across MPI PEs.
- **Output:** Rendered images transmitted to the Visapult viewer.

**Data Structures:**
1.  **Shared Memory Block:** A memory region shared between the reader thread and the render process.
    -   **Size:** $2 \times S$, where $S$ is the size of a single time step's data.
    -   **Layout:** Double-buffered. Buffer 0 (indices $0$ to $S-1$) and Buffer 1 (indices $S$ to $2S-1$).
    -   **Access Control:** Implicit even-odd decomposition based on the current time step index $t$. If $t$ is even, the reader writes to Buffer 0 and the reader reads from Buffer 1 (or vice versa depending on initialization, but strictly alternating). The sources state: "the reader thread will use one half of the buffer for writing into, while the render process reads from the other half... Access control is implicit as a function of the time step using an even-odd decomposition" [S1].
2.  **Control Variables:** Stored in shared memory, indicating the requested action (read specific time step or exit).
3.  **Semaphores:** Two SystemV shared memory semaphores, `sem_A` and `sem_B`.
    -   `sem_A`: Acts as a barrier for the reader thread.
    -   `sem_B`: Acts as a barrier for the render process.

**Numbered Steps:**
1.  **Initialization:** Each MPI PE creates a pair of SystemV shared memory semaphores (`sem_A`, `sem_B`) and the shared memory block. The semaphores are initialized to allow the reader thread to proceed first (or as defined by the specific implementation, typically `sem_A` starts open, `sem_B` starts closed to enforce ordering).
2.  **Thread Launch:** The MPI process (render process) launches a detached pthread (reader thread).
3.  **Reader Thread Loop:**
    a.  The reader thread blocks waiting for `sem_A`.
    b.  Upon gaining access, it reads the control variable from shared memory.
    c.  If the control variable requests data for time step $t$, the reader thread loads the data for time step $t$ into the appropriate half of the shared memory block (determined by $t \pmod 2$).
    d.  If the control variable requests termination, the reader thread exits.
    e.  After completing the activity, the reader thread posts to `sem_B` and blocks waiting for `sem_A` again.
4.  **Render Process Loop:**
    a.  The render process requests data for time step 0.
    b.  It waits for the data to be loaded (implicitly synchronized by the semaphore logic and buffer availability).
    c.  It requests data for time step 1 from the reader thread (updating the control variable).
    d.  It renders the data for time step 0 from the shared memory buffer.
    e.  Upon rendering completion, it transmits results to the viewer.
    f.  It blocks waiting for `sem_B`.
    g.  Upon gaining access to `sem_B`, it requests the next time step's data (updating the control variable) and posts to `sem_A`.
    h.  Steps d-g repeat until all time steps are processed.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Shared state initialization
shared_mem = allocate_memory(2 * size_of_time_step)
sem_A = create_semaphore(1)  # Reader starts first
sem_B = create_semaphore(0)  # Render waits initially
control_var = { 'action': 'READ', 'time_step': 0 }

def reader_thread():
    while True:
        sem_A.wait()  # Block until render process signals readiness
        action = control_var['action']
        t = control_var['time_step']
        
        if action == 'EXIT':
            break
            
        # Determine buffer index based on even-odd decomposition
        buffer_idx = t % 2
        buffer_start = buffer_idx * size_of_time_step
        
        # Load data for time step t into shared_mem[buffer_start : buffer_start + size_of_time_step]
        load_volume_data(t, shared_mem, buffer_start)
        
        sem_B.post()  # Signal render process that data is ready

def render_process():
    # Request first time step
    control_var['action'] = 'READ'
    control_var['time_step'] = 0
    
    # Initial handoff to reader
    sem_A.post() 
    
    # Loop through time steps
    # Note: The source describes a pipeline where step t is requested while step t-1 is rendered.
    # We must handle the final step carefully.
    
    t = 0
    while True:
        # Wait for data for current time step t to be loaded
        sem_B.wait()
        
        # Render time step t
        buffer_idx = t % 2
        buffer_start = buffer_idx * size_of_time_step
        render_volume(shared_mem, buffer_start, buffer_start + size_of_time_step)
        
        # Transmit results
        transmit_to_viewer()
        
        # Prepare for next time step
        t += 1
        if t >= total_time_steps:
            control_var['action'] = 'EXIT'
            sem_A.post()  # Wake up reader to exit
            break
            
        control_var['time_step'] = t
        sem_A.post()  # Signal reader to load next time step
```

**Edge Cases and Pitfalls:**
-   **CPU Contention:** On distributed memory systems with single-CPU nodes, the reader thread and render process share a single CPU, leading to contention. This may negate the benefits of overlapping if context switching overhead is high [S1].
-   **Synchronization Errors:** Incorrect initialization of semaphores can lead to deadlocks. The reader must block on `sem_A` initially, and the render process must block on `sem_B` initially to ensure the reader loads data before the render process attempts to read it.
-   **Buffer Overwrite:** The even-odd decomposition guarantees that the reader and render process do not access the same buffer simultaneously. If the rendering time exceeds the I/O time for two time steps, the render process might wait for `sem_B` while the reader has already overwritten the buffer. The sources imply a strict pipeline where the render process waits for the reader to finish loading the *current* time step before proceeding, ensuring no overwrite occurs during rendering.

## Operating characteristics
-   **Time Complexity:** The method overlaps I/O and computation. The total time is approximately $\max(\sum I/O_t, \sum Render_t + \sum Transmit_t)$.
-   **Memory Complexity:** $O(S)$ per PE, where $S$ is the size of one time step. The shared memory block is $2S$.
-   **Performance:** On SMP systems with sufficient CPUs, CPU contention is minimized, and performance is improved by hiding I/O latency [S1]. On single-CPU nodes, contention may reduce performance gains.
-   **Strengths:** Avoids large data transfers between MPI processes, reducing network overhead. Simple synchronization using SystemV semaphores.
-   **Limitations:** Tied to shared-memory threading (pthreads) within an MPI process. Not suitable for pure distributed memory nodes without shared memory support. CPU contention on single-core nodes.

## Background
The method was developed as part of the Visapult project, a distributed, parallel volume rendering application designed for interactive visualization of massive datasets over Wide Area Networks (WAN). It was described in a paper by the Visapult team, which detailed the use of high-speed WANs and network data caches to enable remote and distributed visualization. The specific implementation details of the overlapped back end were published in 1999 [S1].

## Sources
[S1] Using High-Speed WANs and Network Data Caches to Enable Remote and Distributed Visualization, https://doi.org/10.5555/370049.370397, fetched 2026-10-26
