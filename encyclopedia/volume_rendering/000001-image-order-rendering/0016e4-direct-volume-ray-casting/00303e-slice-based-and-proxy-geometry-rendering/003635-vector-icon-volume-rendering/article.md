# Vector Icon Volume Rendering

Also called: Billboard vector icons, Vector Icon Splatting with Billboard Textures, Vector icons embedded in splat footprint, Volume Rendering with Embedded Vector Icons

## Overview
Vector Icon Volume Rendering is a technique for visualizing vector fields (such as wind or fluid flow) within a 3D volume by embedding small billboard textures containing vector icons into the splat footprints of a sheet-based volume renderer [S1, unofficial]. It solves the problem of conveying direction and magnitude in scalar volume renderings by stochastically placing oriented icons that create an illusion of coherent motion [S1, unofficial]. It differs from pure scalar volume rendering by adding directional glyphs and from line-based methods (like Line Bundles) by using discrete, billboarded icons rather than continuous streamlines [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A 3D scalar field (e.g., density/cloudiness) and a 3D vector field (e.g., wind velocity); a set of billboard textures containing vector icons; camera position and orientation.
*   **Outputs:** A 2D image where scalar volume data is composited with oriented vector icons [S1, unofficial].

**Data Structures:**
*   **Billboard Textures:** 2D textures containing pre-rendered vector icons. These may be static or part of a texture atlas used for animation [S1, unofficial].
*   **Proxy Geometry:** Quads or triangle fans representing slices of the volume, consistent with the parent method [S1, unofficial].

**Steps:**
1.  **Sheet Generation:** Divide the volume into parallel slicing planes oriented relative to the view or a fixed axis [S1, unofficial].
2.  **Grid Sampling:** On each sheet, establish a regular grid of sampling points [S1, unofficial].
3.  **Splat Placement:** For each grid point, place an image reconstruction operator (splat) [S1, unofficial].
4.  **Stochastic Icon Selection:** Apply a stochastic heuristic to determine if a vector icon should be included in the current splat [S1, unofficial].
5.  **Icon Placement and Offset:** If an icon is selected, randomly offset its position from the center of the splat footprint [S1, unofficial].
6.  **Billboard Orientation:** Orient the billboard image such that it is perpendicular to the viewing ray and parallel to the projected vector direction at the splat's center point [S1, unofficial].
7.  **Icon Scaling:** Select the icon size based on the flow direction relative to the viewer:
    *   If flow is parallel to the screen plane, use the largest icon [S1, unofficial].
    *   If flow is parallel to the viewer’s line of sight, reduce the icon to a small dot [S1, unofficial].
8.  **Color Mapping:** Assign color to the vector icons, potentially representing scalar attributes such as altitude [S1, unofficial].
9.  **Compositing:** Render the discrete set of billboard images with the scalar volume data. Composite the fragments back-to-front using alpha blending [S1, unofficial].
10. **Animation (Optional):** To alleviate sparse representation and create motion, cycle through a set of textures where icons move across the texture, fading in and out, at rendering time [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_vector_icon_volume(volume_data, vector_data, camera, billboard_textures):
    # 1. Generate sheets parallel to image plane or fixed axis
    sheets = generate_sheets(volume_data.bounding_box)
    
    # 2. Sort sheets back-to-front relative to camera
    sheets.sort(key=lambda s: distance_to_camera(s.center, camera.position))
    
    for sheet in sheets:
        # 3. Regular grid sampling on the current sheet
        grid_points = generate_regular_grid(sheet)
        
        for point in grid_points:
            # 4. Stochastic heuristic to determine if splat includes a cylinder/icon
            if stochastic_heuristic(point):
                # 5. Randomly offset the cylinder/icon from the center
                offset = random_offset()
                icon_position = point.center + offset
                
                # 6. Get vector at this point
                vector = vector_data.sample(point.position)
                
                # 7. Determine billboard orientation
                # Perpendicular to viewing ray, parallel to projected vector direction
                view_direction = camera.position - point.position
                projected_vector = project_vector(vector, view_direction)
                billboard_rotation = compute_rotation(view_direction, projected_vector)
                
                # 8. Select icon size based on flow direction
                # If flow parallel to screen -> largest icon
                # If flow parallel to line of sight -> small dot
                icon_scale = compute_icon_scale(vector, view_direction)
                
                # 9. Select color (e.g., altitude)
                color = map_altitude_to_color(point.position.y)
                
                # 10. Render billboard
                render_billboard(
                    position=icon_position,
                    rotation=billboard_rotation,
                    scale=icon_scale,
                    texture=billboard_textures,
                    color=color
                )
                
                # 11. Add volume integral for scalar field at this pixel
                scalar_value = volume_data.sample(point.position)
                render_scalar_splat(point.position, scalar_value)
```

**Parameters:**
*   **Stochastic Heuristic Threshold:** Controls the density of icons. Not given in the sources.
*   **Icon Size Range:** Minimum (dot) to maximum (full icon). Not given in the sources.
*   **Texture Cycling Rate:** Speed at which icon textures fade in/out for motion illusion. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Sparse Representation:** The discrete nature of icons can make flow difficult to understand. Mitigated by texture cycling [S1, unofficial].
*   **Direction Ambiguity:** Icons parallel to the line of sight reduce to dots, losing directional information [S1, unofficial].
*   **Stochastic Noise:** Random placement may lead to uneven distribution if not carefully tuned [S1, unofficial].

## Subtopics

### Billboard Vector Icons
Uses pre-rendered 2D textures of icons that always face the camera. Best for clear directional cues in dense flows. [Vector Icon Volume Rendering](6226-vector-icon-volume-rendering.md)

### Vector Icon Splatting with Billboard Textures
Emphasizes the integration of icons into the splat footprint. Best for combining scalar volume rendering with vector direction. [Vector Icon Volume Rendering](6226-vector-icon-volume-rendering.md)

### Vector Icons Embedded in Splat Footprint
Focuses on the geometric embedding of icons within the reconstruction kernel. Best for precise control over icon placement relative to data points. [Vector Icon Volume Rendering](6226-vector-icon-volume-rendering.md)

### Volume Rendering with Embedded Vector Icons
General term for the technique. Best for general-purpose flow visualization in volume data. [Vector Icon Volume Rendering](6226-vector-icon-volume-rendering.md)

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of grid points and splats. Stochastic checks and texture lookups add constant overhead per splat.
*   **Memory Complexity:** Low, as it relies on standard volume textures and a small set of billboard textures.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Scales well with hardware rasterization. Performance depends on the number of billboards and alpha blending overhead.
*   **Strengths:** Intuitive representation of flow direction; easy to merge with scalar volume rendering; creates illusion of motion through animation.
*   **Limitations:** Sparse representation can obscure flow; direction ambiguity when flow is parallel to view; stochastic placement may appear noisy.
*   **When to Use:** When visualizing vector fields (wind, fluid) within a volume where direction is critical and scalar context (density) is also needed.
*   **When Not to Use:** When precise streamline continuity is required (use Line Bundles or Streamlines); when flow is nearly parallel to the view direction (icons become dots).

## Background
Originated with Crawfis and Max, who developed a 3D raster resampling technique for volume rendering [S1, unofficial]. They modified the volume integral to include tiny cylinders within splats, using stochastic placement to create the illusion of coherent flow in time-varying wind fields [S1, unofficial]. Further refinements embedded vector icons directly into the splat footprint using billboard images, allowing for size variation based on flow direction relative to the viewer [S1, unofficial]. The technique was applied to global climate simulations, merging white density clouds (cloudiness) with colored vector icons (wind velocity) [S1, unofficial].

## Sources
[S1] Flow Visualization Techniques for CDF using Volume Rendering, https://www.osti.gov/servlets/purl/790542, fetched 2026-10-26 (unofficial)
