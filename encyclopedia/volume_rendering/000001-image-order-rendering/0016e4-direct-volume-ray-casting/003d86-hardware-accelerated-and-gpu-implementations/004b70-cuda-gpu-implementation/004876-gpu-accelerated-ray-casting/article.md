# GPU-accelerated ray casting

Also called: GPU raycasting algorithm, Standard image order approach

## Overview
GPU-accelerated ray casting is a direct volume rendering method that offloads the per-ray sampling and integration loop to the graphics processor, enabling real-time interaction with volumetric datasets [S1]. It differs from CPU-based ray casting by leveraging the GPU's parallel architecture to significantly reduce computational complexity for intersection calculation, resampling, and compositing [S1]. As a standard image-order approach, it is embarrassingly parallel and supports numerous performance and quality enhancements [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volumetric dataset (e.g., seismic, CT, MRI), a camera viewpoint (eye point and direction), and a transfer function mapping scalar values to color and opacity.
*   **Output:** A 2D image (framebuffer) with color and opacity values for each pixel.

**Data Structures:**
*   **Volumetric Data:** Stored in GPU memory, typically as a 3D texture object to facilitate hardware-accelerated tri-linear interpolation.
*   **Bounding Box:** The axis-aligned bounding box (AABB) of the volume, used to determine ray entry and exit points.

**Algorithm Steps:**
1.  **Ray-Bounding Box Intersection Calculation:**
    For each pixel, calculate the intersection points $p_{in}$ (entry) and $p_{out}$ (exit) of the viewing ray with the volume's bounding box. This is achieved by rendering the bounding box geometry twice:
    *   First pass: Cull front faces to find the entry point $p_{in}$.
    *   Second pass: Cull back faces to find the exit point $p_{out}$ [S1].
    *   *Note:* This hardware-accelerated geometric intersection replaces the computationally expensive analytical ray-box intersection often used in CPU implementations [S1].

2.  **Ray Marching and Sampling:**
    Iterate along the ray segment from $p_{in}$ to $p_{out}$. At each step, sample the 3D texture using tri-linear interpolation to obtain the scalar value $s$ at the current position [S1].

3.  **Compositing:**
    Apply the transfer function to map the scalar value $s$ to color $C$ and opacity $\alpha$. Perform front-to-back compositing to accumulate the final color and opacity for the pixel [S1].
    $$ C_{out} = C_{in} + C_{sample} \cdot \alpha_{sample} \cdot (1 - \alpha_{in}) $$
    $$ \alpha_{out} = \alpha_{in} + \alpha_{sample} \cdot (1 - \alpha_{in}) $$

4.  **Shader Execution:**
    Perform resampling and compositing within the GPU shader to reduce running time [S1].

**Parameters:**
*   **Step Size:** Determined by the desired sampling density along the ray. Not explicitly defined in sources, but implied by the bounding box intersection and volume dimensions [S1].
*   **Transfer Function:** Maps scalar values to color and opacity. Not explicitly defined in sources [S1].

**Edge Cases and Pitfalls:**
*   **Memory Addressing:** As dataset size increases, processing time shows a slowly wavelike rising trend due to increased RAM/graphics card memory addressing time and varying computing resources, despite constant computational complexity for interpolation and compositing [S1].
*   **Viewpoint Dependence:** At different viewpoints, the positions and number of resampling points vary due to different intersections, causing fluctuation features in response time [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for GPU-accelerated ray casting
# Assumes 3D texture 'volumeTex' and transfer function 'transferFunction' are available

def ray_casting_pixel(ray_origin, ray_direction, bounding_box):
    # Step 1: Calculate ray-bounding box intersections
    # Render bounding box twice: once culling front faces, once culling back faces
    p_in = get_entry_intersection(ray_origin, ray_direction, bounding_box, cull_front=True)
    p_out = get_exit_intersection(ray_origin, ray_direction, bounding_box, cull_back=True)
    
    # If no intersection, return background color
    if p_in is None or p_out is None:
        return background_color
    
    # Step 2: Ray Marching and Compositing
    accumulated_color = (0.0, 0.0, 0.0, 0.0)
    current_pos = p_in
    step_size = calculate_step_size(p_in, p_out) # Implementation detail [G]
    
    while current_pos < p_out:
        # Sample volume using tri-linear interpolation
        scalar_value = sample_3d_texture(volumeTex, current_pos)
        
        # Map scalar to color and opacity
        color, opacity = transferFunction(scalar_value)
        
        # Front-to-back compositing
        alpha_in = accumulated_color[3]
        color_in = accumulated_color[0:3]
        
        # Update accumulated color and opacity
        # C_out = C_in + C_sample * alpha_sample * (1 - alpha_in)
        # alpha_out = alpha_in + alpha_sample * (1 - alpha_in)
        new_color = color_in + color * opacity * (1.0 - alpha_in)
        new_alpha = alpha_in + opacity * (1.0 - alpha_in)
        
        accumulated_color = (new_color[0], new_color[1], new_color[2], new_alpha)
        
        # Advance along ray
        current_pos = current_pos + ray_direction * step_size
        
    return accumulated_color
```

## Subtopics

### [GPU Ray Casting Pipeline](14281-gpu-ray-casting-pipeline.md)
The foundational hardware-accelerated method for direct volume rendering, mapping volumetric data to a 3D texture on the GPU for front-to-back compositing. It differs from CPU-based ray casting by offloading the per-ray sampling and integration loop to the graphics processor, enabling real-time interaction with CT and MRI datasets.

## Operating characteristics
*   **Time Complexity:** Computational complexity for tri-linear interpolation and compositing is approximately constant regardless of dataset size increase, as the number of resampling points in the data volume does not grow with dataset size for the same viewpoint [S1].
*   **Memory Complexity:** Processing time shows a slowly wavelike rising trend as dataset size increases due to increased RAM/graphics card memory addressing time and varying computing resources [S1].
*   **Measured Performance:** In experiments with a 101×101×250 seismic data volume at 10 viewpoints (50 trials each, removing max/min), GPU average time was 14.9947 ms (range [14.1886, 15.7046]), compared to CPU average 9486.24 ms (range [9250.04, 9737.42]) [S1].
*   **Strengths:** Significantly faster than CPU algorithms due to lower computational complexity for intersection calculation, resampling, and compositing [S1]. Supports real-time interaction [S1].
*   **Limitations:** Processing time fluctuates with viewpoint due to varying intersection points and resampling counts [S1]. Performance degrades slightly with increasing dataset size due to memory addressing overhead [S1].

## Background
GPU-accelerated ray casting was introduced by Röttger et al. in 2003 and Krüger and Westermann in 2003 [S2, unofficial]. It emerged as a standard image-order approach that is embarrassingly parallel and supports many performance and quality enhancements [S2, unofficial]. It is used for interactive visualization of large-scale scientific data, such as seismic and medical imaging datasets [S1].

## Sources
[S1] Volume Rendering Method for Spatial Seismic Data Based on GPU, https://doi.org/10.2991/isccca.2013.95, fetched 2026-10-26
[S2] GPU-Based Large-Scale Scientific Visualization, https://johanna-b.github.io/LargeSciVis2018/slides/LargeScaleVisualization_Architectures.pdf, fetched 2026-10-29 (unofficial)
