# Texture-Mapped Parallel-Beam Backprojection

Also called: 2D image reconstruction using texture mapping, Parallel-Beam Texture Reconstruction, RM-based backprojection

## Overview
This method performs filtered backprojection of 2D tomographic data (sinograms) onto a 2D image grid by leveraging graphics hardware texture mapping and frame buffer accumulation, thereby avoiding CPU-based floating-point arithmetic [S1]. It distinguishes itself from CPU-based alternatives by generating the reconstructed image directly in frame-buffer memory, allowing intermediate results to be visible on the screen projection-by-projection without additional display operations [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A filtered sinogram $p(s, \theta)$, where $s$ is the detector coordinate and $\theta$ is the projection angle. The sinogram consists of 512 samples per row and 804 rows (angles) representing 180 degrees of coverage [S1].
*   **Output:** A reconstructed 2D image grid (e.g., 512x512 pixels) stored in the frame buffer [S1].

**Data Structures**
*   **Texture:** All projection functions $p$ at a specific slice $z$ are treated as a single two-dimensional texture [S1].
*   **Frame Buffer:** Accumulates the backprojected values. The implementation uses 12-bit precision to match the Raster Manager frame memory configuration [S1].

**Parameters**
*   **Bit Depth:** 12 bits [S1].
*   **Image Resolution:** 512x512 pixels [S1].
*   **Projection Angles:** 804 angles covering 180 degrees [S1].

**Algorithm Steps**
1.  **Filtering:** Filter the sinogram data by row on the main CPU [S1].
2.  **Initialization:** Initialize the frame buffer to zero [G].
3.  **Loop over Angles:** For each angle $\theta_n$:
    a.  **Rotate:** Rotate the entire filtered projection data by $\theta_n$ [S1].
    b.  **Setup Geometry:** Setup depth cueing for the parallel beam geometry [S1].
    c.  **Render:** Render a texture-mapped circle (representing the parallel projection) using the rotated texture $p(s_n, \theta_n)$ [S1].
    d.  **Accumulate:** Accumulate the rendered result into the frame buffer [S1].
4.  **Finalization:** Scale and remap the accumulated frame buffer to produce the final reconstructed image [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def texture_mapped_parallel_beam_backprojection(sinogram, num_angles, image_size):
    """
    Performs parallel-beam backprojection using hardware texture mapping.
    
    Args:
        sinogram: 2D array of shape (num_angles, num_samples).
        num_angles: Number of projection angles.
        image_size: Width/Height of the output image (assumed square).
        
    Returns:
        reconstructed_image: 2D array representing the reconstructed slice.
    """
    # Step 1: Filter sinogram by row on CPU
    # Note: The specific filter kernel (e.g., Ram-Lak) is not detailed in sources,
    # but the source states filtering is done by row on the main CPU [S1].
    filtered_sinogram = filter_sinogram_by_row(sinogram)
    
    # Initialize frame buffer (accumulation buffer)
    # Using 12-bit precision as per hardware constraints [S1]
    frame_buffer = initialize_frame_buffer(image_size, bit_depth=12)
    
    # Step 3: Loop over all angles
    for n in range(num_angles):
        theta = n * (180.0 / num_angles)  # Angle in degrees
        
        # Step 3a: Rotate the entire projection by theta_n
        # The source describes rotating the projection data [S1].
        # In a texture-mapped context, this is often achieved by rotating the texture coordinates
        # or the geometry of the primitive being rendered.
        rotated_projection = rotate_projection(filtered_sinogram[n], theta)
        
        # Step 3b: Setup depth cueing for parallel beam geometry
        # This handles the geometry setup for the hardware renderer [S1].
        setup_depth_cueing(geometry_type="parallel_beam")
        
        # Step 3c: Render a texture-mapped circle with texture p(s_n, theta_n)
        # The "circle" represents the domain of integration for the parallel beam backprojection.
        # The texture contains the filtered projection values.
        render_texture_mapped_circle(
            texture=rotated_projection,
            geometry="circle",
            accumulation_mode="add"
        )
        
        # Step 3d: Accumulate into the framebuffer
        # The hardware accumulation happens automatically during rendering [S1].
        accumulate_to_framebuffer()
        
    # Step 4: Scale and remap the accumulation
    # Convert the accumulated 12-bit values to the final output format [S1].
    reconstructed_image = scale_and_remap(frame_buffer)
    
    return reconstructed_image

def filter_sinogram_by_row(sinogram):
    """
    Applies a 1D filter to each row of the sinogram.
    This is performed on the CPU [S1].
    """
    # Implementation detail: Standard tomographic filtering (e.g., convolution with ramp filter)
    # is applied along the detector axis (rows).
    filtered = apply_1d_filter_along_axis(sinogram, axis=1)
    return filtered

def initialize_frame_buffer(size, bit_depth):
    """
    Initializes the accumulation buffer.
    """
    return np.zeros((size, size), dtype=np.uint16) # 12-bit fits in 16-bit container

def rotate_projection(projection_data, angle_degrees):
    """
    Rotates the projection data.
    In the context of texture mapping, this may involve rotating texture coordinates
    rather than the data array itself, but the logical operation is a rotation [S1].
    """
    # Placeholder for rotation logic. 
    # In hardware acceleration, this is often handled by the texture coordinate transformation matrix.
    return projection_data # Conceptual rotation

def setup_depth_cueing(geometry_type):
    """
    Configures the graphics hardware for the specific beam geometry.
    For parallel beam, this sets up the appropriate depth/geometry parameters [S1].
    """
    pass # Hardware-specific setup

def render_texture_mapped_circle(texture, geometry, accumulation_mode):
    """
    Renders the texture-mapped primitive.
    The texture contains the projection values. The geometry (circle) defines the backprojection path.
    """
    pass # Hardware-specific rendering call

def accumulate_to_framebuffer():
    """
    Ensures the rendered output is added to the existing frame buffer content.
    """
    pass # Hardware-specific accumulation

def scale_and_remap(frame_buffer):
    """
    Scales the accumulated 12-bit values to the final output range.
    """
    # Normalize and convert to standard image format
    return frame_buffer / (2**12 - 1)
```

**Edge Cases and Pitfalls**
*   **Precision:** The backprojection is implemented in 12-bit precision to match the Raster Manager frame memory configuration. This reduced precision causes no significant error in the reconstruction [S1].
*   **Filtering Location:** The filtering part of the algorithm is currently done on the main CPU, not the hardware [S1].

## Operating characteristics
*   **Performance:** On a four Raster Manager (RM) Reality Engine2 Onyx with a 150 MHz R4400 CPU, the RM implementation reconstructs a 512x512 image in 2.1 seconds [S1].
*   **Comparison:** This is approximately a 600x speedup compared to a CPU-based floating-point algorithm on an SGI Indigo2, which takes 20+ minutes [S1].
*   **Scalability:** The reconstruction algorithm shows linear speed improvement with the number of Raster Managers; removing half the RMs doubles the processing time [S1].
*   **Strengths:** Direct generation of the reconstructed 2D image in frame-buffer memory allows intermediate results to be visible on the screen projection-by-projection without an additional display operation [S1].
*   **Limitations:** The filtering step is performed on the CPU, which may become a bottleneck for very large datasets or if higher-performance filtering is required [S1].

## Background
This technique was developed for tomographic image reconstruction using texture-mapping hardware, specifically targeting Silicon Graphics Reality Engine2 systems with Raster Managers [S1]. It leverages the parallel processing capabilities of graphics hardware to accelerate the computationally intensive backprojection step of filtered backprojection algorithms [S1].

## Sources
[S1] Tomographic image reconstruction and rendering with texture-mapping hardware, https://doi.org/10.1117/12.179259, fetched 2026-10-26
