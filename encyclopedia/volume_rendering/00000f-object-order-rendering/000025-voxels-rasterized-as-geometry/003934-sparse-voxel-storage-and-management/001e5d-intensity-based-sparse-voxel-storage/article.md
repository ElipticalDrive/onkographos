# Intensity-based Sparse Voxel Storage

Also called: Detail Coefficient Representation (8-byte), Histogram-based Voxel Buffer

## Overview
Intensity-based Sparse Voxel Storage is a method for representing volumetric data as geometric primitives optimized for rasterization by exploiting scalar field value redundancy [S1]. It reduces memory bandwidth and storage costs by grouping voxels with identical intensity levels into contiguous runs, using a histogram to map intensity levels to memory offsets [S1]. Unlike [SVO LOD Management](2990-svo-lod-management.md), [Sparse Voxel Rasterization](3032-sparse-voxel-rasterization.md), [Voxel Chunk Management](13837-voxel-chunk-management.md), and [Voxel Dropout](15702-voxel-dropout.md) which manage spatial hierarchy or chunk structures, this approach sorts data by attribute value and eliminates redundant per-voxel intensity attributes [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A scalar volume dataset containing intensity values [S1].
*   **Output:** A vertex buffer containing voxel positions and a histogram structure mapping intensity levels to buffer offsets, ready for GPU rendering [S1].

**Data Structures:**
1.  **Voxel Buffer:** A continuous memory region created via the `ARB_vertex_buffer_object` extension [S1]. Voxels are stored in a continuous region sorted per intensity value [S1].
    *   *Layout (Naive):* 8 bytes per voxel, storing intensity and x, y, z coordinates as 16-bit shorts [S1].
    *   *Layout (Optimized):* 6 bytes per voxel, storing only x, y, z coordinates as 16-bit shorts [S1]. Intensity is not stored per voxel and is instead passed via a shader constant or texture coordinate [S1].
    *   *Layout (4-byte alternative):* Coordinates are packed into 32 bits (e.g., 12 bits for X, 10 bits for Y, 10 bits for Z), supporting volumes up to $4096 \times 1024 \times 1024$ [S1]. This requires unpacking in the vertex program using bit shifts or emulation via multiplication by a fraction and the floor function [S1].
2.  **Histogram:** An array of structures, one per unique intensity level [S1].
    *   *Fields:* `begin_offset` (index in voxel buffer) and `end_offset` (index in voxel buffer) [S1].
    *   *Purpose:* Acts as a run-length encoding, defining the span of voxels sharing a specific intensity [S1].

**Steps:**
1.  **Sort:** Sort all voxels in the volume by their intensity value [S1].
2.  **Build Histogram:** Iterate through the sorted voxels and record the starting and ending indices of each contiguous block of identical intensity in the buffer [S1].
3.  **Pack Coordinates:** Store the x, y, z coordinates of each voxel into the buffer [S1]. If using 6-byte storage, the intensity is removed from the vertex attributes [S1]. If using 4-byte storage, coordinates are packed and unpacked in the vertex program [S1].
4.  **Render:** For each intensity level, retrieve `begin_offset` and `end_offset` from the histogram, set a shader constant or texture coordinate to the intensity value, and invoke `glDrawArrays` once to render the span [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_intensity_storage(volume_data, resolution):
    """
    volume_data: 3D array of intensity values.
    resolution: (X, Y, Z) dimensions.
    Returns: voxel_buffer (list of (x, y, z) tuples), histogram (dict mapping intensity to (start, end))
    """
    # 1. Flatten and collect voxels as (intensity, x, y, z)
    voxels = []
    for z in range(resolution[2]):
        for y in range(resolution[1]):
            for x in range(resolution[0]):
                voxels.append((volume_data[z, y, x], x, y, z))
    
    # 2. Sort by intensity
    voxels.sort(key=lambda v: v[0])
    
    # 3. Build histogram and optimized coordinate buffer
    histogram = {}
    voxel_buffer = []
    current_intensity = None
    start_idx = 0
    
    for i, (intensity, x, y, z) in enumerate(voxels):
        if intensity != current_intensity:
            if current_intensity is not None:
                histogram[current_intensity] = (start_idx, i)
            current_intensity = intensity
            start_idx = i
        # Store only coordinates for 6-byte optimized storage
        voxel_buffer.append((x, y, z))
        
    # Finalize last span
    if current_intensity is not None:
        histogram[current_intensity] = (start_idx, len(voxels))
        
    return voxel_buffer, histogram

def render_intensity_voxels(histogram, voxel_buffer):
    """
    Simulates the rendering loop per intensity span.
    """
    for intensity, (start, end) in histogram.items():
        # Set shader constant to the intensity for this span
        shader_constants['current_intensity'] = intensity
        
        # Number of voxels in this span
        count = end - start
        
        # Submit to pipeline (equivalent to glDrawArrays)
        draw_primitives(primitive_type='POINTS', start_index=start, count=count)

def draw_primitives(primitive_type, start_index, count):
    """
    Standard operation simulating a GPU draw call.
    """
    # In practice: glDrawArrays(primitive_type, start_index, count)
    pass
```

**Parameters:**
*   **Bit Depth:** Typically 16-bit shorts for coordinates, with 12/10/10 bit packing as an alternative for 4-byte storage [S1].
*   **Intensity Resolution:** Determined by the number of unique values in the histogram [S1].

**Edge Cases and Pitfalls:**
*   **Redundancy:** Storing intensity with each voxel is redundant because the histogram acts as run-length-encoding, so omitting it saves 2 bytes per voxel [S1].
*   **Memory Limits:** Theoretical volume support reaches $65536^3$ with 6-byte storage, but practical limits are lower due to hardware memory constraints [S1].
*   **Bitwise Operations:** On GPUs lacking native bit shift/logical operators, coordinate unpacking from packed formats must be emulated using multiplication by fractions and the floor function, which may impact performance [S1].

## Subtopics

### LiteVoxel
An extension to SVRaster that introduces three modules to counteract low-frequency underfitting and memory overgrowth [S2, unofficial]. It uses an Inverse-Sobel Reweighted Loss to shift gradient focus to flat regions after geometric stabilization, Depth-Quantile Pruning with EMA-hysteresis guards to prevent VRAM inflation near silhouettes, and Priority-Driven Subdivision restricted by ray-footprint eligibility [S2, unofficial]. LiteVoxel matches SVRaster’s perceptual quality at 40–60% of peak VRAM, reducing peak VRAM by up to 60% (e.g., from 12 GB to 7.9 GB) with less than 1% loss in PSNR, SSIM, or LPIPS [S2, unofficial].

### Object-Voxel Set Storage for Time-Varying Data
A method for rendering animated volume data by segmenting the volume into objects and storing only voxels belonging to objects of interest in RAM [S3, unofficial]. It keeps multiple time-steps in memory simultaneously, allowing animated data (e.g., 12 animation phases for heart data) to be rendered identically to static data, avoiding memory bandwidth limitations of hardware renderers that reload data for each time-step [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Sorting voxels by intensity is $O(N \log N)$ where $N$ is the number of voxels [G]. Histogram construction is $O(N)$ [G]. Rendering time scales linearly with the number of unique intensity levels due to draw call overhead [S1].
*   **Memory Complexity:** $O(N)$ for voxel storage [G]. Optimized to 6 bytes per voxel, down from 8 bytes [S1].
*   **Performance:** Provides a major speedup for large volumes by reducing memory bandwidth requirements and leveraging GPU vertex buffers [S1].
*   **Strengths:** Simple implementation, efficient for data with many repeated intensity values, and reduces redundancy [S1].
*   **Limitations:** Requires sorting, which is costly for dynamic data [G]. Limited by the number of unique intensity levels causing draw call overhead [S1]. Practical volume size is limited by hardware memory, not theoretical address space [S1].

## Background
The method was described in the context of multiresolution MIP rendering of large volumetric data accelerated on graphics hardware [S1]. It leverages the `ARB_vertex_buffer_object` extension to cache data in high-performance graphics memory, enabling efficient data transfers for GL-drawing commands [S1]. The approach is particularly relevant for static or slowly changing volumetric data where intensity-based grouping is effective [G].

## Sources
[S1] Multiresolution MIP Rendering of Large Volumetric Data Accelerated on Graphics Hardware, https://doi.org/10.2312/vissym/eurovis07/243-250, fetched 2026-10-26
[S2] Sparse Voxel Rasterization (SVRaster), https://www.emergentmind.com/topics/sparse-voxel-rasterization-svraster, fetched 2026-10-27 (unofficial)
[S3] Distributed Software-Based Volume Visualization in a Virtual Environment, https://doi.org/10.2312/egve/ipt_egve2005/129-140, fetched 2026-10-26 (unofficial)
