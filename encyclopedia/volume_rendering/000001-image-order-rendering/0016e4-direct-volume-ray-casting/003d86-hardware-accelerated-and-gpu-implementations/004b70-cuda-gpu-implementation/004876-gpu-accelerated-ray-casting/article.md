# GPU-accelerated ray casting

Also called: CUDA Grid/Block Raycasting, CUDA Raycasting Kernel, GPU raycasting algorithm, Many-Core GPU Implementation, Standard image order approach

## Overview
GPU-accelerated ray casting is a direct volume rendering method that offloads ray traversal and compositing to a graphics processing unit (GPU) to achieve real-time frame rates [S1][S3, unofficial]. It differs from CPU-based approaches by leveraging massive parallelism, where each thread handles a single pixel, making it an embarrassingly parallel standard image-order approach [S3, unofficial]. It builds on [CUDA GPU Implementation](13016-cuda-gpu-implementation.md) by explicitly mapping the image grid to CUDA thread blocks and utilizing hardware features like context switching to hide memory latency [S2, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A 3D volumetric data set $V$, a camera view (eye point, direction), and a transfer function mapping scalar values to color and opacity.
- Output: A 2D image of resolution $W \times H$.

**Data Structures:**
- **Thread Block Layout:** The image is treated as a 2D CUDA grid divided into thread blocks [S2, unofficial]. An $M \times N$ thread block corresponds to an $M \times N$ pixel image tile [S2, unofficial].
- **Memory Layout:** The source volume $S$ can be stored in array-order or Z-order [S2, unofficial].
    - Array-order: $S[i, j, k]$ is at memory location $i + j \cdot xsize + k \cdot xsize \cdot ysize$.
    - Z-order: A 1D layout of a $2^n$ tree where nearby index space points are nearby in physical memory, computed via the formulation by Pascucci and Frank [S2, unofficial].

**Steps:**
1. **Ray-Bounding Box Intersection:** Calculate the entry ($p_{in}$) and exit ($p_{out}$) points of the ray intersecting the volume bounding box. This is achieved by rendering the bounding box twice: once culling front faces and once culling back faces [S1].
2. **Thread Assignment:** The CUDA runtime dynamically schedules thread blocks, assigning up to eight blocks at a time to each of the GPU’s multiprocessors [S2, unofficial].
3. **Ray Marching and Compositing:** Each individual CUDA thread performs raycasting for a single pixel. The thread traverses the ray from $p_{in}$ to $p_{out}$, sampling the volume and compositing the results [S2, unofficial].
4. **Resampling:** Perform tri-linear interpolation to resample the volume data at the ray positions [S1].
5. **Shading:** Perform resampling and compositing on the shader to reduce running time [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Kernel entry point for a single thread
def gpu_ray_cast_kernel(thread_id, block_id, grid_dims, volume_texture, transfer_function):
    # 1. Determine pixel coordinates from thread/block indices
    # Assuming a 2D grid of blocks, each block is M x N threads
    # This maps directly to image tiles
    pixel_x = block_id.x * block_width + thread_id.x
    pixel_y = block_id.y * block_height + thread_id.y
    
    if pixel_x >= image_width or pixel_y >= image_height:
        return

    # 2. Calculate ray origin and direction for this pixel
    ray_origin = get_eye_point()
    ray_direction = get_ray_direction(pixel_x, pixel_y, camera_matrix)

    # 3. Calculate intersection with bounding box
    # Render bounding box twice (front/back culling) to get pin and pout
    # [S1]
    p_in, p_out = calculate_ray_bbox_intersection(ray_origin, ray_direction, bounding_box)

    # 4. Initialize compositing variables
    accumulated_color = (0.0, 0.0, 0.0, 0.0)
    t = 0.0
    step_size = calculate_step_size(p_in, p_out, resolution)

    # 5. Ray march from p_in to p_out
    while t < distance(p_in, p_out):
        current_pos = ray_origin + ray_direction * t
        
        # 6. Sample volume using tri-linear interpolation
        # [S1]
        scalar_value = sample_volume(volume_texture, current_pos)
        
        # 7. Map scalar to color and opacity using transfer function
        color, opacity = apply_transfer_function(scalar_value, transfer_function)
        
        # 8. Composite (front-to-back)
        # alpha_out = alpha_in + (1 - alpha_in) * alpha_sample
        # color_out = color_in + (1 - alpha_in) * alpha_sample * color_sample
        accumulated_color = composite(accumulated_color, color, opacity)
        
        t += step_size

    # 9. Write result to image buffer
    write_pixel(pixel_x, pixel_y, accumulated_color)

def calculate_ray_bbox_intersection(ray_origin, ray_direction, bounding_box):
    # [S1] Implementation detail: Render bounding box twice
    # Once culling front faces to find the first hit (p_in)
    # Once culling back faces to find the last hit (p_out)
    # This avoids analytical intersection code in the kernel
    p_in = render_bbox_front_culled(ray_origin, ray_direction, bounding_box)
    p_out = render_bbox_back_culled(ray_origin, ray_direction, bounding_box)
    return p_in, p_out
```

**Parameters:**
- **CUDA Version:** 3.2 compiler and runtime [S2, unofficial].
- **NVIDIA Driver:** Version 270.41.19 [S2, unofficial].
- **Compute Capability:** Targeted at 2.0 (Fermi-series) for additional optimizations [S2, unofficial].
- **Shared Memory:** Configured to be used as L1 cache for global memory requests [S2, unofficial].
- **ECC Support:** Enabled [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Memory Latency:** The GPU uses hardware context switches between blocks within a multiprocessor to hide memory latency [S2, unofficial].
- **Data Size Scaling:** While computational complexity for interpolation and compositing is approximately constant for the same viewpoint, processing time shows a slowly wavelike rising trend as data set size increases due to increased RAM/graphics card memory addressing time and varying computing resources [S1].

## Subtopics

### [GPU Ray Casting Pipeline](14281-gpu-ray-casting-pipeline.md)
The foundational hardware-accelerated method defining the data flow between CPU-side texture preparation and GPU-side ray execution. Chosen when explicit pipeline control is needed for real-time displays like HTC Vive.

### [GPU Direct Volume Rendering with Anisotropic Shading](16499-gpu-direct-volume-rendering-with-anisotropic-shading.md)
Enhances visual quality by applying anisotropic shading models (Ward or Lafortune BRDFs) using precomputed covariance matrices. Chosen when surface orientation and anisotropy are critical.

### [GPU-Based Ray-Casting for AMR](17572-gpu-based-ray-casting-for-amr.md)
Supports Adaptive Mesh Refinement (AMR) data with wavelength-dependent absorption. Chosen for complex light models and higher quality compared to slicing-based approaches.

## Operating characteristics
- **Time Complexity:** The computational complexity for tri-linear interpolation and compositing is approximately constant regardless of data set size increase, as the number of resampling points does not grow with data set size for the same viewpoint [S1].
- **Performance:** In experiments with a $101 \times 101 \times 250$ seismic data volume at 10 viewpoints (50 trials each, removing max/min), the GPU average time was 14.9947 ms (range [14.1886, 15.7046]), compared to a CPU average of 9486.24 ms (range [9250.04, 9737.42]) [S1].
- **Scaling:** When the data set size increased ten times (increment $10 \times 28 \times 25$) at the same viewpoint, both GPU and CPU processing times showed a slowly wavelike rising trend due to increased memory addressing time and varying computing resources, despite constant computational complexity for interpolation and compositing [S1].
- **Strengths:** Significantly faster than CPU algorithms due to lower computational complexity for intersection calculation, resampling, and compositing [S1]. It is an embarrassingly parallel standard image-order approach that supports many performance and quality enhancements [S3, unofficial].
- **Limitations:** Processing time can fluctuate based on the number of resampling points required for different viewpoints and intersections [S1].

## Background
GPU-accelerated ray casting was introduced by Röttger et al. in 2003 and Krüger and Westermann in 2003 [S3, unofficial]. It emerged as a standard image-order approach to overcome the limitations of earlier texture slicing methods, which suffered from visual artifacts and less flexibility [S3, unofficial]. It is widely used in interactive visualization and real-time applications requiring high frame rates [S1][S3, unofficial].

## Sources
[S1] Volume Rendering Method for Spatial Seismic Data Based on GPU, https://doi.org/10.2991/isccca.2013.95, fetched 2026-10-26
[S2] Multi-core and Many-core Shared-memory Parallel Raycasting Volume Rendering Optimization and Tuning, https://digital.library.unt.edu/ark:/67531/metadc829701/?q=empty space skipping volume rendering, fetched 2026-10-28 (unofficial)
[S3] GPU-Based Large-Scale Scientific Visualization, https://johanna-b.github.io/LargeSciVis2018/slides/LargeScaleVisualization_Architectures.pdf, fetched 2026-10-29 (unofficial)
