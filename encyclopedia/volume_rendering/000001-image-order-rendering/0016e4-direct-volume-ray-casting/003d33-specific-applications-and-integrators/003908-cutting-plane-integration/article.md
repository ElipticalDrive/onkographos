# Cutting Plane Integration

Also called: CuttingPlaneRender, Interactive Cutting Plane

## Overview
Cutting Plane Integration is an interactive volume rendering technique that dynamically slices a 3D dataset using a semi-transparent geometric plane to reveal internal structures [S1, unofficial]. It solves the problem of occlusion in dense volumes where full opacity prevents visibility of deeper features by restricting the integration domain to one side of a user-defined plane [S1, unofficial]. It sets itself apart from standard ray casting by intercepting the compositing stage to discard samples based on their position relative to the plane's normal and origin, rather than integrating through the entire volume extent.

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume texture (scalar field), camera parameters, and cutting plane parameters (position, rotation, orientation flag).
*   **Output:** A 2D image where voxels on the "cut" side are composited, and voxels on the "other" side are discarded or rendered with zero opacity.

**Data Structures:**
The cutting plane is represented by a geometric primitive (e.g., `PlaneGeometry`) and a visual helper (e.g., `ArrowHelper`) for the normal vector [S1, unofficial].
*   **Plane Position:** Defined by a normalized height parameter $h \in [0, 1]$ mapped to world Z-coordinate $z_p$.
*   **Plane Rotation:** Defined by angles $\theta_x, \theta_y$ in degrees.
*   **Orientation:** A boolean flag `renderAbove` determines the integration direction relative to the plane normal.

**Steps:**
1.  **Initialize Plane:** Create a plane mesh and an arrow helper representing the normal vector (initially along the Z-axis) [S1, unofficial].
2.  **Update Position:** Map the normalized height $h$ to the world Z-position $z_p$ using the formula:
    $$ z_p = 200 - 400 \cdot h $$
    [S1, unofficial]
3.  **Update Rotation:** Convert rotation angles $\theta_x, \theta_y$ from degrees to radians. Apply rotation to the plane's orientation. If `renderAbove` is true, apply an additional 180° ($\pi$ radians) offset to the Y-rotation to flip the plane's effective normal direction [S1, unofficial].
4.  **Ray Casting Loop:** For each pixel, cast a ray through the volume.
5.  **Plane Intersection Check:** For each sample point $P$ along the ray, determine its position relative to the cutting plane.
6.  **Conditional Compositing:**
    *   If the sample point $P$ is on the "visible" side of the plane (determined by the dot product of $(P - \text{planeOrigin})$ and the plane normal, adjusted by `renderAbove`), accumulate the sample's color and opacity using standard front-to-back or back-to-front compositing.
    *   If the sample point $P$ is on the "hidden" side, discard the sample (contribute zero opacity/color).
7.  **Render Helper:** Render the semi-transparent plane mesh and the normal arrow on top of the volume, disabling depth testing for the arrow to ensure visibility [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cutting_plane_render(volume, camera, plane_height, plane_rot_x, plane_rot_y, render_above):
    # 1. Calculate plane position
    # Map normalized height [0, 1] to world Z [200, -200]
    plane_z = 200.0 - 400.0 * plane_height
    plane_origin = Vector3(0, 0, plane_z)
    
    # 2. Calculate plane normal
    # Start with Z-axis normal
    normal = Vector3(0, 0, 1)
    
    # Apply rotations (convert degrees to radians)
    theta_x = math.radians(plane_rot_x)
    theta_y = math.radians(plane_rot_y)
    
    # Rotate around X axis
    normal = rotate_x(normal, theta_x)
    # Rotate around Y axis
    normal = rotate_y(normal, theta_y)
    
    # If render_above is true, flip the normal by 180 degrees around Y
    if render_above:
        normal = rotate_y(normal, math.pi)
    
    # 3. Ray casting loop
    for each pixel (u, v):
        ray = camera.get_ray(u, v)
        color = Vector3(0, 0, 0)
        alpha = 0.0
        
        # Sample along the ray
        for t in range(ray_start, ray_end, step_size):
            point = ray.origin + ray.direction * t
            voxel_value = volume.sample(point)
            
            # 4. Check if point is on the visible side of the plane
            # Vector from plane origin to point
            vec_to_point = point - plane_origin
            
            # Dot product with normal determines side
            # Positive dot product means point is in direction of normal
            is_visible = dot(vec_to_point, normal) > 0
            
            if is_visible:
                # Standard compositing
                sample_color = transfer_function(voxel_value)
                sample_alpha = opacity_function(voxel_value)
                
                # Front-to-back compositing
                color = color + sample_color * sample_alpha * (1.0 - alpha)
                alpha = alpha + sample_alpha * (1.0 - alpha)
                
                # Early ray termination if fully opaque
                if alpha >= 1.0:
                    break
            else:
                # Discard sample on the hidden side
                pass
                
        image.set_pixel(u, v, color)

    # 5. Render plane helper (semi-transparent) and normal arrow (no depth test)
    render_plane(plane_origin, normal, alpha=0.3)
    render_arrow(plane_origin, normal, color=red, depth_test=False)
```

**Parameters:**
*   `planeHeight`: Normalized float $[0, 1]$. Typical value: $0.5$ (center). [S1, unofficial]
*   `planeRotX`, `planeRotY`: Rotation angles in degrees. Typical value: $0$. [S1, unofficial]
*   `renderAbove`: Boolean. Determines if the integration occurs in the direction of the normal or opposite. [S1, unofficial]

**Edge Cases:**
*   **Precision:** Floating point comparisons for plane intersection may cause artifacts if the ray step size is large relative to the plane thickness. [G]
*   **Boundaries:** The plane position formula assumes a specific world coordinate system (Z-axis centered at 0, range -200 to 200). Changes to volume bounds require adjusting the formula. [S1, unofficial]

## Subtopics

### [Region of Interest (ROI) Rendering](10737-region-of-interest-roi-rendering.md)
Restricts integration to a user-defined geometric or volumetric subset. Choose when you need to isolate specific anatomical or structural regions rather than slicing the entire volume.

## Operating characteristics
*   **Time Complexity:** $O(N)$ per pixel, where $N$ is the number of samples along the ray. However, the effective $N$ is reduced by the clipping plane, potentially improving performance if the visible region is small.
*   **Memory Complexity:** $O(1)$ auxiliary memory for the plane parameters. The volume data dominates memory usage.
*   **Strengths:** Interactive exploration of internal structures; simple implementation; low computational overhead compared to multi-pass techniques.
*   **Limitations:** Only reveals structures on one side of the plane; requires manual adjustment of plane position and orientation; does not handle complex occlusion within the visible region better than standard ray casting.
*   **When to use:** Interactive visualization, educational tools, debugging volume data.
*   **When not to use:** Production-quality medical imaging requiring precise segmentation; scenes with complex internal occlusion where the plane might hide relevant structures.

## Background
Cutting Plane Integration is a technique commonly implemented in educational and open-source volume rendering projects to demonstrate interactive clipping. It is not associated with a single seminal peer-reviewed paper but is a standard feature in many volume rendering toolkits and tutorials. The specific implementation details (Z-axis mapping, arrow helpers) are derived from open-source GitHub repositories [S1, unofficial].

## Sources
[S1] SubairProject1/Volume-Rendering-Visualization-Project, https://github.com/SubairProject1/Volume-Rendering-Visualization-Project, fetched 2026-10-26 (unofficial)
