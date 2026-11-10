# GPU-accelerated ray casting

Also called: GPU raycasting algorithm

## Overview
GPU-accelerated ray casting is a direct volume rendering technique that offloads the computationally intensive tasks of ray-bounding box intersection, resampling, and compositing to the graphics processing unit (GPU) to achieve real-time interactive performance [S1]. It distinguishes itself from CPU-based implementations by leveraging parallel shader execution, which significantly reduces running time despite the underlying computational complexity of interpolation remaining constant for a given viewpoint [S1].

## How it works
The method performs resampling and compositing within the GPU shader to minimize latency [S1]. The core pipeline involves determining the entry and exit points of rays through the volume data and then integrating the optical properties along those rays.

**Inputs and Outputs**
*   **Inputs:** A 3D volume dataset (e.g., seismic data), a transfer function mapping scalar values to color and opacity, camera parameters (eye point, view direction), and a bounding box defining the volume extent.
*   **Outputs:** A 2D image (gray mode or pseudo-color mode) representing the volume rendering.

**Data Structures**
*   **Volume Data:** Stored in GPU memory (RAM/graphics card memory). The specific bit-packing is not detailed in the sources, but the data is accessed via tri-linear interpolation [S1].
*   **Bounding Box:** A 3D axis-aligned box used to cull rays. The sources do not specify the exact memory layout of the bounding box representation, only that it is rendered to determine intersections [S1].

**Algorithm Steps**
1.  **Calculate Ray-Bounding Box Intersections:** Determine the entry point ($p_{in}$) and exit point ($p_{out}$) for each ray cast from the camera through each pixel. This is achieved by rendering the bounding box twice: once with front-face culling to find the nearest intersection ($p_{in}$) and once with back-face culling to find the farthest intersection ($p_{out}$) [S1].
2.  **Resampling:** For each ray segment between $p_{in}$ and $p_{out}$, sample the volume data at regular intervals. The number of resampling points depends on the viewpoint and the specific intersections, leading to fluctuating computational complexity per frame [S1].
3.  **Compositing:** Perform front-to-back or back-to-front compositing using the sampled values and the transfer function to generate the final pixel color and opacity [S1].

**Formulas**
*   **Intersection Calculation:** The sources describe the geometric method (rendering bounding box with face culling) but do not provide the explicit mathematical formula for calculating $p_{in}$ and $p_{out}$ [S1].
*   **Tri-linear Interpolation:** Used during resampling. The specific coefficients or formula are not provided, but the computational complexity is noted as approximately constant for a fixed viewpoint regardless of total dataset size [S1].

**Parameters**
*   **Viewpoint:** The position and orientation of the camera. Changing viewpoints alters the number of resampling points, causing fluctuations in response time [S1].
*   **Data Set Size:** The dimensions of the volume. The sources note that increasing the dataset size (e.g., by increments of $10 \times 28 \times 25$) does not significantly increase the number of resampling points for the same viewpoint, keeping interpolation complexity stable [S1].

**Edge Cases and Pitfalls**
*   **Memory Addressing Overhead:** As the dataset size increases, the occupied memory in RAM and graphics card memory grows, leading to increased addressing time. This causes a slowly wavelike rising trend in processing time, even when computational complexity is constant [S1].
*   **Fluctuating Response Time:** Because the number of resampling points varies with different viewpoints and intersections, the response time exhibits fluctuation features [S1].
*   **Precision:** Not given in the sources.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# GPU Shader Execution per Pixel
def render_pixel(camera_eye, camera_direction, volume_texture, transfer_function, bounding_box):
    # Step 1: Calculate Ray-Bounding Box Intersections
    # Render bounding box twice: once culling front faces, once culling back faces
    # to obtain p_in and p_out.
    # Note: The specific geometric intersection formula is not provided in sources.
    p_in = get_front_face_intersection(camera_eye, camera_direction, bounding_box)
    p_out = get_back_face_intersection(camera_eye, camera_direction, bounding_box)
    
    # Initialize compositing variables
    accumulated_color = (0.0, 0.0, 0.0, 0.0)
    accumulated_opacity = 0.0
    
    # Step 2 & 3: Resampling and Compositing Loop
    # Determine step size based on resolution and distance
    step_size = calculate_step_size(p_in, p_out)
    current_pos = p_in
    
    while current_pos < p_out:
        # Sample volume data using tri-linear interpolation
        # The number of points varies by viewpoint, causing complexity fluctuations
        value = tri_linear_sample(volume_texture, current_pos)
        
        # Map value to color and opacity using transfer function
        color, opacity = transfer_function(value)
        
        # Front-to-back compositing
        # C_out = C_in + (1 - A_in) * C_sample * A_sample
        # A_out = A_in + (1 - A_in) * A_sample
        accumulated_color = accumulated_color + (1.0 - accumulated_opacity) * color * opacity
        accumulated_opacity = accumulated_opacity + (1.0 - accumulated_opacity) * opacity
        
        # Advance ray
        current_pos = current_pos + step_size * camera_direction
        
        # Early exit if fully opaque
        if accumulated_opacity >= 1.0:
            break
            
    return accumulated_color
```

## Subtopics

### Slab-based Raycasting
A sibling method that processes the volume in slabs. It differs in how it manages memory access patterns and traversal, potentially offering different trade-offs in cache coherence compared to the direct ray casting approach described here. [Slab-based Raycasting](11383-slab-based-raycasting.md)

## Operating characteristics
*   **Time Complexity:** The computational complexity for tri-linear interpolation and compositing is approximately constant regardless of the data set size increase, provided the viewpoint remains the same, because the number of resampling points does not grow with the dataset size [S1]. However, overall processing time shows a slowly wavelike rising trend as dataset size increases due to increased RAM/graphics card memory addressing time and varying computing resources [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** In experiments with a $101 \times 101 \times 250$ seismic data volume at 10 viewpoints (50 trials each, removing max/min):
    *   GPU average time: 14.9947 ms (range [14.1886, 15.7046]) [S1].
    *   CPU average time: 9486.24 ms (range [9250.04, 9737.42]) [S1].
*   **Scaling:** Increasing the dataset size ten times (increment $10 \times 28 \times 25$) at the same viewpoint resulted in a slowly wavelike rise in processing time for both GPU and CPU, despite the constant computational complexity of interpolation and compositing [S1].
*   **Strengths:** Significantly faster than CPU algorithms due to lower computational complexity for intersection calculation, resampling, and compositing on the GPU [S1]. Supports real-time interactive adjustment of transfer functions and eye points [S1].
*   **Limitations:** Processing time fluctuates with viewpoints due to discrepant computational complexity of resampling and compositing [S1]. Performance degrades slightly with very large datasets due to memory addressing overhead [S1].
*   **When to use:** When real-time interactive visualization of large volume datasets (e.g., seismic data) is required.
*   **When not:** When the hardware lacks sufficient GPU memory or compute capability, or when the viewpoint is static and the overhead of GPU setup outweighs the benefits (though the sources do not explicitly state this, the massive speedup suggests GPU is preferred for interactivity).

## Background
The method was presented in the proceedings of the 2nd International Symposium on Computer, Communication, Control and Automation (ISCCCA-13) in 2013 [S1]. It was developed to address the need for real-time interactive visualization of spatial seismic data, where distinct layer surface features need to be intelligible [S1]. The work acknowledges contributions from colleagues at the College of Information Engineering, China University of Petroleum, and A/Prof. LU You [S1].

## Sources
[S1] Volume Rendering Method for Spatial Seismic Data Based on GPU, https://doi.org/10.2991/isccca.2013.95, fetched 2026-10-26
