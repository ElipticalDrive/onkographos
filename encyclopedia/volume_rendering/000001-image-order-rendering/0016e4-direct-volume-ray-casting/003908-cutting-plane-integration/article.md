# Cutting Plane Integration

Also called: CuttingPlaneRender, Interactive Cutting Plane

## Overview
Cutting Plane Integration is an interactive volume rendering technique that dynamically slices a 3D dataset using a semi-transparent geometric plane to reveal internal structures obscured by foreground opacity [S1, unofficial]. It solves the problem where standard front-to-back compositing accumulates full opacity before reaching deep internal features, by restricting the integration domain to one side of a user-defined clipping plane [S1, unofficial]. It sets itself apart from standard volume rendering by introducing a dynamic, user-controllable boundary that truncates the ray integration path, distinct from static clipping regions or multi-pass post-processing techniques [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D scalar volume dataset $V$ (typically stored as a 1D array of normalized floats), camera parameters (origin $\mathbf{o}$, direction $\mathbf{d}$), and cutting plane parameters (position, orientation, and side).
*   **Output:** A 2D image where pixels are composited using the front-to-back equation, but only for samples lying within the valid half-space defined by the cutting plane.

**Data Structures:**
The volume data is typically loaded into a 3D texture or a 1D `Float32Array` with dimensions $(W, H, D)$. The cutting plane is represented by a geometric mesh (e.g., a `PlaneGeometry`) and a visual indicator (e.g., an `ArrowHelper` for the normal vector) [S1, unofficial].

**Parameters:**
*   `planeHeight`: A normalized float in $[0, 1]$ mapping to the Z-axis position [S1, unofficial].
*   `planeRotX`, `planeRotY`: Rotation angles in degrees around the X and Y axes [S1, unofficial].
*   `renderAbove`: A boolean flag that flips the plane 180° around the Y-axis, effectively swapping the "valid" side of the cut [S1, unofficial].

**Steps:**
1.  **Plane Position Calculation:** Map the normalized `planeHeight` to a world Z-coordinate. Based on the source implementation, the position is calculated as:
    $$ z_{plane} = 200 - 400 \cdot \text{planeHeight} $$
    This maps the range $[0, 1]$ to $[200, -200]$ [S1, unofficial].
2.  **Plane Orientation:** Convert `planeRotY` from degrees to radians. If `renderAbove` is true, add $\pi$ to the rotation angle to flip the plane's normal direction [S1, unofficial].
3.  **Ray Casting:** For each pixel, define the ray $r(t) = \mathbf{o} + t\mathbf{d}$ [G].
4.  **Clipping Check:** For each sample point $\mathbf{p}_i$ along the ray, determine if it lies on the "visible" side of the cutting plane. Let $\mathbf{n}$ be the plane normal and $\mathbf{p}_{plane}$ be a point on the plane (e.g., the center). The sample is valid if:
    $$ (\mathbf{p}_i - \mathbf{p}_{plane}) \cdot \mathbf{n} \ge 0 $$
    (Assuming the normal points towards the visible side; this depends on the `renderAbove` logic) [S1, unofficial].
5.  **Compositing:** Perform standard front-to-back compositing [Parent] only for samples that pass the clipping check. Samples on the invalid side are skipped (treated as transparent/background) [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_cutting_plane(volume, camera, plane_params):
    """
    volume: 3D array of floats [W, H, D]
    camera: object with origin o, direction d, and image dimensions
    plane_params: dict with height (0-1), rot_x, rot_y, render_above (bool)
    """
    # 1. Calculate plane position and normal
    # Map normalized height to world Z
    plane_z = 200.0 - 400.0 * plane_params['height']
    plane_center = Vector3(0, 0, plane_z)
    
    # Calculate normal based on rotation
    rot_y_rad = math.radians(plane_params['rot_y'])
    if plane_params['render_above']:
        rot_y_rad += math.pi
        
    # Simple rotation around Y axis for normal (assuming initial normal is Z)
    # Initial normal is (0, 0, 1)
    # Rotate around Y: x' = x*cos + z*sin, z' = -x*sin + z*cos
    nx = 0.0 * math.cos(rot_y_rad) + 1.0 * math.sin(rot_y_rad)
    nz = -0.0 * math.sin(rot_y_rad) + 1.0 * math.cos(rot_y_rad)
    normal = Vector3(nx, 0.0, nz).normalize()
    
    # 2. Iterate over pixels
    for y in range(camera.height):
        for x in range(camera.width):
            # Define ray for this pixel
            ray_o = camera.origin
            ray_d = camera.get_direction(x, y)
            
            # Intersect ray with volume bounding box to get t_near, t_far
            t_near, t_far = intersect_box(ray_o, ray_d, volume.bounds)
            
            acc_color = Vector3(0, 0, 0)
            acc_alpha = 0.0
            
            # 3. Ray march
            num_samples = 100
            dt = (t_far - t_near) / num_samples
            
            for i in range(num_samples):
                t = t_near + i * dt
                p = ray_o + ray_d * t
                
                # 4. Check if point is on the valid side of the cutting plane
                # Vector from plane center to sample point
                vec_to_sample = p - plane_center
                
                # Dot product with normal
                # If dot >= 0, it's on the "front" side defined by the normal
                if (vec_to_sample.dot(normal)) >= 0:
                    # Sample volume
                    val = volume.sample(p)
                    
                    # Map to color and opacity (Transfer Function)
                    color, alpha = transfer_function(val, dt)
                    
                    # Front-to-back compositing
                    # C_out = C_acc + (1 - alpha_acc) * c * alpha
                    # alpha_out = alpha_acc + (1 - alpha_acc) * alpha
                    weight = (1.0 - acc_alpha) * alpha
                    acc_color += weight * color
                    acc_alpha += weight
                    
                    # Early ray termination optimization
                    if acc_alpha > 0.99:
                        break
                else:
                    # Point is behind the cut; skip compositing
                    pass
            
            # Write final color to pixel buffer
            pixel_buffer.set_pixel(x, y, acc_color)
```

## Subtopics

### [Region of Interest (ROI) Rendering](10737-region-of-interest-roi-rendering.md)
Region of Interest (ROI) rendering is a technique to visualize internal features by defining a specific region and rendering only that portion of the dataset. It solves the problem where full opacity is obtained before reaching internal structures, making them difficult to see. It sets itself apart from standard volume rendering by restricting the integration domain to a user-defined subset, which can be geometric or volumetric, rather than the entire volume extent.

## Operating characteristics
*   **Time Complexity:** $O(W \cdot H \cdot N)$, where $W, H$ are image dimensions and $N$ is the number of samples per ray. The clipping check adds a constant-time dot product per sample, which is negligible.
*   **Memory Complexity:** $O(W \cdot H \cdot D)$ for the volume data. The plane geometry is minimal.
*   **Strengths:** Interactive performance for exploration; allows users to "slice" through data to see inside without destroying the 3D context of the remaining volume.
*   **Limitations:** The cut is binary (inside/outside); partial volume effects at the cut boundary may appear sharp unless the plane itself is rendered semi-transparently (as done in the source implementation) [S1, unofficial].
*   **When to use:** Interactive exploration of large datasets where internal structures are occluded.
*   **When not to use:** When precise quantitative analysis of the cut surface is required, or when the cut needs to be non-planar.

## Background
The technique is implemented in the "SubairProject1/Volume-Rendering-Visualization-Project" repository, authored by SubairProject1 [S1, unofficial]. It utilizes the Three.js library for WebGL rendering, leveraging `Data3DTexture` for volume storage and `ArrowHelper` for visualizing the cutting plane's normal vector [S1, unofficial]. The implementation demonstrates an interactive clipping technique that renders a semi-transparent plane mesh and normal arrow, dynamically updating raycasting uniforms to slice the volume [S1, unofficial].

## Sources
[S1] SubairProject1/Volume-Rendering-Visualization-Project, https://github.com/SubairProject1/Volume-Rendering-Visualization-Project, fetched 2026-10-26 (unofficial)
