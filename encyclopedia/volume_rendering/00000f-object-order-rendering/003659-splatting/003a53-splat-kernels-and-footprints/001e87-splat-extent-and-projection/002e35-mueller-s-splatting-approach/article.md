# Mueller's Splatting Approach

Also called: Footprint-based Splatting, Object Order Algorithm, Splatting, Splatting Algorithms

## Overview
Mueller's Splatting Approach is an object-order volume rendering method that treats a gray-scale volume as a 3D discrete density function, calculating the influence of each voxel on multiple output pixels to accelerate Direct Volume Rendering (DVR) [S1, unofficial][S2][S3, S4, unofficial]. It distinguishes itself from ray-casting by exploiting cache coherency through sequential memory traversal and supporting incremental rendering, though it introduces blurring due to spherically symmetric reconstruction kernels [S1, unofficial][S2][S3, S4, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume represented as a discrete density function $f(s)$, where $s=(x_s, y_s, z_s)$ is a voxel position and $f(s)$ is its density [S1, S3, S4, unofficial]. A reconstruction kernel $h$ (e.g., Gaussian, Cone, Sinc) [S1, unofficial][S2][S3, S4, unofficial]. A viewing transformation (orthographic or perspective) [S1, S3, S4, unofficial].
*   **Output:** A 2D image where each pixel $(x,y)$ accumulates contributions from all voxels [S1, S3, S4, unofficial].

**Data Structures:**
*   **Footprint Table:** A precomputed 2D grid storing the integrated kernel weights. The resolution of this table is much higher than the image plane resolution to ensure accuracy [S1, S3, S4, unofficial]. Values outside the table extent are zero and ignored [S1, S3, S4, unofficial].
*   **Sheet Buffers:** In the axis-aligned variant, three stacks of sheet buffers are maintained to handle depth sorting and prevent color bleeding [S2][S4, unofficial]. In the image-aligned variant, buffers are parallel to the image plane [S2][S4, unofficial].

**Numbered Steps:**

1.  **Kernel Definition:** Define the 3D reconstruction kernel $h(x, y, z)$. Common choices include Gaussian, Cone, Sinc, or Bilinear functions [S1, unofficial][S2][S3, S4, unofficial]. The kernel is typically rotationally symmetric [S2][S3, S4, unofficial].

2.  **Footprint Table Generation:** Compute the footprint function $F(x, y)$ by integrating the kernel along the viewing ray direction (axis $u$) [S1, S3, S4, unofficial]:
    $$ F(x, y) = \int_{-\infty}^{\infty} h(x - x_s, y - y_s, u) \, du $$
    where $(x, y)$ is the displacement from the sample's projected center [S1, S3, S4, unofficial]. Evaluate this integral on a high-resolution grid to create the Footprint Table [S1, S3, S4, unofficial]. For orthographic projection, only one generic footprint table is needed per view, as offsets are handled during rendering [S1, S3, S4, unofficial].

3.  **Voxel Processing (Object Order):** For each voxel $s$ in the volume:
    a. Transform the voxel position $s$ to screen space $(x_s, y_s, z_s)$ [S1, S3, S4, unofficial].
    b. Classify the voxel density $f(s)$ into color and opacity using a Transfer Function (TF) [S2][S4, unofficial].
    c. Determine the 2D footprint on the image plane centered at $(x_s, y_s)$ [S1, S3, S4, unofficial].

4.  **Splatting (Projection):** Distribute the voxel's contribution to the affected pixels. The contribution $C(x, y)$ to a pixel $(x, y)$ is:
    $$ C(x, y) = f(s) \cdot F(x - x_s, y - y_s) $$
    where $F$ is sampled from the Footprint Table [S1, S3, S4, unofficial]. The table is centered on the projected location of $s$ [S1, S3, S4, unofficial].

5.  **Compositing:**
    *   **Axis-Aligned (Sheet Buffer):** Voxels are processed in sheets (x-y, x-z, or y-z) parallel to the image plane. Sheets are composited back-to-front using the Porter-Duff over operator (alpha blending) [S1, unofficial][S2][S4, unofficial]. Three stacks of sheets are maintained to handle view changes [S2][S4, unofficial].
    *   **Image-Aligned:** Sheet buffers are aligned parallel to the image plane to reduce popping artifacts [S2][S4, unofficial].
    *   **Incremental:** Rendering can be performed incrementally in front-to-back or back-to-front order [S1, S4, unofficial].

**Parameters:**
*   **Footprint Table Size:** Small tables cause blocky images; large tables smooth details and consume more memory [S1, S3, S4, unofficial].
*   **Sampling Method:** Nearest-neighbor sampling is fast but causes aliasing; bilinear interpolation is smoother but slower [S1, S3, S4, unofficial].
*   **Reconstruction Filter:** Choice of $h$ (Gaussian, Cone, Sinc, Bilinear) affects image quality [S1, S3, S4, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mueller_splatting(volume, transfer_function, kernel, view_transform, image_width, image_height):
    """
    volume: 3D array of densities f(s)
    transfer_function: function mapping density to (color, opacity)
    kernel: function h(x, y, z)
    view_transform: function mapping 3D voxel coord to 2D screen coord (x, y) and depth z
    image_width, image_height: dimensions of the output image
    """
    
    # Step 1: Generate Footprint Table
    # Determine extent of kernel h for integration
    kernel_extent = determine_kernel_extent(kernel)
    
    # High-resolution grid for footprint table
    footprint_resolution = image_width * 4  # Example: 4x image resolution
    footprint_table = zeros((footprint_resolution, footprint_resolution))
    
    # Evaluate integral F(x,y) = integral(h(x-xs, y-ys, u) du)
    # For simplicity, assume orthographic projection along z-axis for table generation
    # In practice, this is done per view or transformed later
    for i in range(-kernel_extent, kernel_extent):
        for j in range(-kernel_extent, kernel_extent):
            # Integrate along the viewing ray (u)
            integral_value = integrate_kernel_along_ray(kernel, i, j)
            footprint_table[i + kernel_extent, j + kernel_extent] = integral_value
            
    # Step 2: Initialize Image Buffer
    image_buffer = zeros((image_height, image_width, 4)) # RGBA
    
    # Step 3: Process Voxels
    # Traverse volume in memory order for cache coherency
    for s in volume.iter_voxels():
        density = volume[s]
        
        # Classify
        color, opacity = transfer_function(density)
        
        # Project to screen
        sx, sy, sz = view_transform(s)
        
        # Determine footprint extent on screen
        # Map kernel extent to screen space
        footprint_radius = map_kernel_extent_to_screen(kernel_extent, sz)
        
        # Iterate over pixels covered by the splat
        for dy in range(-footprint_radius, footprint_radius):
            for dx in range(-footprint_radius, footprint_radius):
                px = int(sx + dx)
                py = int(sy + dy)
                
                if 0 <= px < image_width and 0 <= py < image_height:
                    # Sample footprint table
                    # Normalize dx, dy to table coordinates
                    table_x = int((dx + kernel_extent) * (footprint_resolution / (2 * kernel_extent)))
                    table_y = int((dy + kernel_extent) * (footprint_resolution / (2 * kernel_extent)))
                    
                    # Clamp to table bounds
                    table_x = clamp(table_x, 0, footprint_resolution - 1)
                    table_y = clamp(table_y, 0, footprint_resolution - 1)
                    
                    weight = footprint_table[table_x, table_y]
                    
                    # Contribution
                    contribution = weight * density
                    
                    # Alpha Blend
                    # Pre-multiply color by opacity for standard blending
                    src_color = color * opacity
                    src_alpha = opacity
                    
                    dst_color = image_buffer[py, px, :3]
                    dst_alpha = image_buffer[py, px, 3]
                    
                    # Porter-Duff Over Operator
                    new_alpha = src_alpha + dst_alpha * (1 - src_alpha)
                    if new_alpha > 0:
                        new_color = (src_color + dst_color * dst_alpha * (1 - src_alpha)) / new_alpha
                    else:
                        new_color = src_color
                        
                    image_buffer[py, px, :3] = new_color
                    image_buffer[py, px, 3] = new_alpha

    return image_buffer

def integrate_kernel_along_ray(kernel, dx, dy):
    """
    Computes integral of kernel along u (viewing ray) for displacement dx, dy.
    """
    # Numerical integration
    u_range = determine_u_range(kernel, dx, dy)
    integral = 0
    step = 0.1
    for u in u_range:
        integral += kernel(dx, dy, u) * step
    return integral

def determine_kernel_extent(kernel):
    """
    Determines the radius where kernel value is negligible.
    """
    # For Gaussian with sigma=1, radius 2 is common
    return 2

def map_kernel_extent_to_screen(kernel_extent, depth):
    """
    Maps 3D kernel extent to 2D screen pixels based on depth and projection.
    """
    # Depends on view transform (orthographic vs perspective)
    # Placeholder for specific projection math
    return kernel_extent
```

**Edge Cases and Pitfalls:**
*   **Blurring:** Images are blurred because of the spherically symmetric reconstruction kernel [S1, S3, S4, unofficial].
*   **Color Bleeding:** Basic splatting suffers from color bleeding artifacts [S2][S4, unofficial].
*   **Popping:** Axis-aligned sheet buffers introduce "popping" artifacts when the view switches between stacks [S2][S4, unofficial].
*   **Aliasing:** Splatting is prone to aliasing problems [S2][S4, unofficial].
*   **Performance:** Ideal splatting (evaluating kernel for every pixel) is terribly expensive [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(V \cdot A)$, where $V$ is the number of voxels and $A$ is the average area of the footprint in pixels. Each voxel is touched once, but each pixel is touched often [S4, unofficial].
*   **Memory Complexity:** $O(V)$ for the volume and $O(F)$ for the footprint table, where $F$ is the footprint table size [S1, S3, S4, unofficial].
*   **Performance:** Faster than ray-casting due to cache coherency (sequential memory access) [S1, S3, S4, unofficial]. Supports incremental rendering [S1, S4, unofficial].
*   **Strengths:** Cache-friendly, supports incremental rendering, renders smooth surfaces without staircase artifacts [S1, S3, S4, unofficial].
*   **Limitations:** Blurred images due to spherical kernels, color bleeding, popping artifacts (in axis-aligned variants), aliasing [S1, unofficial][S2][S3, S4, unofficial].
*   **When to Use:** When rendering speed is prioritized over sharp boundaries, and cache coherency is beneficial [S1, S3, S4, unofficial].
*   **When Not to Use:** When sharp object boundaries are required (use ray-casting with trilinear reconstruction) [S1, S3, S4, unofficial].

## Background
Mueller's Splatting Approach is based on the work of Westover (1989, 1990) and was further developed by Mueller (1998, 1999) with image-aligned sheet-based splatting and post-classified splatting [S3, S4, unofficial]. It is an object-order method that treats the volume as a 3D discrete density function [S1, S3, S4, unofficial]. It is widely used in medical image visualization [S2][S4, unofficial].

## Sources
[S1] Interactive Volume-Rendering Techniques for Medical Data Visualization, https://doi.org/10.2312/diss.20018138, fetched 2026-10-26 (unofficial)
[S2] Volume Visualization: A Technical Overview with a Focus on ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC3138940/, fetched 2026-10-27
[S3] Medical Volume Rendering Techniques, https://arxiv.org/pdf/1802.07710v1, fetched 2026-10-27 (unofficial)
[S4] PDF Direct Volume Rendering, https://www.cs.unh.edu/~cs880/lectureNotes/DVR.pdf, fetched 2026-10-28 (unofficial)
