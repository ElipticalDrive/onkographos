# Screen Parallel Rendering

Also called: Sort-first parallel rendering

## Overview
Screen Parallel Rendering (SPR) is a parallelization strategy that divides the screen into subscreens, assigning each to a separate processor to exploit parallelism in screen space [S2, unofficial]. It differs from object-parallel rendering by having each processor compute the full ray for its assigned pixels, avoiding the need for image compositing but requiring every processor to hold the entire volume dataset in memory [S2, unofficial].

## How it works
**Inputs:** Screen resolution, number of processors $p$, volume dataset $V$, camera parameters.
**Outputs:** Final composite image.

**Data Structures:**
*   **Screen Partition:** The screen is divided into $p$ disjoint rectangular regions (subscreens).
*   **Volume Data:** The complete volume dataset $V$ is stored in the main memory of every processor.

**Steps:**
1.  **Partition:** Divide the screen into $p$ subscreens.
2.  **Assign:** Assign each subscreen to a distinct processor. Tasks are assigned in a cyclic manner to statically balance the processing workload [S2, unofficial].
3.  **Render:** Each processor independently renders its assigned subscreen. Because each processor takes responsibility for the entire ray for its pixels, existing sequential ray-casting techniques (such as Early Ray Termination) can be applied directly [S2, unofficial].
4.  **Assemble:** Paste the rendered portions together to form the final image [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def screen_parallel_rendering(screen_width, screen_height, num_processors, volume_data, camera):
    # 1. Divide screen into portions
    # Assuming equal height tiles for simplicity
    tile_height = screen_height // num_processors
    
    # 2. Assign tasks in cyclic manner (conceptually distributed to processors)
    # In a real cluster, this involves sending the volume data to each node
    for processor_id in range(num_processors):
        # Each processor loads the ENTIRE volume into its local memory [S2, unofficial]
        local_volume = volume_data 
        
        # Define the y-range for this processor's subscreen
        y_start = processor_id * tile_height
        y_end = (processor_id + 1) * tile_height
        
        # 3. Render each portion separately
        # Each processor computes full rays for its subscreen
        render_subscreen(local_volume, camera, y_start, y_end)
        
    # 4. Assemble/paste portions into the final image [S1, unofficial]
    final_image = assemble_tiles()
    return final_image
```

**Edge Cases and Pitfalls:**
*   **Memory Usage:** This method requires large main memory because every processor must load the entire volume into memory [S2, unofficial]. This makes it unsuitable for large-scale datasets where data decomposition is necessary [S2, unofficial].
*   **Load Balancing:** While cyclic assignment statically balances workloads, it assumes uniform computational cost per pixel. If the volume has varying density or opacity, some processors may finish significantly later than others, leading to idle time [S2, unofficial].

## Operating characteristics
*   **Memory Complexity:** $O(N \times P)$, where $N$ is the size of the volume dataset and $P$ is the number of processors. Every processor stores a full copy of the data [S2, unofficial].
*   **Time Complexity:** $O(T / P)$, where $T$ is the time to render the full screen sequentially, assuming perfect load balancing.
*   **Strengths:** Simple implementation; allows use of sequential optimization techniques like Early Ray Termination (ERT) without modification; no image compositing overhead [S2, unofficial].
*   **Limitations:** High memory bandwidth and storage requirements; poor scalability for large datasets; static load balancing may be inefficient for non-uniform scenes [S2, unofficial].
*   **When to use:** Small datasets that fit in the memory of all processors; scenarios where data decomposition is not desired or feasible [S2, unofficial].
*   **When not to use:** Large-scale datasets requiring data decomposition; memory-constrained environments [S2, unofficial].

## Background
Screen Parallel Rendering is one of the three primary types of parallel rendering techniques identified by Molnar et al., alongside sort-middle and sort-last [S1, unofficial]. It is also known as the "sort-first" approach because the sorting from object space to screen space occurs at the beginning of the pipeline [S1, unofficial]. The Chromium System initially used this approach to distribute rendering work to different nodes in a cluster before the transformation and lighting stages [S1, unofficial].

## Sources
[S1] Volume visualization of time-varying data using parallel, multiresolution and adaptive-resolution techniques, https://scholars.unh.edu/cgi/viewcontent.cgi?article=1245&context=thesis, fetched 2026-10-27 (unofficial)
[S2] Medical Volume Rendering Techniques, https://arxiv.org/pdf/1802.07710v1, fetched 2026-10-27 (unofficial)
