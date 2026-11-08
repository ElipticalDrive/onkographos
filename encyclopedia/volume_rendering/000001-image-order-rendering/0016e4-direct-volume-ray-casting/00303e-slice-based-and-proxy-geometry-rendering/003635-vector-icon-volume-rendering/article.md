# Vector Icon Volume Rendering

Also called: Billboard vector icons, Vector Icon Splatting with Billboard Textures, Vector icons embedded in splat footprint, Volume Rendering with Embedded Vector Icons

## Overview
Vector Icon Volume Rendering is a 3D raster resampling technique that integrates vector field visualization into volume rendering by embedding small billboard images containing vector icons within the splat footprint of scalar data [S1, unofficial]. It solves the problem of visualizing flow direction and magnitude within a volumetric scalar field (such as density clouds) by creating an illusion of coherent flow through stochastic placement and orientation of these icons [S1, unofficial]. It sets itself apart from standard scalar volume rendering by modifying the volume integral to include the rendering of tiny cylinders or icons within the splat neighborhood, and from pure vector visualization by merging the flow representation with the underlying scalar field density [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A 3D scalar field (e.g., density), a 3D vector field (e.g., wind velocity), a transfer function mapping scalar values to RGBA, and a set of pre-defined billboard textures containing vector icons.
*   **Outputs:** A 2D image where scalar density is rendered via standard volume splatting, and vector flow is indicated by overlaid, oriented icons.

**Data Structures:**
*   **Volume Data:** Stored as a 3D texture or stack of 2D textures on the GPU [S1, unofficial].
*   **Transfer Function:** A 1D texture mapping scalar values to color and opacity [S1, unofficial].
*   **Billboard Textures:** A set of 2D textures containing vector icons of varying sizes and orientations. These textures are cycled during rendering to simulate motion [S1, unofficial].

**Method Steps:**
The method builds upon the slice-based proxy geometry rendering pipeline [Parent]. The specific modifications for vector icons are:

1.  **Sheet Construction:** Build volume rendering in sheets oriented parallel to the image plane [S1, unofficial].
2.  **Grid Sampling:** Take a regular grid sampling on the current sheet [S1, unofficial].
3.  **Splat Placement:** Place an image reconstruction operator (splat) over each grid point [S1, unofficial].
4.  **Stochastic Icon Inclusion:** Use a stochastic heuristic to determine whether a specific splat should include a vector icon (cylinder) [S1, unofficial].
5.  **Icon Offset:** If the heuristic includes an icon, randomly offset the cylinder from the center of the splat [S1, unofficial].
6.  **Billboard Orientation:** Orient the billboard image such that it lies perpendicular to the viewing ray and parallel to the projected vector direction at the splat's center point [S1, unofficial].
7.  **Icon Selection:** Select the specific billboard texture based on the flow direction relative to the viewer:
    *   If the flow is parallel to the screen, choose the largest icon [S1, unofficial].
    *   If the flow is parallel to the viewer’s line of sight, reduce the icon to a small dot [S1, unofficial].
    *   Intermediate orientations use intermediate icon sizes to indicate direction into/out of the screen [S1, unofficial].
8.  **Color Mapping:** Assign color to the vector icons based on altitude (or another scalar attribute) [S1, unofficial].
9.  **Volume Integral Update:** Modify the standard volume integral to include the rendering contribution of the tiny cylinder/icon within the small neighborhood of the splat [S1, unofficial]. Add the volume integral for each pixel under both the scalar splat and the vector icon [S1, unofficial].
10. **Compositing:** Composite the sheets in a back-to-front order using alpha blending [S1, unofficial].
11. **Animation (Optional):** To alleviate the sparse and discrete representation of icons, cycle through a set of textures where icons move across the texture, fading in and out, to achieve the illusion of continuous motion [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_vector_icon_volume(volume_texture, vector_field, transfer_function, billboard_textures, camera):
    # Initialize back-to-front compositing buffer
    color_buffer = zeros(screen_width, screen_height, 4)
    
    # 1. Build sheets oriented parallel to image plane
    for sheet in volume_sheets:
        # 2. Regular grid sampling on current sheet
        for grid_point in sheet.grid:
            # 3. Place splat over grid point
            splat_footprint = get_splat_footprint(grid_point)
            
            # Sample scalar value for standard volume rendering
            scalar_val = sample_volume(volume_texture, grid_point.position)
            scalar_rgba = sample_transfer_function(transfer_function, scalar_val)
            
            # 4. Stochastic heuristic for icon inclusion
            if stochastic_include_icon(scalar_val, grid_point.position):
                # 5. Random offset for icon
                offset = random_offset()
                icon_position = grid_point.position + offset
                
                # 6. Get vector direction at splat center
                vector_dir = sample_vector_field(vector_field, grid_point.position)
                
                # 7. Determine billboard orientation and selection
                # Orient billboard perpendicular to viewing ray and parallel to projected vector
                billboard_transform = create_billboard_transform(vector_dir, camera.view_direction)
                
                # Select icon size based on flow alignment
                if is_parallel_to_screen(vector_dir, camera):
                    icon_tex = billboard_textures[LARGEST_ICON]
                elif is_parallel_to_view(vector_dir, camera):
                    icon_tex = billboard_textures[SMALL_DOT]
                else:
                    icon_tex = interpolate_icon_size(vector_dir, camera, billboard_textures)
                
                # 8. Color icons by altitude
                icon_color = map_altitude_to_color(icon_position.y)
                
                # 9. Render icon into buffer (additive to scalar splat)
                render_billboard(icon_tex, icon_position, billboard_transform, icon_color, color_buffer)
            
            # Render scalar splat
            render_splat(splat_footprint, scalar_rgba, color_buffer)
            
        # 10. Composite sheet back-to-front (handled by depth sorting of sheets)
        composite_sheet(color_buffer)

    return color_buffer
```

**Parameters:**
*   **Stochastic Heuristic Threshold:** Not given in the sources. [G] Typically a random value compared against a probability derived from vector magnitude or density.
*   **Icon Size Scaling:** Not given in the sources. [G] Maps the angle between the vector and view direction to a discrete set of icon sizes.
*   **Texture Cycling Rate:** Not given in the sources. [G] Determines the speed of the motion illusion.

**Edge Cases and Pitfalls:**
*   **Sparse Representation:** The discrete nature of icons can make flow difficult to understand in regions with low vector density [S1, unofficial].
*   **Occlusion:** Icons may be occluded by high-opacity scalar regions if not carefully composited.
*   **Precision:** Random offsetting requires careful handling of floating-point precision to avoid jittering artifacts in animation.

## Subtopics

### Billboard Vector Icons
A specific implementation where icons are rendered as 2D quads always facing the camera, embedded within the splat. Chosen when simple directional cues are sufficient. [S1, unofficial]

### Vector Icon Splatting with Billboard Textures
Uses pre-rendered textures for icons to improve performance. Chosen when rendering speed is critical and icon variety is limited. [S1, unofficial]

### Volume Rendering with Embedded Vector Icons
The general technique of modifying the splat footprint to include vector data. Chosen for merging scalar and vector visualization. [S1, unofficial]

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of grid points in the sheet and the number of splats. The stochastic check and icon selection add constant overhead per splat.
*   **Memory Complexity:** Requires storage for the 3D volume texture, 1D transfer function, and a set of billboard textures. The texture cycling mechanism requires storing multiple icon variations.
*   **Performance:** Efficient on standard rasterization hardware as it leverages the existing splatting pipeline. Performance degrades if the number of billboard textures is large or if complex blending is required.
*   **Strengths:** Intuitive visualization of flow direction and magnitude; merges well with scalar density; creates illusion of motion through animation.
*   **Limitations:** Sparse representation can obscure flow; stochastic placement may lead to inconsistent visual density; discrete icon sizes limit smooth directional representation.
*   **When to Use:** When visualizing wind fields, climate simulations, or other vector fields within a volumetric scalar context.
*   **When Not to Use:** When precise, continuous flow lines are required (use streamlines or line bundles instead) [S1, unofficial].

## Background
The technique was developed by Crawfis and Max [S1, unofficial]. It was presented at the 9th International Symposium on Flow Visualization, Heriot-Watt University, Edinburgh, 2000 [S1, unofficial]. The work builds on earlier 3D raster resampling techniques where volume rendering was built up in sheets [S1, unofficial]. The authors also developed related techniques such as Line Bundles, which rely on rendering and compositing many semitransparent line segments [S1, unofficial].

## Sources
[S1] Flow Visualization Techniques for CDF using Volume Rendering, https://www.osti.gov/servlets/purl/790542, fetched 2026-10-26 (unofficial)
