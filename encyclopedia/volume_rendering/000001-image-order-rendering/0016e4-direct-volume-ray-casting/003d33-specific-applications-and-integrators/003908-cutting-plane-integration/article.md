# Cutting Plane Integration

Also called: CuttingPlaneRender, Interactive Cutting Plane

## Overview
Cutting Plane Integration is an interactive clipping technique that renders a semi-transparent plane mesh and normal arrow to dynamically slice a volume, revealing internal structures by updating raycasting uniforms [S1, unofficial]. It differs from sibling methods like [Customization Volume Integration](5291-customization-volume-integration.md) or [TVAM Integrator](6814-tvam-integrator.md) by explicitly visualizing the clipping boundary as a geometric entity within the scene, rather than merely excluding data or modulating transfer functions [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume dataset (typically stored as a normalized float array), camera parameters, and user interaction controls for plane position and orientation.
*   **Output:** A 2D image where the volume is rendered with a visible cut plane intersecting the data.

**Data Structures:**
*   **Volume Data:** A `Float32Array` of voxel values normalized to $[0, 1]$. The source structure maps a `uint16Array` where the first three elements are width, height, and depth, and subsequent elements are voxel intensities scaled by $1/4095.0$ [S1, unofficial].
*   **Plane Geometry:** A mesh representing the cutting plane, positioned in world space.
*   **Normal Arrow:** A geometric helper (e.g., `ArrowHelper`) indicating the plane's normal direction, rendered with depth testing disabled to ensure visibility [S1, unofficial].

**Parameters:**
*   `planeHeight`: A normalized float in $[0, 1]$ mapping to the Z-axis position [S1, unofficial].
*   `planeRotX`, `planeRotY`: Rotation angles in degrees around the X and Y axes [S1, unofficial].
*   `renderAbove`: A boolean flag that flips the plane 180° around the Y-axis when true [S1, unofficial].

**Algorithm Steps:**
1.  **Map Height to Position:** Calculate the world Z-position of the plane using the normalized height. The formula maps `planeHeight` from $[0, 1]$ to a Z-range of $[200, -200]$ (assuming a center at 0 and range of 400):
    $$ z_{plane} = 200 - 400 \times \text{planeHeight} $$
    Set the plane mesh's Z-position to $z_{plane}$ [S1, unofficial].
2.  **Apply Rotation:** Convert `planeRotY` from degrees to radians. If `renderAbove` is true, add $\pi$ to the rotation angle to flip the plane orientation:
    $$ \theta_{final} = \text{planeRotY}_{rad} + (\text{renderAbove} ? \pi : 0) $$
    Apply the rotation to the plane mesh [S1, unofficial].
3.  **Render Plane and Arrow:** Render the semi-transparent plane mesh. Render the normal arrow originating from the plane center, pointing along the local Z-axis (initially), with depth testing disabled (`depthTest = false`, `depthWrite = false`) to ensure it appears on top [S1, unofficial].
4.  **Raycasting with Clipping:** During the volume ray casting loop (as defined in [Direct volume ray casting](19-direct-volume-ray-casting.md)), use the plane's world-space position and orientation to clip the ray. Samples are only integrated if they lie on the correct side of the plane (or within a defined thickness, if applicable). The plane's uniform is updated each frame to reflect the current `planeHeight` and rotation [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def update_cutting_plane(plane_mesh, normal_arrow, plane_height, plane_rot_x, plane_rot_y, render_above):
    # Map normalized height to world Z position
    # Range: 200 (top) to -200 (bottom)
    z_position = 200.0 - 400.0 * plane_height
    plane_mesh.position.z = z_position

    # Convert degrees to radians
    rot_y_rad = math.radians(plane_rot_y)
    
    # Apply rotation offset if rendering above
    if render_above:
        rot_y_rad += math.pi
        
    # Apply rotations to the plane mesh
    plane_mesh.rotation.x = math.radians(plane_rot_x)
    plane_mesh.rotation.y = rot_y_rad

    # Update normal arrow to reflect plane orientation
    # The arrow originates at the plane center and points along the local Z-axis
    normal_arrow.position.copy(plane_mesh.position)
    # Note: The arrow's direction is typically handled by the graphics API's 
    # coordinate system transformation relative to the plane mesh.
    # Ensure it renders on top
    normal_arrow.line.material.depth_test = False
    normal_arrow.line.material.depth_write = False

def render_volume_with_cutting_plane(volume_texture, camera, plane_mesh, transfer_function):
    # Initialize raycasting loop as per Direct Volume Ray Casting
    for each pixel (x, y):
        ray = camera.generate_ray(x, y)
        color = vec3(0.0)
        alpha = 0.0
        
        # Clip ray against the cutting plane
        # Calculate intersection of ray with plane
        plane_normal = plane_mesh.get_world_normal() # Vector pointing in local Z direction
        plane_origin = plane_mesh.position
        
        # Ray-plane intersection: t = (plane_origin - ray_origin) . normal / (ray_direction . normal)
        denom = dot(ray.direction, plane_normal)
        if abs(denom) > 1e-6:
            t = dot(plane_origin - ray.origin, plane_normal) / denom
            if t > 0:
                intersection_point = ray.origin + ray.direction * t
                # Determine which side of the plane is visible
                # Assuming we want to render the side "behind" the normal or "in front"
                # This logic depends on the specific 'renderAbove' or clipping convention
                # For this implementation, we clip the ray segment to be within the volume
                # and potentially stop integration at the plane if it's a hard cut
                # Or integrate only up to the plane if it's a "slice" view
                
                # Simplified: Integrate along the ray, but mask contributions based on plane
                for sample_pos along ray:
                    if is_inside_volume(sample_pos):
                        # Check if sample is on the visible side of the cutting plane
                        # Vector from plane origin to sample
                        vec_to_sample = sample_pos - plane_origin
                        # Project onto normal
                        dist = dot(vec_to_sample, plane_normal)
                        
                        # Example: Render only if dist < 0 (behind the plane normal)
                        # Adjust inequality based on desired visibility
                        if dist < 0: 
                            density = volume_texture.sample(sample_pos)
                            color_val, alpha_val = transfer_function(density)
                            color, alpha = composite(color, alpha, color_val, alpha_val)
        else:
            # Ray parallel to plane, no intersection
            pass
            
    return color
```

## Subtopics

### Region of Interest (ROI) Rendering
A technique to visualize internal features by defining a region of interest and rendering only that portion of the dataset [S2, book]. It differs from basic cutting planes by supporting complex geometric definitions beyond a single plane [S2, book].
*   **Definition:** Define a region using camera near/far clipping planes, six orthographic clipping planes (rectangular subvolume), arbitrarily oriented half-space clipping planes, closed geometric objects, or an auxiliary binary mask volume [S2, book].
*   **Implementation:**
    *   **Image-order:** Clip the ray against all geometric region definitions as a preprocessing step. Evaluate the ray function only along segments within the ROI. Consult mask values at each sample to include/exclude contributions [S2, book].
    *   **Object-order:** Determine for each sample if it is within the ROI before incorporating its contribution. Use hardware clipping planes if utilizing texture mapping [S2, book].

## Operating characteristics
*   **Time Complexity:** $O(N)$ per pixel, where $N$ is the number of samples along the ray within the volume and ROI. The clipping check adds negligible overhead per sample [S2, book].
*   **Memory Complexity:** $O(V)$ for the volume data, plus $O(1)$ for the plane geometry and uniforms [S1, unofficial].
*   **Strengths:** Interactive performance for exploring internal structures; visual feedback via the plane mesh and arrow [S1, unofficial].
*   **Limitations:** Limited to planar or simple geometric clipping unless extended with ROI techniques [S2, book]. Precision issues may arise if the plane is parallel to the ray direction (denominator near zero) [S1, unofficial].
*   **When to use:** Interactive exploration of volumetric data where the user needs to see the cutting boundary.
*   **When not to use:** When complex, non-planar regions of interest are required (use ROI Rendering instead) [S2, book].

## Background
Cutting Plane Integration is a standard technique in interactive volume visualization, often implemented in educational or lightweight visualization projects [S1, unofficial]. It builds on the foundational ray casting loop defined in [Direct volume ray casting](19-direct-volume-ray-casting.md). The concept of using clipping planes to define regions of interest is discussed in advanced computer graphics literature, such as the VTK Book [S2, book].

## Sources
[S1] SubairProject1/Volume-Rendering-Visualization-Project, https://github.com/SubairProject1/Volume-Rendering-Visualization-Project, fetched 2026-10-26 (unofficial)
[S2] Chapter 7 - Advanced Computer Graphics - VTK, https://examples.vtk.org/site/VTKBook/07Chapter7/, fetched 2026-10-29 (book)
