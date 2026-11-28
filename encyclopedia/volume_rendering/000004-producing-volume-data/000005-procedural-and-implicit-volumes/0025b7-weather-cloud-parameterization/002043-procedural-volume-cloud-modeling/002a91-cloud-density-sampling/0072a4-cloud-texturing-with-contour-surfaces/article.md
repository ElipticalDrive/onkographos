# Cloud Texturing with Contour Surfaces

Also called: Gardner's Cloud Rendering, Poor Man's Fractal Texture

## Overview
This method renders cloud density variables as contour surfaces with transparency, using a 3D volume texture function to determine color and opacity, allowing partial cloud cover detail within a single grid element [S1, unofficial]. It differs from direct volume rendering by providing a more faithful representation of the cloud density variable's meaning (percent cloud cover) by creating transparency where the texture function is below a threshold [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Cloud density data (e.g., from a climate model), wind velocity field for advection, terrain geometry (for z-buffer depth).
*   **Outputs:** Rendered image with translucent clouds and atmospheric haze.

**Data Structures:**
*   **Grid:** 3D grid of cloud density values. Cell size is approximately 1.125 degrees on a side (about 100 km square at the equator) [S1, unofficial].
*   **Texture Coordinates:** 3D coordinates $(u, v, w)$ ranging from 0 to 1, proportional to longitude, latitude, and altitude respectively [S1, unofficial]. Longitude wraps around at the Greenwich meridian ($u=0$ and $u=1$) [S1, unofficial].
*   **Z-Buffer:** Stores depth of opaque terrain or back-facing polygons [S1, unofficial].
*   **Density Buffer (Optional):** Accumulates integrated densities from scan-converted volume cells [S1, unofficial].

**Steps:**

1.  **Smoothing:** Apply a 3x3x3 smoothing kernel to the cloud density intensity before contouring to reduce roughness [S1, unofficial].
2.  **Texture Coordinate Advection:** Advect the 3D texture coordinates $(u, v, w)$ by the wind field to produce new coordinates for each frame [S1, unofficial].
3.  **Contouring:** Sort volume cells back to front. Generate front-facing cloud contour surfaces from the smoothed density data [S1, unofficial].
4.  **Thickness Calculation:**
    *   When an opaque terrain polygon or a back-facing contour polygon is scan-converted, set the z-buffer to the polygon's depth at each pixel [S1, unofficial].
    *   When a front-facing contour polygon is rendered, calculate cloud thickness by subtracting the polygon's depth from the z-buffer depth at each pixel [S1, unofficial]. This represents the length of the viewing ray inside the volume bounded by the contour surface [S1, unofficial].
5.  **Texture Function Evaluation:** Evaluate the 3D volume texture function at surface points to determine color and opacity [S1, unofficial].
    *   The texture function is a sum of 3D plane sine waves:
        $$f(u, v, w) = \sum_i a_i \cos(b_i u + c_i v + d_i w + q_i)$$
    *   Phase perturbation breaks up regularity:
        $$q_i = 2 \sin (0.5 b_i u - 0.5 a_i v)$$
    *   Add a term proportional to cloud thickness to the texture function before comparing to a threshold to make clouds appear more opaque where thicker [S1, unofficial].
    *   If the function value is below a threshold, clouds become completely transparent and colorless [S1, unofficial].
6.  **Shading:** Render front-facing cloud contour surfaces using Phong shading with the Gardner color and transparency texture [S1, unofficial].
7.  **Haze Integration (Optional):** Add haze with density decreasing exponentially with altitude to serve as a depth cue. Integrate analytically along ray segments using the normal error function stored in a table [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_clouds(grid, wind_field, terrain_z_buffer, haze_table):
    # Step 1: Smooth density
    smoothed_grid = apply_3x3x3_smoothing_kernel(grid)
    
    # Step 2: Advect texture coordinates
    # Note: Advection logic depends on specific wind integration method
    # For each grid vertex, update (u, v, w) based on wind velocity over time step
    advected_coords = advect_texture_coordinates(grid.vertices, wind_field)
    
    # Step 3: Sort cells and generate surfaces
    sorted_cells = sort_cells_back_to_front(smoothed_grid)
    front_facing_surfaces = []
    for cell in sorted_cells:
        surface = generate_contour_surface(cell, smoothed_grid)
        if surface.is_front_facing():
            front_facing_surfaces.append(surface)
            
    # Step 4 & 5: Render surfaces with thickness and texture
    for surface in front_facing_surfaces:
        for pixel in surface.scan_convert():
            # Get depth from z-buffer (set by terrain or back-facing polygons)
            z_buffer_depth = get_z_buffer_depth(pixel)
            surface_depth = get_surface_depth(pixel)
            
            # Calculate thickness
            thickness = z_buffer_depth - surface_depth
            
            # Interpolate texture coordinates for this pixel
            u, v, w = interpolate_texture_coordinates(surface, pixel)
            
            # Evaluate texture function
            texture_value = evaluate_texture_function(u, v, w)
            
            # Add thickness term
            texture_value += thickness * thickness_factor
            
            # Determine opacity and color based on threshold
            if texture_value < threshold:
                opacity = 0.0
                color = (0, 0, 0)
            else:
                opacity = map_texture_to_opacity(texture_value)
                color = map_texture_to_color(texture_value)
                
            # Apply Phong shading
            shaded_color = phong_shading(surface.normal, pixel.position, color)
            
            # Composite with background (considering haze if enabled)
            composite_pixel(pixel, shaded_color, opacity, haze_table)

def evaluate_texture_function(u, v, w):
    value = 0.0
    for i in range(num_waves):
        # Phase perturbation
        q_i = 2.0 * sin(0.5 * b[i] * u - 0.5 * a[i] * v)
        # Sine wave component
        phase = b[i] * u + c[i] * v + d[i] * w + q_i
        value += a[i] * cos(phase)
    return value
```

**Parameters:**
*   **Smoothing Kernel:** 3x3x3 [S1, unofficial].
*   **Grid Cell Size:** ~1.125 degrees (100 km at equator) [S1, unofficial].
*   **Texture Function Waves:** Sum of 3D plane sine waves [S1, unofficial].
*   **Phase Perturbation Amplitude:** 2 [S1, unofficial].
*   **Phase Perturbation Frequency Factors:** 0.5 [S1, unofficial].
*   **Threshold:** Not given in the sources.
*   **Thickness Factor:** Not given in the sources.
*   **Haze Density Decay:** Exponential with altitude [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Roughness:** Original cloud density contouring can be too rough and bumpy; smoothing is required [S1, unofficial].
*   **Constant Density Assumption:** The cloud thickness effect assumes constant density inside the contour surface [S1, unofficial].
*   **Variable Density:** Variable density can be accounted for by scan converting volume cells, accumulating integrated densities in a special buffer, and using that to modify the transparency threshold, though this was found not to add realism [S1, unofficial].
*   **Texture Coordinate Wrapping:** Longitude wraps around at the Greenwich meridian ($u=0$ and $u=1$) [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Creates partial cloud cover detail within a single grid element; more faithful representation of cloud density variable meaning than direct volume rendering; haze serves as a depth cue [S1, unofficial].
*   **Limitations:** Contour surfaces have complicated shape and topology, changing for every frame of animation; variable density accounting did not add realism [S1, unofficial].
*   **When to Use:** When rendering cloud density variables from climate models where percent cloud cover is the key metric and partial transparency is desired [S1, unofficial].
*   **When Not to Use:** When high-fidelity variable density representation is critical and the added complexity of density buffering is justified [S1, unofficial].

## Background
This method is based on the cloud rendering method of Gardner [S1, unofficial]. It was used in the context of 3D scalar and vector field visualization, specifically for rendering cloud data from climate models [S1, unofficial]. The technique was developed to address the limitations of direct volume rendering in representing the meaning of cloud density variables [S1, unofficial].

## Sources
[S1] New techniques in 3D scalar and vector field visualization, https://digital.library.unt.edu/ark:/67531/metadc1384657/?q=volume%20splatting, fetched 2026-10-28 (unofficial)
