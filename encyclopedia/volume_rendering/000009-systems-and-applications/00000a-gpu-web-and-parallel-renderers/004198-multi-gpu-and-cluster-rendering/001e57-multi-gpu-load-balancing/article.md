# Multi-GPU Load Balancing

Also called: Multi-Context GPU Rendering, Multi-GPU Rendering

## Overview
Multi-GPU Load Balancing distributes volumetric rendering workloads across multiple graphics processing units (GPUs) to support large datasets and achieve near-optimal speedup [S1]. It addresses the limitation of single-GPU memory and compute capacity by splitting the data and compositing results, distinct from cluster-based approaches that rely on network communication [S1][S2, unofficial]. Unlike single-node multi-GPU server architectures which may focus on hardware topology, this method specifically manages the rendering context and data partitioning logic [S1].

## How it works
The method relies on creating separate OpenGL contexts for each GPU and interleaving data to balance the load during progressive refinement [S1].

**Inputs and Outputs:**
- **Input:** A sorted array of voxels (or detail coefficients from a multi-resolution pyramid) and a scene definition.
- **Output:** A single composite image.

**Data Structures:**
- **Voxel Data:** For streaming MIP, each voxel requires 6 bytes: 4 bytes for position (with pyramid level rolled into two bits) and 2 bytes for intensity [S1].
- **Histogram:** Split alongside the coefficients to support transfer function mapping [S1].

**Steps:**
1. **Context Creation:** Create an OpenGL context on each GPU's associated X-screen (or display device) [S1].
2. **Data Partitioning:** Split the sorted array of voxels equally among the GPUs. The split is performed by **interleaving** the detail coefficients (and histogram) for each pyramid level, rather than splitting the array in the middle [S1].
3. **Rendering Loop:**
   a. Switch the active rendering context to GPU 1.
   b. Render the assigned interleaved subset of data.
   c. Switch the active rendering context to GPU 2.
   d. Render the assigned interleaved subset.
   e. Repeat for all GPUs [S1].
4. **Data Transfer (Intermediate):** Since no direct data shuffling extension exists between cards, intermediate results must pass through the CPU [S1].
   a. On the first GPU's context, read the result using `glReadPixels`.
   b. On the second GPU's context, draw the result using `glDrawPixels` [S1].
   c. Preferred pixel format for this transfer is `GL_UNSIGNED_INT_8_8_8_8` [S1].
5. **Compositing:** Combine the resulting images from the GPUs using a **pixel-wise maximum operator** [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def multi_gpu_rendering(gpus, voxel_data, histogram):
    # 1. Partition data by interleaving
    # Assuming 2 GPUs for simplicity
    gpu0_data, gpu1_data = interleave_split(voxel_data)
    gpu0_hist, gpu1_hist = interleave_split(histogram)

    # 2. Render on GPU 0
    ctx0 = gpus[0].create_context()
    ctx0.bind()
    ctx0.set_data(gpu0_data, gpu0_hist)
    image0 = ctx0.render()
    
    # 3. Transfer to CPU
    # Note: glReadPixels is used here
    cpu_buffer_0 = ctx0.read_pixels(format=GL_UNSIGNED_INT_8_8_8_8)

    # 4. Render on GPU 1
    ctx1 = gpus[1].create_context()
    ctx1.bind()
    ctx1.set_data(gpu1_data, gpu1_hist)
    
    # 5. Combine GPU 0 result with GPU 1 rendering
    # Draw GPU 0's result into GPU 1's context
    ctx1.draw_pixels(cpu_buffer_0, format=GL_UNSIGNED_INT_8_8_8_8)
    
    # Render GPU 1's own data on top
    image1 = ctx1.render()
    
    # 6. Final Composition
    # Pixel-wise maximum operator
    final_image = pixel_wise_max(image0, image1)
    
    return final_image

def interleave_split(data):
    """
    Splits data by interleaving elements for progressive refinement balance.
    Returns two lists: even-indexed elements and odd-indexed elements.
    """
    even = []
    odd = []
    for i, item in enumerate(data):
        if i % 2 == 0:
            even.append(item)
        else:
            odd.append(item)
    return even, odd
```

**Parameters:**
- **Pixel Format:** `GL_UNSIGNED_INT_8_8_8_8` is preferred for CPU transfers [S1].

**Edge Cases and Pitfalls:**
- **No Direct Inter-GPU Transfer:** Direct shuffling between cards is not supported; data must traverse the CPU [S1].
- **Threading Overhead:** Using multiple threads for rendering loops does not yield performance gains because the algorithm is GPU-bound; CPU synchronization overhead outweighs the minimal CPU load [S1].

## Subtopics

### Parallel streaming model with GPU-CPU matching
Matches each GPU to a specific CPU core within a parallel streaming model. This approach supports multiple GPUs per node and aims to keep communication, CPU/GPU data transfers, disk access, and GPU kernel execution concurrent [S2, unofficial].

## Operating characteristics
- **Performance:** Achieves almost optimal speedup for large volumes [S1].
- **Complexity:** The load balancing strategy (interleaving) is designed to better divide the load during progressive refinement compared to simple mid-array splitting [S1].
- **Limitations:** The method is susceptible to load imbalance if the data distribution is not uniform, as the system speed is bounded by the slowest unit [S2, unofficial]. Sort-first algorithms are particularly susceptible to this [S2, unofficial].
- **Hardware Dependency:** Relies on drivers supporting multiple cards for one X-server for multiple screens (e.g., newer NVIDIA drivers) [S1].

## Background
The method was developed to support large volumetric datasets on graphics hardware, specifically utilizing multi-resolution MIP rendering [S1]. It emerged from research into accelerating volume visualization on consumer-grade multi-GPU setups where cluster solutions were not feasible [S1]. The interleaving strategy was identified as superior to contiguous splitting for progressive rendering scenarios [S1].

## Sources
[S1] Multiresolution MIP Rendering of Large Volumetric Data Accelerated on Graphics Hardware, https://doi.org/10.2312/vissym/eurovis07/243-250, fetched 2026-10-26
[S2] Volume Visualization and Compositing on Large-Scale Displays Using Handheld Touchscreen Interaction, https://doi.org/10.25781/kaust-d5m7l, fetched 2026-10-26 (unofficial)
