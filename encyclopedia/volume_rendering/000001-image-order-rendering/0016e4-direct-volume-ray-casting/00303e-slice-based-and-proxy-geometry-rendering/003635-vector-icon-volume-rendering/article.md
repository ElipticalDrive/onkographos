# Vector Icon Volume Rendering

Also called: Billboard vector icons, Vector Icon Splatting with Billboard Textures, Vector icons embedded in splat footprint, Volume Rendering with Embedded Vector Icons

## Overview
Vector Icon Volume Rendering is a 3D raster resampling technique that integrates vector field visualization into scalar volume rendering by embedding tiny cylinder-like icons within splat footprints [S1, unofficial]. It solves the problem of visualizing flow direction and magnitude within volumetric data by creating an illusion of coherent motion through stochastic placement and orientation of billboard textures [S1, unofficial]. It differs from standard scalar volume rendering by modifying the volume integral to include directional primitives, and from pure flow visualization by merging these icons with density clouds in a single back-to-front compositing pass [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A 3D scalar field (e.g., density/altitude), a 3D vector field (e.g., wind velocity), a set of billboard textures containing vector icons, and camera parameters.
*   **Outputs:** A 2D raster image where scalar volume is composited with directional vector icons.

**Data Structures:**
*   **Billboard Textures:** A discrete set of 2D textures. Each texture contains a vector icon (e.g., an arrow or cylinder) and potentially a fading mask. The icons vary in size to indicate depth/orientation relative to the viewer [S1, unofficial].
*   **Splat Grid:** A regular grid sampling defined on the current sheet parallel to the image plane [S1, unofficial].

**Steps:**
1.  **Sheet Generation:** Construct a sheet of proxy geometry oriented parallel to the image plane, intersecting the volume [S1, unofficial].
2.  **Grid Sampling:** Sample the vector and scalar fields on a regular grid over the current sheet [S1, unofficial].
3.  **Stochastic Placement:** For each grid point, apply a stochastic heuristic to determine if a vector icon (cylinder) should be rendered [S1, unofficial].
4.  **Icon Selection and Orientation:**
    *   Calculate the projected vector direction at the splat's center point [S1, unofficial].
    *   Orient the billboard image such that it lies perpendicular to the viewing ray and parallel to the projected vector direction [S1, unofficial].
    *   Select the specific billboard texture based on the flow angle:
        *   If flow is parallel to the screen, choose the largest icon [S1, unofficial].
        *   If flow is parallel to the viewer’s line of sight, reduce the icon to a small dot [S1, unofficial].
    *   If the stochastic heuristic passes, randomly offset the cylinder from the grid center [S1, unofficial].
5.  **Color Mapping:** Map the scalar value (e.g., altitude) to the color of the vector icon [S1, unofficial].
6.  **Compositing:**
    *   Render the scalar field splat (density cloud) for the grid point [S1, unofficial].
    *   Render the vector icon billboard if selected [S1, unofficial].
    *   Add the volume integral contributions for each pixel under both the splat and the cylinder [S1, unofficial].
    *   Composite the results back-to-front [S1, unofficial].
7.  **Animation (Optional):** To alleviate sparse representation and create motion, cycle through a set of textures where icons move across the texture, fading in and out, at rendering time [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_vector_icon_volume(scalar_field, vector_field, billboard_textures, camera):
    # 1. Generate sheets parallel to image plane
    sheets = generate_sheets_parallel_to_image_plane(scalar_field.bounding_box, camera)
    
    # 2. Sort sheets back-to-front
    sheets.sort(key=lambda s: s.depth_from_camera, reverse=True)
    
    frame_buffer = initialize_frame_buffer()
    
    for sheet in sheets:
        # 3. Regular grid sampling on the current sheet
        grid_points = sample_regular_grid(sheet)
        
        for point in grid_points:
            scalar_val = scalar_field.sample(point.position)
            vector_val = vector_field.sample(point.position)
            
            # 4. Stochastic heuristic for icon placement
            if stochastic_heuristic(vector_val):
                # 5. Determine orientation and size
                projected_vector = project_vector(vector_val, camera)
                billboard = select_billboard_texture(projected_vector, billboard_textures)
                
                # 6. Random offset from center
                offset = random_offset()
                icon_position = point.position + offset
                
                # 7. Orientation: Perpendicular to view ray, parallel to projected vector
                billboard_orientation = compute_billboard_orientation(
                    camera.view_direction, 
                    projected_vector
                )
                
                # 8. Color based on altitude/scalar value
                icon_color = map_scalar_to_color(scalar_val)
                
                # 9. Render and composite
                # Render scalar splat
                render_splat(frame_buffer, point.position, scalar_val, billboard_textures[0]) # Assuming scalar is base splat
                
                # Render vector icon
                render_billboard(
                    frame_buffer, 
                    icon_position, 
                    billboard, 
                    billboard_orientation, 
                    icon_color
                )
    
    return frame_buffer

def select_billboard_texture(projected_vector, textures):
    # Select largest icon if flow parallel to screen
    if is_parallel_to_screen(projected_vector):
        return textures.largest_icon
    
    # Reduce to small dot if flow parallel to line of sight
    if is_parallel_to_view_direction(projected_vector):
        return textures.smallest_dot
    
    # Intermediate sizes for other angles
    return textures.get_icon_for_angle(projected_vector)
```

**Parameters:**
*   **Stochastic Heuristic Threshold:** Not given in the sources.
*   **Grid Resolution:** Not given in the sources.
*   **Icon Size Scaling:** Not given in the sources.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of grid points and sheets. The stochastic check and texture selection are constant time per point.
*   **Memory Complexity:** Depends on the number of billboard textures and the volume data. The discrete nature of icons reduces memory compared to dense ray casting but requires texture storage.
*   **Strengths:** Provides clear directional cues; merges scalar and vector fields seamlessly; creates an illusion of coherent flow through animation and stochastic placement.
*   **Limitations:** Sparse and discrete representation can make flow difficult to understand; relies on stochastic placement which may lead to visual noise or gaps.
*   **When to use:** When visualizing time-varying wind or fluid flow fields where direction is critical and must be integrated with density information.
*   **When not to use:** When dense, continuous flow lines are required, or when the vector field is too sparse for the stochastic method to provide meaningful coverage.

## Background
Originating from the work of Crawfis and Max, this technique was developed as a 3D raster resampling method for volume rendering [S1, unofficial]. It was applied to time-varying wind fields and global climate simulations, where white density clouds represented cloudiness and colored vector icons represented wind velocity [S1, unofficial]. The method was presented at the 9th International Symposium on Flow Visualization in 2000 [S1, unofficial]. It builds on the concept of embedding vector icons directly into the splat footprint to indicate flow direction and magnitude [S1, unofficial].

## Sources
[S1] Flow Visualization Techniques for CDF using Volume Rendering, https://www.osti.gov/servlets/purl/790542, fetched 2026-10-26 (unofficial)
