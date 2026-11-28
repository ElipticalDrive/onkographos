# Vector Icon Volume Rendering

Also called: Billboard vector icons, Vector Icon Splatting with Billboard Textures, Vector icons embedded in splat footprint, Volume Rendering with Embedded Vector Icons

## Overview
Vector Icon Volume Rendering is a 3D raster resampling technique that integrates vector field visualization into scalar volume rendering by embedding small billboard images containing vector icons within the splat footprint of the volume rendering sheets [S1, unofficial]. It solves the problem of visualizing flow direction and magnitude within a volumetric scalar field by creating an illusion of coherent flow through stochastic placement and orientation of these icons [S1, unofficial]. It sets itself apart from continuous integration methods by modifying the volume integral to include rendering of tiny cylinders within a small neighborhood of each sampling point, and differs from pure vector field visualization by merging the rendering of a scalar field with that of a vector field in a single compositing pass [S1, unofficial].

## How it works
The method builds on the slice-based rendering principles defined in [Slice-based and Proxy Geometry Rendering](6225-slice-based-and-proxy-geometry-rendering.md). It modifies the standard volume integral to include the rendering of tiny cylinders (vector icons) within a small neighborhood of each sampling point [S1, unofficial].

### Inputs and Outputs
*   **Inputs**:
    *   3D scalar field data (e.g., density clouds) [S1, unofficial].
    *   3D vector field data (e.g., wind velocity) [S1, unofficial].
    *   Grid spacing for sheet sampling [S1, unofficial].
    *   Stochastic threshold for icon placement [S1, unofficial].
    *   Icon size limits (maximum and minimum) [S1, unofficial].
    *   Color mapping parameters (e.g., altitude to color) [S1, unofficial].
    *   Time step for time-varying fields [S1, unofficial].
*   **Outputs**:
    *   A 2D image where the scalar volume is rendered via alpha blending, and vector flow is indicated by colored, oriented billboard icons [S1, unofficial].

### Data Structures
*   **Volume Texture**: 3D texture storing scalar values [S1, unofficial].
*   **Icon Texture Set**: A set of 2D textures containing pre-rendered vector icons that move across the texture and fade in/out to simulate motion [S1, unofficial].
*   **Billboard Geometry**: Implicitly defined by the splat footprint; each billboard image is oriented perpendicular to the viewing ray and parallel to the projected vector direction at the splat's center [S1, unofficial].

### Parameters
*   **Grid Spacing**: Regular grid sampling on the current sheet [S1, unofficial].
*   **Stochastic Threshold**: Heuristic determining whether a splat includes a cylinder/icon [S1, unofficial].
*   **Icon Size Scaling**: Maximum size when flow is parallel to the screen plane; minimum size (dot) when flow is parallel to the viewer’s line of sight [S1, unofficial].
*   **Color Mapping**: Scalar value (e.g., altitude) mapped to icon color [S1, unofficial].

### Steps
1.  Decompose the volume into sheets oriented parallel to the image plane [G].
2.  Sort the sheets in back-to-front order relative to the camera for correct alpha blending compositing [S1, unofficial].
3.  For each sheet, take a regular grid sampling of the vector field data [S1, unofficial].
4.  Place an image reconstruction operator (splat) over each grid point [S1, unofficial].
5.  Use a stochastic heuristic to determine if the current splat should include a vector icon (cylinder) [S1, unofficial].
6.  If included, randomly offset the icon from the splat center [S1, unofficial].
7.  Orient the billboard image such that it lies perpendicular to the viewing ray and parallel to the projected vector direction at the splat's center point [S1, unofficial].
8.  Scale the icon size based on the angle between the flow vector and the view direction: largest when parallel to the screen, reduced to a dot when parallel to the line of sight [S1, unofficial].
9.  Render the scalar volume for the sheet using standard volume rendering techniques (texture mapping, classification, blending) [G].
10. Add the volume integral for each pixel under both the splat and the cylinder [S1, unofficial].
11. Composite the results back-to-front [S1, unofficial].
12. For time-varying fields, produce a new rendering for each time step and animate in a movie loop [S1, unofficial].
13. Cycle through the icon texture set at rendering time, where icons move across the texture and fade in/out, to achieve the illusion of motion [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def render_vector_icon_volume(scalar_volume, vector_field, camera, time_step, grid_spacing, stochastic_threshold, max_icon_size, min_icon_size):
    # 1. Generate sheets parallel to image plane [G]
    sheets = generate_sheets_parallel_to_image_plane(scalar_volume.bounding_box, camera.view_direction)
    # 2. Sort sheets back-to-front
    sheets.sort(key=lambda s: math.dist(s.center, camera.position))
    
    frame_buffer = [0.0, 0.0, 0.0, 0.0]
    
    for sheet in sheets:
        # 9. Render scalar volume for sheet [G]
        sheet_color = render_scalar_sheet_standard(scalar_volume, sheet, camera)
        
        # 3. Regular grid sampling on the current sheet
        grid_points = sample_regular_grid_on_sheet(sheet, grid_spacing)
        
        icon_layer = []
        for point in grid_points:
            # 5. Stochastic heuristic
            if random.random() < stochastic_threshold:
                # 6. Random offset from center
                offset = (random.uniform(-0.5, 0.5), random.uniform(-0.5, 0.5))
                icon_pos = (point.x + offset[0], point.y + offset[1])
                
                # 7. Orientation: perpendicular to viewing ray, parallel to projected vector
                flow_vec = vector_field.sample(point)
                projected_dir = project_vector_to_screen(flow_vec, camera)
                billboard = create_billboard(icon_pos, projected_dir, camera.view_direction)
                
                # 8. Icon size scaling
                view_dir = normalize(camera.view_direction)
                flow_norm = normalize(flow_vec)
                angle = math.acos(max(-1.0, min(1.0, math.dot(flow_norm, view_dir))))
                if abs(angle - math.pi/2) < 0.1:
                    size = max_icon_size
                elif abs(angle) < 0.1 or abs(angle - math.pi) < 0.1:
                    size = min_icon_size
                else:
                    size = max_icon_size * math.sin(angle)
                billboard.size = size
                
                # 13. Texture cycling for motion illusion
                texture_frame = get_texture_frame(time_step, point, icon_layer)
                billboard.texture = texture_frame
                
                # Color by altitude
                altitude = scalar_volume.sample(point)
                billboard.color = map_altitude_to_color(altitude)
                
                icon_layer.append(billboard)
        
        # 10 & 11. Add volume integral and composite back-to-front
        sheet_color = alpha_blend_splats(sheet_color, icon_layer)
        frame_buffer = alpha_blend(frame_buffer, sheet_color)
        
    return frame_buffer

# Helper definitions
def generate_sheets_parallel_to_image_plane(bounding_box, view_direction):
    # [G] Standard slice generation from parent method
    return [Sheet(center=bounding_box.min + i * step, normal=view_direction) for i in range(num_slices)]

def sample_regular_grid_on_sheet(sheet, spacing):
    return [Point(x, y, sheet.center) for x in range(sheet.min_x, sheet.max_x, spacing) for y in range(sheet.min_y, sheet.max_y, spacing)]

def project_vector_to_screen(vec, camera):
    # [G] Standard projection to screen space
    return vec - math.dot(vec, normalize(camera.view_direction)) * normalize(camera.view_direction)

def create_billboard(position, direction, view_direction):
    return {"position": position, "direction": direction, "view_direction": view_direction, "size": 1.0, "texture": None, "color": (1.0, 1.0, 1.0, 1.0)}

def get_texture_frame(time_step, point, icon_layer):
    # [G] Cycle through texture set based on time and point index
    return icon_layer.index(point) % len(texture_set)

def map_altitude_to_color(altitude):
    # [G] Standard transfer function mapping
    return (altitude, 1.0 - altitude, 0.0, 1.0)

def alpha_blend_splats(base_color, splats):
    # [G] Standard alpha compositing
    color = list(base_color)
    for s in splats:
        color = blend_two_colors(color, s.color)
    return color

def alpha_blend(a, b):
    return blend_two_colors(a, b)

def blend_two_colors(c1, c2):
    return (c1[0]*c1[3] + c2[0]*c2[3]*(1-c1[3]), c1[1]*c1[3] + c2[1]*c2[3]*(1-c1[3]), c1[2]*c1[3] + c2[2]*c2[3]*(1-c1[3]), c1[3] + c2[3]*(1-c1[3]))

def normalize(v):
    mag = math.sqrt(sum(x*x for x in v))
    return [x/mag for x in v] if mag > 0 else v
```

### Edge Cases and Pitfalls
*   **Sparse Representation**: The discrete and sparse nature of the icons can make the flow difficult to understand [S1, unofficial]. This is mitigated by the texture cycling technique to create an illusion of motion [S1, unofficial].
*   **View Alignment**: Icon size must be carefully scaled to avoid occlusion or invisibility when the flow is parallel to the view direction [S1, unofficial].

## Subtopics

### Billboard Vector Icons
Uses discrete billboard images to represent flow [S1, unofficial]. Sets itself apart by being purely discrete and stochastic, unlike continuous vector fields [S1, unofficial]. Choose when visual clarity of individual flow elements is preferred over dense field representation [G]. [Link](6225-slice-based-and-proxy-geometry-rendering.md)

### Vector Icon Splatting with Billboard Textures
Enhances the basic billboard method by using animated textures to simulate motion [S1, unofficial]. Choose when animation is required to convey flow direction in a sparse representation [G]. [Link](6225-slice-based-and-proxy-geometry-rendering.md)

### Vector Icons Embedded in Splat Footprint
Integrates icons directly into the volume rendering splat footprint [S1, unofficial]. Sets itself apart by merging scalar and vector rendering in a single pass per splat [S1, unofficial]. Choose for tight integration of scalar context and vector direction [G]. [Link](6225-slice-based-and-proxy-geometry-rendering.md)

## Operating characteristics
*   **Time Complexity**: Not given in the sources.
*   **Memory Complexity**: Not given in the sources.
*   **Performance**: Efficient on GPU due to rasterization of billboards. Performance scales with the number of sheets and grid resolution [S1, unofficial].
*   **Strengths**: Intuitive representation of flow direction; integrates well with scalar volume rendering; creates illusion of motion through animation [S1, unofficial].
*   **Limitations**: Sparse representation can obscure flow; stochastic placement may lead to visual artifacts; difficult to understand flow in complex regions due to discrete nature [S1, unofficial].
*   **When to Use**: Time-varying wind or fluid flow visualization where direction and magnitude are critical, and scalar context (e.g., clouds) is also needed [S1, unofficial].
*   **When Not to Use**: When precise quantitative flow analysis is required, or when the flow is dense and continuous (use streamlines or pathlines instead) [S1, unofficial].

## Background
The technique was developed by R. Crawfis, H. Shen, and N. Max [S1, unofficial]. It is described in the paper "Flow Visualization Techniques for CDF using Volume Rendering" presented at the 9th International Symposium on Flow Visualization, Heriot-Watt University, Edinburgh, 2000 [S1, unofficial]. The authors previously developed related techniques such as Line Bundles [S1, unofficial]. The method was applied to global climate simulation data, merging white density clouds (cloudiness) with colored vector icons (wind velocity) [S1, unofficial].

## Sources
[S1] Flow Visualization Techniques for CDF using Volume Rendering, https://www.osti.gov/servlets/purl/790542, fetched 2026-10-26 (unofficial)
