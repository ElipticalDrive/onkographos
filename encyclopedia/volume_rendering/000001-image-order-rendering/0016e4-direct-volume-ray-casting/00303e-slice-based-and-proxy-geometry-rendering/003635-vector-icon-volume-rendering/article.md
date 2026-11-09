# Vector Icon Volume Rendering
Also called: Billboard vector icons, Vector Icon Splatting with Billboard Textures, Vector icons embedded in splat footprint, Volume rendering with embedded vector icons

## Overview
Vector Icon Volume Rendering is a 3D raster resampling technique that integrates vector field visualization into scalar volume rendering by embedding small billboard images containing vector icons within splat footprints [S1, unofficial]. It solves the problem of visualizing flow direction and magnitude within volumetric data by creating an illusion of coherent flow through stochastic placement and compositing of these icons [S1, unofficial]. It differs from standard direct volume ray casting by replacing or augmenting the scalar sampling integral with discrete, oriented billboard primitives that indicate vector orientation, rather than relying solely on color or opacity transfer functions for directionality [S1, unofficial].

## How it works
The method builds upon the sheet-based compositing structure of the parent [Slice-based and Proxy Geometry Rendering](6225-slice-based-and-proxy-geometry-rendering.md). It operates on a per-sheet basis, where the volume is processed in sheets oriented parallel to the image plane, composited in back-to-front order [S1, unofficial].

### Inputs and Outputs
**Inputs:**
*   Scalar volume data (e.g., density, cloudiness) [S1, unofficial].
*   Vector field data (e.g., wind velocity) defined on a grid [S1, unofficial].
*   Camera parameters (viewing ray direction) [S1, unofficial].
*   Time step $t$ for time-varying fields [S1, unofficial].

**Outputs:**
*   A 2D image where scalar fields are rendered as density clouds and vector fields are rendered as colored, oriented icons [S1, unofficial].

### Data Structures
*   **Sheet Grid:** A 2D regular grid sampling of the current sheet parallel to the image plane [S1, unofficial].
*   **Billboard Texture:** A texture atlas containing discrete sets of billboard images. Each image contains a vector icon. The set includes variations for different flow orientations (e.g., largest icon for flow parallel to screen, small dot for flow parallel to line of sight) [S1, unofficial].
*   **Motion Texture Set:** A set of textures where icons move across the texture, fading in and out, to simulate continuous motion [S1, unofficial].

### Steps
1.  **Sheet Selection:** Select a sheet oriented parallel to the image plane. Composite sheets in back-to-front order [S1, unofficial].
2.  **Grid Sampling:** Take a regular grid sampling on the current sheet [S1, unofficial].
3.  **Splat Placement:** Place an image reconstruction operator (splat) over each grid point [S1, unofficial].
4.  **Stochastic Icon Inclusion:** Use a stochastic heuristic to determine whether the splat should include a vector icon cylinder [S1, unofficial].
5.  **Icon Orientation and Selection:**
    *   If included, orient the billboard image such that it lies perpendicular to the viewing ray and parallel to the projected vector direction at the splat's center point [S1, unofficial].
    *   Select the appropriate icon from the discrete set based on the flow direction relative to the viewer:
        *   If flow is parallel to the screen, choose the largest icon [S1, unofficial].
        *   If flow is parallel to the viewer’s line of sight, reduce the icon to a small dot [S1, unofficial].
    *   Randomly offset the cylinder/icon from the center of the splat [S1, unofficial].
6.  **Color Mapping:** Assign color to the vector icon based on scalar attributes, such as altitude [S1, unofficial].
7.  **Compositing:** Add the volume integral for each pixel under the splat and the cylinder. Composite the billboard images with the underlying scalar volume rendering [S1, unofficial].
8.  **Animation (Optional):** For time-varying fields, cycle through the motion texture set at rendering time, where icons move across the texture and fade in/out, to achieve the illusion of motion and alleviate sparse representation [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_vector_icon_volume(scalar_volume, vector_field, camera, time_step):
    # Initialize output image
    output_image = zeros_like(camera.image_plane)
    
    # Process sheets in back-to-front order
    for sheet in get_sheets_parallel_to_image_plane(scalar_volume, camera):
        sheet_image = zeros_like(camera.image_plane)
        
        # Regular grid sampling on the current sheet
        for grid_point in sheet.grid:
            # Retrieve vector at grid point
            vec = vector_field.sample(grid_point.position)
            
            # Stochastic heuristic to determine if icon is included
            if stochastic_include_icon(vec):
                # Determine orientation: perpendicular to viewing ray, parallel to projected vector
                billboard = get_billboard_texture(vec, camera.view_direction)
                
                # Select icon size based on flow direction
                if is_flow_parallel_to_screen(vec, camera):
                    icon_variant = billboard.largest_icon
                elif is_flow_parallel_to_line_of_sight(vec, camera):
                    icon_variant = billboard.small_dot
                else:
                    icon_variant = billboard.intermediate_icon
                
                # Random offset from center
                offset = random_offset()
                final_position = grid_point.position + offset
                
                # Color based on altitude (scalar attribute)
                altitude = scalar_volume.sample(grid_point.position)
                color = map_altitude_to_color(altitude)
                
                # Render billboard to sheet_image
                render_billboard(sheet_image, final_position, icon_variant, color)
        
        # Composite sheet into output image (back-to-front)
        output_image = composite_back_to_front(output_image, sheet_image)
        
        # Optional: Animate using motion textures
        if animate:
            output_image = apply_motion_texture_cycle(output_image, time_step)
            
    return output_image
```

### Parameters
*   **Stochastic Heuristic Threshold:** Not given in the sources. Determines the probability of placing an icon.
*   **Icon Size Variations:** Not given in the sources. Specific pixel dimensions for the "largest icon" and "small dot".
*   **Color Mapping Function:** Maps altitude to color. Not given in the sources.

### Edge Cases and Pitfalls
*   **Sparse Representation:** The stochastic placement can lead to a sparse and discrete representation of the flow, making it difficult to understand [S1, unofficial]. Mitigated by using motion textures [S1, unofficial].
*   **Direction Ambiguity:** When flow is parallel to the line of sight, the icon reduces to a dot, losing directional information [S1, unofficial].
*   **Precision:** Random offsetting requires careful handling to avoid aliasing or gaps in the icon distribution [G].

## Subtopics

### Billboard Vector Icons
Uses discrete billboard images oriented to the viewer to represent vector direction. Chosen when explicit directional cues are needed within a volumetric context. [S1, unofficial]

### Vector Icon Splatting with Billboard Textures
Extends the basic billboard approach by using texture atlases and motion cycles to simulate continuous flow. Chosen when animation and reduced sparsity are priorities. [S1, unofficial]

### Vector Icons Embedded in Splat Footprint
Integrates the icon directly into the splat's reconstruction kernel. Chosen for tighter integration with the scalar volume rendering integral. [S1, unofficial]

### Volume Rendering with Embedded Vector Icons
General term for the technique of merging scalar volume rendering with vector field visualization via icons. [S1, unofficial]

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of grid points per sheet and the number of sheets. The stochastic check and texture lookup are constant time per grid point.
*   **Memory Complexity:** Proportional to the size of the scalar and vector volumes plus the size of the billboard texture atlas.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Provides clear directional cues; integrates well with scalar volume rendering (e.g., density clouds); creates an illusion of coherent flow through stochastic placement and motion [S1, unofficial].
*   **Limitations:** Sparse and discrete representation can make flow difficult to understand; directionality is lost when flow is parallel to the line of sight [S1, unofficial].
*   **When to Use:** When visualizing time-varying vector fields (e.g., wind) within volumetric scalar data (e.g., clouds) is required, and explicit directional icons are preferred over streamlines or glyphs [S1, unofficial].
*   **When Not to Use:** When dense, continuous flow visualization is needed without the discrete nature of icons, or when performance is critical and texture lookups are expensive [S1, unofficial].

## Background
The technique was developed by Crawfis and Max [S1, unofficial]. It is a 3D raster resampling technique that modifies the volume integral to include the rendering of tiny cylinders within splats [S1, unofficial]. It was applied to time-varying wind fields and global climate simulations, merging white density clouds (percent cloudiness) with colored vector icons (wind velocity) [S1, unofficial]. The work is associated with the 9th International Symposium on Flow Visualization, Heriot-Watt University, Edinburgh, 2000 [S1, unofficial].

## Sources
[S1] Flow Visualization Techniques for CDF using Volume Rendering, https://www.osti.gov/servlets/purl/790542, fetched 2026-10-26 (unofficial)
