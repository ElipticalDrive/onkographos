# Intensity-based Sparse Voxel Storage

Also called: Detail Coefficient Representation (8-byte), Histogram-based Voxel Buffer

## Overview
Intensity-based Sparse Voxel Storage is a method for representing volumetric data as a set of geometric primitives (voxels) optimized for rasterization by exploiting the redundancy of scalar field values. It solves the problem of high memory bandwidth and storage costs associated with storing intensity values for every voxel by grouping voxels with identical intensity levels into contiguous runs. Unlike sibling methods that manage spatial hierarchy or chunk structures, this approach sorts data by attribute value, using a histogram to map intensity levels to memory offsets, thereby reducing per-voxel storage from 8 bytes to 6 bytes by eliminating redundant intensity attributes [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A scalar volume dataset (intensity values) and a target rendering resolution.
*   **Output:** A vertex buffer containing voxel positions and a histogram structure mapping intensity levels to buffer offsets, ready for GPU rendering.

**Data Structures:**
1.  **Voxel Buffer:** A continuous memory region (created via `ARB_vertex_buffer_object`) storing voxel coordinates. Voxels are sorted by intensity value.
    *   *Layout (Naive):* 8 bytes per voxel. Stores intensity (2 bytes/short) and x, y, z coordinates (2 bytes/short each).
    *   *Layout (Optimized):* 6 bytes per voxel. Stores only x, y, z coordinates (2 bytes/short each). Intensity is not stored per voxel.
2.  **Histogram:** An array of structures, one per unique intensity level.
    *   *Fields:* `begin_offset` (index in voxel buffer), `end_offset` (index in voxel buffer).
    *   *Purpose:* Acts as a run-length encoding, defining the span of voxels sharing a specific intensity.

**Steps:**
1.  **Sort:** Sort all voxels in the volume by their intensity value.
2.  **Build Histogram:** Iterate through the sorted voxels. For each unique intensity value, record the starting and ending indices of the contiguous block of voxels in the buffer.
3.  **Pack Coordinates:** Store the x, y, z coordinates of each voxel into the buffer.
    *   *Optimization:* If 6 bytes per voxel are used, the intensity is removed from the vertex attributes.
    *   *Alternative (4 bytes):* For extreme memory constraints, coordinates can be packed into 32 bits (e.g., 12 bits for X, 10 bits for Y, 10 bits for Z), allowing support for volumes up to $4096 \times 1024 \times 1024$. This requires unpacking in the vertex program using bit shifts or emulation via multiplication and floor functions [S1].
4.  **Render:** For each intensity level $i$:
    *   Retrieve `begin_offset` and `end_offset` from the histogram.
    *   Set a shader constant or texture coordinate to the intensity value $I_i$.
    *   Invoke `glDrawArrays` (or equivalent) to render the span of voxels from `begin_offset` to `end_offset`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def intensity_based_sparse_storage(volume_data, resolution):
    """
    volume_data: 3D array of intensity values.
    resolution: (X, Y, Z) dimensions.
    Returns: voxel_buffer (list of tuples), histogram (dict mapping intensity to (start, end))
    """
    # 1. Flatten and collect voxels
    voxels = []
    for z in range(resolution[2]):
        for y in range(resolution[1]):
            for x in range(resolution[0]):
                intensity = volume_data[z, y, x]
                # Store as (intensity, x, y, z)
                voxels.append((intensity, x, y, z))
    
    # 2. Sort by intensity
    voxels.sort(key=lambda v: v[0])
    
    # 3. Build histogram and optimized buffer
    histogram = {}
    voxel_buffer = []
    
    current_intensity = None
    start_index = 0
    
    for i, (intensity, x, y, z) in enumerate(voxels):
        if intensity != current_intensity:
            # Finalize previous intensity span
            if current_intensity is not None:
                histogram[current_intensity] = (start_index, i)
            
            # Start new span
            current_intensity = intensity
            start_index = i
            
        # Store only coordinates for optimized 6-byte storage
        voxel_buffer.append((x, y, z))
        
    # Finalize last span
    if current_intensity is not None:
        histogram[current_intensity] = (start_index, len(voxels))
        
    return voxel_buffer, histogram

def render_intensity_voxels(histogram, voxel_buffer, shader_constants):
    """
    Simulates the rendering loop.
    """
    for intensity, (start, end) in histogram.items():
        # Set shader constant to the intensity for this span
        shader_constants['current_intensity'] = intensity
        
        # Number of voxels in this span
        count = end - start
        
        # Draw the span
        # In real GL: glDrawArrays(GL_POINTS, start, count)
        render_span(voxel_buffer[start:end])
```

**Parameters:**
*   **Bit Depth:** Typically 16-bit shorts for coordinates. 12/10/10 bit packing is an alternative for 4-byte storage [S1].
*   **Intensity Resolution:** Determined by the number of unique values in the histogram.

**Edge Cases and Pitfalls:**
*   **Redundancy:** Storing intensity with each voxel is redundant because the histogram acts as run-length-encoding. Omitting it saves 2 bytes per voxel [S1].
*   **Memory Limits:** While theoretically supporting volumes up to $65536^3$ with 6-byte storage, practical limits are lower due to available VRAM/RAM [S1].
*   **Bitwise Operations:** On GPUs lacking native bit shift/logical operators, coordinate unpacking from packed formats (e.g., 4-byte) must be emulated using multiplication by fractions and the floor function, which may impact performance [S1].

## Subtopics

### LiteVoxel
An extension to SVRaster that introduces three modules to counteract low-frequency underfitting and memory overgrowth [S2, unofficial]. It uses an Inverse-Sobel Reweighted Loss to shift gradient focus to flat regions after geometric stabilization, Depth-Quantile Pruning with EMA-hysteresis guards to prevent VRAM inflation near silhouettes, and Priority-Driven Subdivision restricted by ray-footprint eligibility [S2, unofficial]. LiteVoxel matches SVRaster’s perceptual quality at 40–60% of peak VRAM, reducing peak VRAM by up to 60% (e.g., from 12 GB to 7.9 GB) with less than 1% loss in PSNR, SSIM, or LPIPS [S2, unofficial].

### Object-Voxel Set Storage for Time-Varying Data
A method for rendering animated volume data by segmenting the volume into objects and storing only voxels belonging to objects of interest in RAM [S3, unofficial]. It keeps multiple time-steps in memory simultaneously, allowing animated data (e.g., 12 animation phases for heart data) to be rendered identically to static data, avoiding memory bandwidth limitations of hardware renderers that reload data for each time-step [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Sorting voxels by intensity is $O(N \log N)$ where $N$ is the number of voxels. Histogram construction is $O(N)$. Rendering time is proportional to the number of unique intensity levels (draw calls).
*   **Memory Complexity:** $O(N)$ for voxel storage. With 6-byte optimization, memory usage is reduced by 25% compared to the naive 8-byte representation [S1].
*   **Performance:** Provides a major speedup for large volumes by reducing memory bandwidth requirements and leveraging GPU vertex buffers [S1].
*   **Strengths:** Simple implementation, efficient for data with many repeated intensity values (e.g., MIP rendering), reduces redundancy.
*   **Limitations:** Requires sorting, which can be costly for dynamic data. Limited by the number of unique intensity levels (draw call overhead). Practical volume size is limited by hardware memory, not theoretical address space [S1].

## Background
The method was described in the context of multiresolution MIP rendering of large volumetric data accelerated on graphics hardware [S1]. It leverages the `ARB_vertex_buffer_object` extension to cache data in high-performance graphics memory, enabling efficient data transfers for GL-drawing commands [S1]. The approach is particularly relevant for static or slowly changing volumetric data where intensity-based grouping is effective.

## Sources
[S1] Multiresolution MIP Rendering of Large Volumetric Data Accelerated on Graphics Hardware, https://doi.org/10.2312/vissym/eurovis07/243-250, fetched 2026-10-26
[S2] Sparse Voxel Rasterization (SVRaster), https://www.emergentmind.com/topics/sparse-voxel-rasterization-svraster, fetched 2026-10-27 (unofficial)
[S3] Distributed Software-Based Volume Visualization in a Virtual Environment, https://doi.org/10.2312/egve/ipt_egve2005/129-140, fetched 2026-10-26 (unofficial)
