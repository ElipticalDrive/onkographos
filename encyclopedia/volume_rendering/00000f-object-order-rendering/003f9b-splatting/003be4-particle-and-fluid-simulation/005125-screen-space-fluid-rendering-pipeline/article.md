# Screen-Space Fluid Rendering Pipeline

Also called: Van der Laan et al. pipeline, vtkOpenGLFluidMapper pipeline

## Overview
The Screen-Space Fluid Rendering Pipeline is a real-time rendering technique for visualizing particle-based fluid simulations (such as Smoothed Particle Hydrodynamics) by converting discrete particle data into a continuous fluid volume appearance using depth and thickness buffers, thereby avoiding the computational cost of ray tracing [S1, unofficial]. It distinguishes itself from standard particle splatting by accumulating sphere thickness in screen space to derive surface normals and opacity, allowing for the rendering of millions of particles at interactive frame rates [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of particles $P_i$, typically provided as `vtkPolyData` [S1, unofficial]. Each particle $i$ carries position $\mathbf{p}_i$, radius $r_i$, and optional scalar color $\mathbf{c}_i$ [S1, unofficial].
*   **Output:** A final composited image with fluid appearance, including surface normals and opacity derived from screen-space buffers [S1, unofficial].

**Data Structures:**
The pipeline utilizes several render targets (framebuffers) stored in screen space:
1.  **Depth Buffer:** Stores the depth of the nearest particle surface.
2.  **Thickness Buffer:** Accumulates the radial thickness of overlapping particles.
3.  **Color Buffer (Optional):** Accumulates particle colors.
4.  **Filtered Depth Buffer:** Smoothed version of the depth buffer for normal estimation.
5.  **Filtered Thickness Buffer:** Smoothed version of the thickness buffer for opacity/density.

**Numbered Steps:**

1.  **Depth Pass:** Render all particles as spheres into the **Depth Buffer**. Enable the depth test. This creates a "bumpy" surface representing the fluid's outer boundary, where occlusion by static objects (e.g., a dragon) is automatically handled by discarding fluid pixels behind the occluder [S1, unofficial].

2.  **Thickness Pass:** Render all particles as spheres into the **Thickness Buffer**. Disable the depth test. Enable blending (additive) to accumulate the thickness of overlapping spheres. The contribution of each particle to a pixel is based on its spherical kernel [S1, unofficial].

3.  **Color Pass (Optional):** If particle colors are present, repeat the thickness accumulation pass into a **Color Buffer** using additive blending to accumulate color intensity [S1, unofficial].

4.  **Filtering:** Apply filtering to the noisy buffers to produce smooth visual results [S1, unofficial].
    *   **Thickness/Color Filtering:** Apply a high-performance, separable Gaussian filter. This consists of 1-D Gaussian filters applied in each dimension (horizontal then vertical) [S1, unofficial].
    *   **Depth Filtering:** Apply a depth-specific filter to the Depth Buffer to create the **Filtered Depth Buffer**. This step is critical to distinguish between distinct surfaces and surface variations due to grazing angles [S1, unofficial]. Two modes are available:
        *   *Bilateral Gaussian Filter:* Simpler, lower computation, but lower surface quality [S1, unofficial].
        *   *Narrow-Range Filter (Default):* Higher quality, handles the distinction between distinct surfaces and grazing angle smoothing more effectively [S1, unofficial].

5.  **Composition:** Combine the **Filtered Depth Buffer** (to derive normals), **Filtered Thickness Buffer** (for opacity/density), and **Color Buffer** (if used) into the final image. Normals are typically estimated from the gradient of the filtered depth buffer [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def screen_space_fluid_rendering(particles, camera, static_occluders):
    # Initialize screen-space buffers
    depth_buffer = create_buffer(width, height)
    thickness_buffer = create_buffer(width, height, clear=0.0)
    color_buffer = create_buffer(width, height, clear=0.0)
    
    # Pass 1: Depth Buffer with Depth Test
    set_depth_test(enabled=True)
    set_depth_func(LESS)
    render_particles_as_spheres(particles, target=depth_buffer, 
                                write_color=False, write_depth=True)
    
    # Pass 2: Thickness Buffer without Depth Test, Additive Blending
    set_depth_test(enabled=False)
    set_blend_func(src=ONE, dst=ONE) # Additive blending
    render_particles_as_spheres(particles, target=thickness_buffer, 
                                write_color=False, write_depth=False)
    
    # Pass 3: Color Buffer (Optional)
    if particles_have_color:
        set_blend_func(src=ONE, dst=ONE)
        render_particles_as_spheres(particles, target=color_buffer, 
                                    write_color=True, write_depth=False)
        
    # Filtering
    # Thickness and Color use separable Gaussian
    filtered_thickness = apply_separable_gaussian(thickness_buffer)
    filtered_color = apply_separable_gaussian(color_buffer) if particles_have_color else None
    
    # Depth uses Narrow-Range Filter (default) or Bilateral Gaussian
    filtered_depth = apply_narrow_range_filter(depth_buffer)
    
    # Composition
    # Normals are derived from filtered_depth gradient
    normals = compute_normals_from_depth(filtered_depth)
    opacity = map_thickness_to_opacity(filtered_thickness)
    
    final_image = composite(final_image, opacity, normals, filtered_color)
    return final_image

def render_particles_as_spheres(particles, target, write_color, write_depth):
    for particle in particles:
        # Project particle sphere to screen space
        screen_pos, radius = project_sphere(particle.position, particle.radius)
        
        # Rasterize sphere into target buffer
        # For depth: write min depth
        # For thickness: add sphere thickness value (e.g., sqrt(r^2 - d^2))
        rasterize_sphere(screen_pos, radius, target, write_color, write_depth)
```

**Parameters:**
*   **Depth Filter Mode:** 'Bilateral Gaussian Filter' or 'Narrow-Range Filter' (default) [S1, unofficial].
*   **Gaussian Kernel Size:** For thickness/color filtering, not explicitly sized in sources, but described as separable 1-D [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Grazing Angles:** At grazing angles, depth differences within the same surface can cause noise. The depth filter must smooth these without blending distinct surfaces [S1, unofficial].
*   **Occlusion:** The pipeline correctly handles occlusion by static objects during the depth pass [S1, unofficial].
*   **Artifacts:** Discontinuous/noisy surfaces at grazing angles and popping artifacts during camera motion occur due to lack of spatial information in the depth buffer [S1, unofficial].

## Operating characteristics
*   **Performance:** The pipeline renders more than 10 million particles in real-time [S1, unofficial].
*   **Hardware:** Benchmarked on Intel Core i7-8700K, 32GB RAM, NVIDIA 1080Ti GPU [S1, unofficial].
*   **Resolution:** 1920x1080 pixels [S1, unofficial].
*   **Timing:** Render time averages over 1000 frames with explicit GPU synchronization (`glFinish`) [S1, unofficial].
*   **Strengths:** Real-time performance for large-scale simulations; simple implementation using standard rasterization; automatic occlusion handling [S1, unofficial].
*   **Limitations:** Refraction and reflection are non-physical (approximate environment sampling); artifacts at grazing angles and during camera motion; limited to screen-space information [S1, unofficial].

## Background
The pipeline is based on the screen-space fluid rendering method introduced by Van der Laan et al. [S1, unofficial]. It was implemented in the Visualization Toolkit (VTK) as the `vtkOpenGLFluidMapper` class to provide efficient real-time visualization of large-scale particle-based fluid simulations [S1, unofficial].

## Sources
[S1] Screen-Space Fluid Rendering in VTK - Kitware, https://www.kitware.com/screen-space-fluid-rendering-vtk/, fetched 2026-10-27 (unofficial)
