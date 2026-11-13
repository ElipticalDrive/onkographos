# Accumulative Rendering

Also called: Accumulative rendering mode

## Overview
Accumulative rendering is a specialized rendering mode designed for scenarios where the input dataset is acquired incrementally, such as in live microscopy or intraoperative imaging [S1]. It solves the latency problem of conventional volume rendering by allowing immediate visualization of new data without reprocessing the entire volume [S1]. Unlike standard ray casting which requires the full dataset to be present before rendering can begin, accumulative rendering combines the previous rendering output directly with a newly acquired slice, limiting computational load to only two slices regardless of the total dataset size [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A newly acquired slice of volumetric data ($S_{new}$) and the current accumulated rendering result ($C_{prev}$).
*   **Output:** The updated accumulated rendering result ($C_{new}$) and the display-ready image.

**Data Structures:**
The method utilizes four Static Random-Access Memory (SRAM) blocks acting as framebuffers [S1].
*   **Compositing Buffers (2 blocks):** Used for reading the previous result and writing the new accumulated result.
*   **Display Buffers (2 blocks):** Used for presenting the final image to the user.
*   **Memory Layout:** Voxels are arranged in memory as cubes (cubic memory organization) to facilitate efficient access, although cubic memory is specifically noted as not employed for rendering under this mode if arbitrary views were supported; however, the source confirms cubic memory is used for the dataset storage context [S1]. The system relies on a single data bus architecture, necessitating the dual-buffer design to overcome simultaneous read/write limitations [S1].

**Algorithm Steps:**
1.  **Initialization (First Slice):**
    *   The system receives the first slice of data.
    *   Memory writing only: One SRAM block from the compositing pair is accessed to store the acquired slice. No compositing occurs as there is no previous result.
    *   The result is copied to a display buffer.

2.  **Subsequent Stages (Incremental Update):**
    *   For each new slice $S_{new}$ acquired:
    *   **Compositing:** The system performs simultaneous read and write operations using the four SRAM blocks. Two blocks facilitate the compositing operation, while the other two serve as display buffers.
    *   **Interchange:** SRAM blocks interchange roles between memory reading and memory writing to ensure a continuous flow of data. This allows the system to read the previous accumulated result and write the new result without time-multiplexing on a single data bus [S1].
    *   **Formula Application:** The new accumulated color/value is computed using the compositing formula:
        $$ C_{new} = C_{prev} + S_{new} $$
        Where $C_{prev}$ is the accumulated result from the previous step and $S_{new}$ is the newly acquired slice data [S1].
    *   **Display Update:** The resulting $C_{new}$ is made available to the display buffers for visualization.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Global state
# framebuffers: list of 4 SRAM blocks [F0, F1, F2, F3]
# F0, F1 are compositing buffers
# F2, F3 are display buffers
# active_read_idx: index into [0, 1] for current read buffer
# active_write_idx: index into [0, 1] for current write buffer
# display_idx: index into [2, 3] for current display buffer

def initialize():
    # Clear all framebuffers
    for fb in framebuffers:
        fb.clear()
    active_read_idx = 0
    active_write_idx = 1
    display_idx = 2

def process_new_slice(new_slice_data):
    global active_read_idx, active_write_idx, display_idx
    
    # Determine which buffers are currently in use
    read_buf = framebuffers[active_read_idx]
    write_buf = framebuffers[active_write_idx]
    disp_buf = framebuffers[display_idx]
    
    # Step 1: Compositing
    # If this is the first slice, read_buf is empty/zero, so C_prev is 0
    # Formula: C_new = C_prev + S_new
    # Perform pixel-wise addition of new_slice_data into read_buf, storing result in write_buf
    for each pixel p:
        write_buf[p] = read_buf[p] + new_slice_data[p]
        
    # Step 2: Swap roles for next iteration
    # The buffer we just wrote to (active_write_idx) becomes the read buffer for next time
    # The buffer we just read from (active_read_idx) becomes free or can be reused
    # Simple swap of indices for the compositing pair
    active_read_idx, active_write_idx = active_write_idx, active_read_idx
    
    # Step 3: Display
    # Copy the result from the write buffer (which is now the active result) to the display buffer
    # Note: The source mentions simultaneous read/write to overcome bus limits. 
    # In practice, we might just point the display to the write buffer if supported, 
    # or copy if not. The source says "two serve as display buffers" and "interchange... 
    # providing simultaneous reading and writing". 
    # To be safe and generic:
    disp_buf.copy_from(write_buf)
    
    # Update display index for next frame if we want to ping-pong displays
    display_idx = 3 if display_idx == 2 else 2
```

**Parameters and Edge Cases:**
*   **Projection:** Currently supported only for orthographic projection rendering [S1].
*   **Viewing Direction:** The viewing direction is always assumed to be perpendicular to the slices [S1]. Arbitrary viewing directions are not supported in this mode [S1].
*   **Precision:** Not given in the sources.
*   **Pitfall:** The limitation to orthographic projection means the method cannot be used for perspective views or arbitrary angle inspections without significant refinement [S1].

## Operating characteristics
*   **Time Complexity:** $O(1)$ with respect to the total dataset size per frame, as it only processes two slices (the current accumulated result and the new slice) [S1].
*   **Memory Complexity:** Requires four SRAM blocks for framebuffers. The source notes that cubic memory organization increases memory requirements to approximately eight times the dataset size, but this is for the dataset storage context, not necessarily the rendering buffers themselves [S1]. The rendering buffers are fixed at 4 blocks.
*   **Measured Performance:** The prototype system was deployed at the National Cancer Center of Singapore (NCCS) for clinical experiments, demonstrating the feasibility of concurrent acquisition and rendering [S1].
*   **Strengths:** Eliminates the wait time for full dataset acquisition before rendering begins; allows immediate visualization of incremental data; reduces computational load significantly for large datasets [S1].
*   **Limitations:** Restricted to orthographic projection; viewing direction must be perpendicular to slices; not suitable for arbitrary view configurations [S1].
*   **When to use:** Live microscopy, intraoperative imaging, or any scenario with incremental data acquisition where immediate feedback is critical and orthographic views are sufficient [S1].
*   **When not to use:** When perspective views or arbitrary viewing angles are required [S1].

## Background
Accumulative rendering was introduced in the context of a prototype system built for online volume rendering of incrementally acquired LSCEM (Light Sheet Confocal Endomicroscopy) images for superficial oral cancer detection [S1]. The system was deployed at the National Cancer Center of Singapore (NCCS) [S1]. The method was developed to address the inefficiency of conventional practices where the full dataset must be completed before rendering can start, by allowing imaging and rendering to occur concurrently [S1].

## Sources
[S1] Online volume rendering of incrementally accumulated LSCEM images for superficial oral cancer detection, https://doi.org/10.5306/wjco.v2.i4.179, fetched 2026-10-30
