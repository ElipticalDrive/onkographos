# Multi-GPU Load Balancing

Also called: Multi-Context GPU Rendering, Multi-GPU Rendering

## Overview
Multi-GPU Load Balancing distributes volumetric rendering workloads across multiple graphics processing units (GPUs) on a single node to support large datasets and achieve near-optimal speedup [S1]. It solves the problem of memory and computational limits on single GPUs by splitting the data structure and compositing results via the CPU [S1]. It differs from [Single Node Multi-GPU/Server Architecture](5165-single-node-multi-gpu-server-architecture.md) by focusing specifically on the algorithmic splitting of sorted voxel arrays and the synchronization overhead of combining results, rather than general server topology or cluster networking.

## How it works
**Inputs and Outputs:**
*   **Input:** A sorted array of voxels (typically organized in a multiresolution pyramid with detail coefficients and histograms) and a volume dataset.
*   **Output:** A final composite image.

**Data Structures:**
*   **Voxel Array:** Sorted array of voxels. For multiresolution MIP, each voxel entry includes position (4 bytes) and intensity (2 bytes), with the pyramid level encoded in two bits of the position attribute, totaling 6 bytes per voxel [S1].
*   **Split Strategy:** The sorted array is split equally between GPUs by interleaving detail coefficients and histograms for each pyramid level, rather than splitting the array in the middle [S1].

**Steps:**
1.  **Context Creation:** Create a separate OpenGL context for each GPU, mapped to distinct X-screens (if using NVIDIA drivers supporting multiple screens per X-server) [S1].
2.  **Data Distribution:** Distribute the interleaved detail coefficients and histograms to each GPU's context [S1].
3.  **Rendering Loop:**
    *   Render the volume using one GPU context.
    *   Switch to the second GPU context.
    *   Render the volume using the second GPU context.
    *   *Note:* Rendering is switched between contexts once per frame. Multi-threading the rendering loops does not yield performance gains because the algorithm is GPU-bound; CPU synchronization overhead outweighs the minimal load of queuing commands [S1].
4.  **Result Combination:**
    *   Read the intermediate result from the first GPU using `glReadPixels` [S1].
    *   Transfer this data through the CPU.
    *   Write the data to the second GPU context using `glDrawPixels` [S1].
    *   Combine the results using a pixel-wise maximum operator [S1].
    *   *Note:* The second GPU does not need to be connected to a monitor, as its result is combined with the first [S1].

**Parameters:**
*   **Pixel Format:** `GL_UNSIGNED_INT_8_8_8_8` is the preferred format for CPU transfer between contexts [S1].

**Pitfalls:**
*   **No Direct GPU-to-GPU Transfer:** There is no extension for direct data shuffling between cards; intermediate results must pass through the CPU [S1].
*   **CPU Bottleneck:** The CPU is only responsible for queuing commands and data. Spreading this tiny load over multiple CPUs via multi-threading does not outweigh synchronization overhead [S1].
*   **Splitting Strategy:** Splitting the array in the middle is inferior to interleaving coefficients, as interleaving better divides the load during progressive refinement [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Initialization
gpu_contexts = [create_gl_context(i) for i in range(num_gpus)]
# Data is interleaved: GPU 0 gets indices 0, 2, 4... GPU 1 gets 1, 3, 5...
# of the sorted voxel array's detail coefficients and histograms

def render_frame():
    # Render on GPU 0
    bind_context(gpu_contexts[0])
    render_volume(gpu_contexts[0].data)
    
    # Read result from GPU 0
    # Preferred format: GL_UNSIGNED_INT_8_8_8_8
    result_gpu0 = gl_read_pixels(format=GL_UNSIGNED_INT_8_8_8_8)
    
    # Render on GPU 1
    bind_context(gpu_contexts[1])
    render_volume(gpu_contexts[1].data)
    
    # Combine results via CPU
    # Note: glDrawPixels is used to transfer result_gpu0 to GPU 1 context
    # This step is necessary because no direct GPU-to-GPU shuffle exists
    bind_context(gpu_contexts[1])
    gl_draw_pixels(result_gpu0, format=GL_UNSIGNED_INT_8_8_8_8)
    
    # Final combination is pixel-wise maximum
    final_image = pixel_wise_max(result_gpu0, gpu_contexts[1].current_framebuffer)
    
    return final_image
```

## Subtopics

### [Parallel streaming model with GPU-CPU matching](9376-parallel-streaming-model-with-gpu-cpu-matching.md)
Defines hardware mapping and concurrency for multi-GPU systems, enforcing 1:1 CPU-GPU affinity to prevent contention and enable true parallel streaming.

## Operating characteristics
*   **Performance:** Achieves "almost optimal speedup" for large volumes [S1].
*   **Limitations:** The method is GPU-bound. CPU synchronization overhead for multi-threading rendering loops negates any potential gains from parallelizing the command queueing [S1].
*   **Hardware Constraints:** Relies on specific driver capabilities (e.g., NVIDIA drivers supporting multiple screens for one X-server) to manage multiple contexts [S1].
*   **Data Transfer Overhead:** Significant overhead exists due to the requirement to pass intermediate results through the CPU (`glReadPixels`/`glDrawPixels`) as no direct GPU-to-GPU data shuffling extension exists [S1].

## Background
This method originates from research on multiresolution MIP rendering of large volumetric data accelerated on graphics hardware [S1]. It addresses the limitations of single-GPU rendering by leveraging multiple cards within a single system, utilizing the X-server's ability to manage multiple screens for multiple graphics cards [S1].

## Sources
[S1] Multiresolution MIP Rendering of Large Volumetric Data Accelerated on Graphics Hardware, https://doi.org/10.2312/vissym/eurovis07/243-250, fetched 2026-10-26
