# Hybrid Surface-Volume Rendering

Also called: Gas Visualization System, Gaseous volume rendering, Realistic volumetric gas rendering

## Overview
Hybrid Surface-Volume Rendering is a method for visualizing Computational Fluid Dynamics (CFD) data that combines physically-based atmospheric attenuation and illumination models for low-albedo gases with a scanline-buffer algorithm for surface-defined objects [S1, unofficial]. Unlike traditional isosurface rendering which captures only a small fraction of data, this system displays the entirety of the three-dimensional data [S1, unofficial]. It sets itself apart from [Empirical Translucent Volume Shading](4109-empirical-translucent-volume-shading.md) by explicitly integrating surface fragments into the volume traversal loop to determine rendering extent and by using a specific low-albedo illumination model for gases rather than generic empirical shading.

## How it works
**Inputs and Outputs:**
*   **Inputs:** CFD simulation data (scalar fields defining density), camera parameters, scene geometry (surface-defined objects), and rendering flags (e.g., self-shadowing enabled).
*   **Outputs:** Final color image.

**Data Structures:**
*   **Z-buffer fragment list:** A list per pixel containing fragments that partially or fully cover the pixel. Each fragment has an average z-depth value.
*   **Solid shadow table:** A data structure storing shadowing information for solid objects, used when self-shadowing is enabled [S1, unofficial].

**Algorithm Steps:**
1.  **Surface Z-buffer Creation:** Create a z-buffer for a scanline. For each pixel, populate a list with all fragments from surface-defined objects that partially or fully cover the pixel [S1, unofficial].
2.  **Volume Extent Determination:** If a volume is active for a pixel, determine the extent of volume rendering needed based on the existing surface fragments [S1, unofficial].
3.  **Volume Rendering:** Perform volume rendering for the active sections. This creates z-buffer fragments for the separate sections of the volumes. Volume rendering ceases once full coverage of the pixel by volume or surface-defined elements is achieved [S1, unofficial].
4.  **Fragment Sorting:** Sort the z-buffer fragments (both surface and volume) into the z-buffer fragment list based on their average z-depth values [S1, unofficial].
5.  **Final Compositing:** Render the sorted z-buffer fragment list to produce the final color of the pixel [S1, unofficial].

**Gaseous Volume Rendering Sub-routine:**
For each section of gas within the determined extent:
1.  Trace the ray from the eye through the defining geometry of the volume [S1, unofficial].
2.  For each increment along the ray:
    a.  **Sampling:** Use a Monte Carlo method to choose the sample point to reduce aliasing artifacts [S1, unofficial].
    b.  **Data Retrieval:** Get the color, density, and opacity of the current element from the CFD data [S1, unofficial].
    c.  **Shadowing:** If self-shadowing is enabled, retrieve the shadowing of this element from the solid shadow table [S1, unofficial].
    d.  **Illumination Calculation:** Calculate gas illumination using opacity, density, and the appropriate low-albedo model [S1, unofficial].
    e.  **Accumulation:**
        $$ \text{final\_color} = \text{final\_color} + \text{color} $$
        $$ \text{sum\_density} = \text{sum\_density} + \text{density} $$
    f.  **Termination Check:** If transparency < 0.01, stop tracing [S1, unofficial].
    g.  **Increment:** Increment sample_point [S1, unofficial].
3.  **Fragment Creation:** Create the a-buffer (z-buffer) fragment for the accumulated gas section [S1, unofficial].

**Opacity Approximation:**
The opacity is approximated by integrating density along the ray. The approximation used is:
$$ \text{opacity} = \int_{s_i}^{s_e} \rho(s) ds $$
where $\tau$ is the optical depth of the material, $\rho$ is the density of the material, $s_i$ is the starting point for the volume tracing, and $s_e$ is the ending point [S1, unofficial]. The final increment along the ray may be smaller, so its opacity is scaled proportionally [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hybrid_surface_volume_rendering(scanline_pixels, surface_fragments, volumes, camera):
    # Step 1: Create z-buffer for scanline
    z_buffer_fragments = {pixel_id: [] for pixel_id in scanline_pixels}
    
    for frag in surface_fragments:
        if frag.overlaps_pixel(scanline_pixels):
            z_buffer_fragments[frag.pixel_id].append(frag)

    # Step 2 & 3: Volume Rendering
    for pixel_id, fragments in z_buffer_fragments.items():
        if volume_active_for_pixel(pixel_id, volumes):
            # Determine extent based on existing surface fragments
            extent = determine_volume_extent(pixel_id, fragments, volumes)
            
            # Perform volume rendering for each section
            for section in extent.sections:
                gas_fragments = render_gas_section(section, pixel_id, camera)
                fragments.extend(gas_fragments)

    # Step 4: Sort fragments by average z-depth
    for pixel_id, fragments in z_buffer_fragments.items():
        fragments.sort(key=lambda f: f.average_z_depth)

    # Step 5: Render final color
    image = {}
    for pixel_id, fragments in z_buffer_fragments.items():
        image[pixel_id] = composite_fragments(fragments)
    return image

def render_gas_section(gas_section, pixel_id, camera):
    ray = camera.get_ray(pixel_id)
    final_color = [0.0, 0.0, 0.0]
    sum_density = 0.0
    sample_point = ray.origin
    transparency = 1.0
    
    # Trace through defining geometry
    increments = trace_ray_through_geometry(ray, gas_section.geometry)
    
    for increment in increments:
        # Monte Carlo sampling to reduce aliasing
        sample_pos = monte_carlo_sample(increment.start, increment.end)
        
        # Get data from CFD grid
        color, density, opacity = get_cfd_data(sample_pos, gas_section.data)
        
        # Self-shadowing
        shadow_factor = 1.0
        if self_shadowing_enabled:
            shadow_factor = get_shadow_from_table(sample_pos, gas_section.shadow_table)
            
        # Calculate illumination
        lit_color = calculate_gas_illumination(opacity, density, shadow_factor)
        
        # Accumulate
        final_color = final_color + lit_color
        sum_density = sum_density + density
        
        # Update transparency for termination check
        # Note: The source implies transparency is tracked implicitly or via sum_density
        # Assuming transparency is derived from accumulated density or passed state
        # [G] Standard volume rendering transparency decay: T = T * (1 - opacity)
        # However, source says "if transparency < 0.01". We assume a transparency variable exists.
        # If not explicitly maintained, it might be 1 - (sum_density / max_density) or similar.
        # Given the explicit check, we assume a transparency accumulator.
        transparency = transparency * (1.0 - opacity) 
        
        if transparency < 0.01:
            break
            
        sample_point = sample_pos + increment.step
        
    # Create fragment
    fragment = ZBufferFragment(
        pixel_id=pixel_id,
        color=final_color,
        average_z_depth=calculate_average_z(ray, gas_section.geometry),
        density=sum_density
    )
    return [fragment]

def calculate_gas_illumination(opacity, density, shadow_factor):
    # Low-albedo illumination model for gases
    # [G] Specific formula not provided in source, but described as physically-based
    # atmospheric attenuation and illumination.
    # Placeholder for the specific model equation from [5, 11] referenced in [S1, unofficial]
    return density * shadow_factor * atmospheric_attenuation_model(opacity)
```

## Operating characteristics
*   **Time Complexity:** Not explicitly given in complexity notation, but rendering times were measured on a HP 9000 Series Workstation model 735 [S1, unofficial].
*   **Measured Performance:**
    *   Sophisticated illumination model: ~83 seconds [S1, unofficial].
    *   Simplified illumination with emphasis on material modeling: ~64 seconds [S1, unofficial].
    *   Basic illumination model: ~27 seconds [S1, unofficial].
*   **Memory Complexity:** Requires storage for the z-buffer fragment list per pixel and a solid shadow table [S1, unofficial].
*   **Strengths:** Displays the entirety of 3D CFD data, unlike isosurface renderers which hide information; fast combination of surface and volume data; physically-based illumination for low-albedo gases [S1, unofficial].
*   **Limitations:** Performance is sensitive to the complexity of the illumination model; requires resampling curvilinear grids to rectilinear grids for visualization [S1, unofficial].
*   **When to use:** When visualizing low-albedo gaseous CFD data where physical accuracy of illumination and full data visibility are prioritized over speed.
*   **When not to use:** When real-time performance is required or when the data is not gaseous/low-albedo.

## Background
*   **Origin:** Described in a paper by David S. Ebert, Roni Yagel, James N. Scott, and Yair Kurzion at IEEE Visualization '94, October 17-21, 1994 [S1, unofficial].
*   **History:** Developed to address the limitations of current CFD visualization systems that failed to accurately display the entirety of 3D data. Traditional surface-based techniques (stream-lines, contour plots) capture only a small fraction of the data [S1, unofficial]. The system was tested on turbo-jet compressor visualization data, where the computational mesh was a 3D curvilinear grid resampled to a rectilinear grid [S1, unofficial].
*   **Usage:** Used for visualizing complex three-dimensional flow phenomena in aerospace propulsion systems [S1, unofficial].

## Sources
[S1] PDF Volume Rendering Methods for Computational Fluid Dynamics Visualization, https://www.researchgate.net/profile/David-Ebert-4/publication/220943451_Volume_Rendering_Methods_for_Computational_Fluid_Dynamics_Visualization/links/00b7d51a73e2892e23000000/Volume-Rendering-Methods-for-Computational-Fluid-Dynamics-Visualization.pdf, fetched 2026-10-27 (unofficial)
