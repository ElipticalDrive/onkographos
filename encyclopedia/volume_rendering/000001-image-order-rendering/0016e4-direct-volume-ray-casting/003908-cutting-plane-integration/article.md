# Cutting Plane Integration

Also called: CuttingPlaneRender, Interactive Cutting Plane

## Overview
Cutting Plane Integration is an interactive clipping technique that renders a semi-transparent plane mesh and normal arrow, dynamically updating raycasting uniforms to slice the volume and reveal internal structures [S1, unofficial]. It solves the problem where full opacity is obtained before reaching internal structures, making them difficult to see [S1, unofficial]. It sets itself apart from standard volume rendering by restricting the integration domain to a user-defined subset, which can be geometric or volumetric, rather than the entire volume extent [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume dataset (typically stored as a 1D array of normalized floats or a 3D texture), camera parameters, and user-defined plane parameters (position, rotation, orientation).
*   **Output:** A 2D image where ray integration is terminated or modulated by the cutting plane.

**Data Structures:**
The volume data is often loaded into a structure containing dimensions and voxel values [S1, unofficial]:
*   `width`, `height`, `depth`: Integer dimensions.
*   `voxels`: A `Float32Array` of size `width * height * depth`, with values normalized to $[0, 1]$ from raw integer data (e.g., divided by 4095.0) [S1, unofficial].
*   The volume is typically uploaded to the GPU as a `Data3DTexture` with `RedFormat` and `FloatType`, using linear filtering [S1, unofficial].

**Parameters:**
*   `planeHeight`: A normalized float in $[0, 1]$ mapping to the Z-axis position [S1, unofficial].
*   `planeRotX`, `planeRotY`: Rotation angles in degrees around the X and Y axes [S1, unofficial].
*   `renderAbove`: A boolean flag that flips the plane 180° around the Y-axis when true [S1, unofficial].

**Steps:**
1.  **Plane Position Calculation:** Map the normalized `planeHeight` to a world Z position. The formula used in the reference implementation is:
    $$ z_{plane} = 200 - 400 \times planeHeight $$
    This maps `planeHeight` 0 to $z=200$ and 1 to $z=-200$ [S1, unofficial].
2.  **Plane Rotation:** Convert `planeRotY` from degrees to radians. Apply a $\pi$ offset if `renderAbove` is true to flip the plane orientation:
    $$ \theta_{y} = \begin{cases} planeRotY \times \frac{\pi}{180} + \pi & \text{if } renderAbove \\ planeRotY \times \frac{\pi}{180} & \text{otherwise} \end{cases} $$
    The plane is rotated around the X and Y axes using these angles [S1, unofficial].
3.  **Normal Vector Visualization:** A normal vector arrow is created to visualize the plane's orientation. The arrow originates at $(0,0,0)$ and initially points along the Z-axis $(0,0,1)$. It is rendered with depth testing disabled to ensure it appears on top [S1, unofficial].
4.  **Raycasting Integration:** During raycasting, the cutting plane acts as a boundary. Rays are integrated against the volume texture. The plane's world position and orientation are passed as uniforms to the shader. The shader determines if a sample point is within the integration domain (e.g., "above" or "below" the plane) and accumulates color/opacity accordingly [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def update_cutting_plane(plane_height, plane_rot_x, plane_rot_y, render_above):
    # Map normalized height to world Z position
    # Formula from [S1, unofficial]: z = 200 - 400 * planeHeight
    plane_z = 200.0 - 400.0 * plane_height
    
    # Convert degrees to radians
    rot_y_rad = math.radians(plane_rot_y)
    rot_x_rad = math.radians(plane_rot_x)
    
    # Apply rotation offset if rendering above
    if render_above:
        rot_y_rad += math.pi
        
    # Create plane transformation matrix
    # Note: Specific matrix construction depends on the graphics API (e.g., THREE.js)
    # The plane is positioned at (0, 0, plane_z) and rotated by rot_x and rot_y
    plane_matrix = create_rotation_matrix(rot_x_rad, rot_y_rad)
    plane_matrix.translate(0, 0, plane_z)
    
    return plane_matrix

def render_volume(volume_texture, camera, plane_matrix):
    # For each pixel, cast a ray
    for y in range(height):
        for x in range(width):
            ray = camera.get_ray(x, y)
            
            # Intersect ray with volume bounding box
            t_min, t_max = ray.intersect_bounding_box()
            
            alpha_accum = 0.0
            color_accum = (0.0, 0.0, 0.0)
            
            t = t_min
            while t < t_max:
                sample_pos = ray.origin + ray.direction * t
                
                # Check if sample is within the cutting plane domain
                # This requires transforming sample_pos to plane space and checking
                # the dot product with the plane normal or comparing against plane_z
                if is_within_integration_domain(sample_pos, plane_matrix):
                    # Sample the volume texture
                    density = volume_texture.sample(sample_pos)
                    
                    # Apply transfer function to get color and opacity
                    color, opacity = transfer_function(density)
                    
                    # Front-to-back compositing
                    alpha_accum += opacity * (1.0 - alpha_accum)
                    color_accum += color * opacity * (1.0 - alpha_accum)
                    
                    if alpha_accum >= 1.0:
                        break
                
                t += step_size
                
            final_color = color_accum
            set_pixel(x, y, final_color)
```

## Subtopics

### [Region of Interest (ROI) Rendering](10737-region-of-interest-roi-rendering.md)
Region of Interest (ROI) rendering is a technique to visualize internal features by defining a specific region and rendering only that portion of the dataset. It solves the problem where full opacity is obtained before reaching internal structures, making them difficult to see. It sets itself apart from standard volume rendering by restricting the integration domain to a user-defined subset, which can be geometric or volumetric, rather than the entire volume extent.

## Operating characteristics
**Time and Memory Complexity:**
The time complexity is $O(N \cdot S)$, where $N$ is the number of pixels and $S$ is the number of samples per ray within the integration domain. The memory complexity is dominated by the volume texture storage, $O(W \cdot H \cdot D)$, where $W, H, D$ are the volume dimensions [S1, unofficial].

**Measured Performance:**
Performance depends on the resolution of the volume and the efficiency of the GPU-based raycasting. The reference implementation uses WebGL/Three.js, which may be slower than native OpenGL/Vulkan implementations [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Interactive exploration of internal structures; simple to implement as a uniform update; effective for revealing hidden features.
*   **Limitations:** The plane is a simple geometric primitive; may not handle complex internal boundaries as well as ROI rendering with volumetric masks [S1, unofficial].

**When to Use:**
Use when interactive slicing is required to inspect internal structures of a volume.

**When Not to Use:**
Use when complex, non-planar internal boundaries are needed, or when high-performance real-time rendering is critical and WebGL overhead is a bottleneck.

## Background
Cutting Plane Integration is a technique used in interactive volume visualization. The reference implementation is part of a student project for volume rendering visualization [S1, unofficial]. It builds on the principles of direct volume ray casting [G].

## Sources
[S1] SubairProject1/Volume-Rendering-Visualization-Project, https://github.com/SubairProject1/Volume-Rendering-Visualization-Project, fetched 2026-10-26 (unofficial)
