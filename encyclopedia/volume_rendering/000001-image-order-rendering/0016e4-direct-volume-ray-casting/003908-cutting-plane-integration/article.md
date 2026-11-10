# Cutting Plane Integration

Also called: CuttingPlaneRender, Interactive Cutting Plane

## Overview
Cutting Plane Integration is an interactive volume rendering technique that dynamically slices a 3D dataset using a semi-transparent clipping plane to reveal internal structures that would otherwise be occluded by front-facing opaque regions [S1, unofficial]. It differs from standard direct volume ray casting by restricting the integration domain along the ray path to either the region "above" or "below" a user-defined plane, effectively performing a real-time, view-dependent region-of-interest extraction [S1, unofficial].

## How it works
The method extends the standard direct volume ray casting loop by introducing a dynamic clipping boundary. The ray is intersected with the volume bounding box to determine the initial entry and exit points. A cutting plane, defined by a position and orientation, further subdivides the ray segment. The integration proceeds only through the valid sub-segment defined by the `renderAbove` flag.

**Inputs and Outputs:**
*   **Input:** 3D volume data (scalar field), camera parameters (origin $\mathbf{o}$, direction $\mathbf{d}$), cutting plane parameters (height $h$, rotation angles $\theta_x, \theta_y$, boolean `renderAbove`).
*   **Output:** Final pixel color $C_{out}$ and opacity $\alpha_{out}$.

**Data Structures:**
*   **Volume Data:** Stored as a `Float32Array` of normalized values $[0, 1]$. Dimensions are stored as `width`, `height`, `depth` [S1, unofficial].
*   **Cutting Plane:** Defined by a position vector $\mathbf{p}_{plane}$ and a normal vector $\mathbf{n}$. The position is derived from a normalized height parameter $h \in [0, 1]$ [S1, unofficial].

**Steps:**

1.  **Ray Generation:** Determine the ray segment $[t_{start}, t_{end}]$ intersecting the volume bounding box.
2.  **Plane Definition:** Calculate the cutting plane's world position and orientation based on input parameters [S1, unofficial]:
    *   Map normalized height $h$ to world Z position: $z_{plane} = 200 - 400 \cdot h$.
    *   Convert rotation angles $\theta_x, \theta_y$ (degrees) to radians. Apply rotation to the plane's normal vector. If `renderAbove` is true, apply a $180^\circ$ ($\pi$ radians) offset to the Y-rotation to flip the plane's orientation [S1, unofficial].
3.  **Ray-Plane Intersection:** Intersect the ray $r(t) = \mathbf{o} + t\mathbf{d}$ with the cutting plane to find the intersection parameter $t_{clip}$.
    *   If the ray is parallel to the plane, $t_{clip}$ is undefined; handle as no cut.
    *   Otherwise, $t_{clip} = \frac{(\mathbf{p}_{plane} - \mathbf{o}) \cdot \mathbf{n}}{\mathbf{d} \cdot \mathbf{n}}$.
4.  **Segment Clipping:** Determine the effective integration interval $[t_{eff\_start}, t_{eff\_end}]$ based on `renderAbove`:
    *   If `renderAbove` is true: The valid region is between the plane and the far boundary. Set $t_{eff\_start} = \max(t_{start}, t_{clip})$ and $t_{eff\_end} = t_{end}$.
    *   If `renderAbove` is false: The valid region is between the near boundary and the plane. Set $t_{eff\_start} = t_{start}$ and $t_{eff\_end} = \min(t_{end}, t_{clip})$.
    *   If $t_{eff\_start} \ge t_{eff\_end}$, the ray does not pass through the valid region; output background color.
5.  **Discretization and Integration:** Discretize the interval $[t_{eff\_start}, t_{eff\_end}]$ into $N$ samples with step $\Delta t = (t_{eff\_end} - t_{eff\_start}) / N$. For each sample $i$:
    *   Compute sample point $\mathbf{p}_i = \mathbf{o} + (t_{eff\_start} + i \cdot \Delta t) \mathbf{d}$.
    *   Sample volume value $V(\mathbf{p}_i)$ using trilinear interpolation [S1, unofficial].
    *   Map $V(\mathbf{p}_i)$ to color $C_i$ and opacity $\alpha_i$ using the transfer function.
    *   Composite front-to-back:
        $$ C_{out} = C_{in} + (1 - \alpha_{in}) \cdot C_i \cdot \alpha_i $$
        $$ \alpha_{out} = \alpha_{in} + (1 - \alpha_{in}) \cdot \alpha_i $$
        Initialize $C_{in} = 0, \alpha_{in} = 0$.

**Parameters:**
*   `planeHeight`: Normalized float $[0, 1]$ controlling the Z-axis position of the plane [S1, unofficial].
*   `planeRotX`, `planeRotY`: Rotation angles in degrees [S1, unofficial].
*   `renderAbove`: Boolean flag determining which side of the plane is rendered [S1, unofficial].

**Edge Cases:**
*   **Ray Parallel to Plane:** If $\mathbf{d} \cdot \mathbf{n} \approx 0$, the ray never intersects the plane. The integration interval remains $[t_{start}, t_{end}]$ (full volume) or is skipped depending on implementation logic [G].
*   **Precision:** Floating-point errors in ray-plane intersection may cause gaps or overlaps in the slice. Use a small epsilon for comparisons [G].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cutting_plane_integration(ray_origin, ray_direction, volume, plane_height, plane_rot_x, plane_rot_y, render_above):
    # 1. Ray-Volume Intersection (simplified bounding box)
    t_min, t_max = intersect_ray_bbox(ray_origin, ray_direction, volume.bbox)
    if t_min > t_max:
        return background_color

    # 2. Define Cutting Plane
    # Map height to Z position
    z_plane = 200 - 400 * plane_height
    plane_pos = Vector3(0, 0, z_plane)
    
    # Calculate normal based on rotations
    # Convert degrees to radians
    theta_x = math.radians(plane_rot_x)
    theta_y = math.radians(plane_rot_y)
    
    # Base normal along Z
    normal = Vector3(0, 0, 1)
    
    # Apply rotations (simplified rotation matrix application)
    # Rotate around X
    normal = rotate_x(normal, theta_x)
    # Rotate around Y
    normal = rotate_y(normal, theta_y)
    
    # Flip if renderAbove is true (180 degree offset around Y)
    if render_above:
        normal = rotate_y(normal, math.pi)
    
    normal.normalize()

    # 3. Ray-Plane Intersection
    denom = dot(ray_direction, normal)
    if abs(denom) < 1e-6:
        # Ray parallel to plane
        t_clip = float('inf')
    else:
        t_clip = dot(plane_pos - ray_origin, normal) / denom

    # 4. Determine Integration Interval
    if render_above:
        t_start = max(t_min, t_clip)
        t_end = t_max
    else:
        t_start = t_min
        t_end = min(t_max, t_clip)

    if t_start >= t_end:
        return background_color

    # 5. Discretize and Composite
    num_samples = 100 # Typical value, not specified in sources
    delta_t = (t_end - t_start) / num_samples
    
    acc_color = Vector3(0, 0, 0)
    acc_alpha = 0.0
    
    for i in range(num_samples):
        t = t_start + i * delta_t
        p = ray_origin + ray_direction * t
        
        # Sample volume
        val = volume.sample_trilinear(p)
        
        # Transfer function
        color, alpha = transfer_function(val)
        
        # Front-to-back compositing
        acc_color = acc_color + (1 - acc_alpha) * color * alpha
        acc_alpha = acc_alpha + (1 - acc_alpha) * alpha
        
        if acc_alpha > 0.99: # Early ray marching termination
            break
            
    return acc_color
```

## Subtopics

### [Region of Interest (ROI) Rendering](10737-region-of-interest-roi-rendering.md)
A generalization where the cut region is defined by arbitrary geometric or volumetric masks rather than a single plane. Choose when complex internal structures need isolation beyond a simple slice.

## Operating characteristics
*   **Time Complexity:** $O(N)$ per pixel, where $N$ is the number of samples in the valid interval. The interval length varies dynamically, potentially reducing $N$ compared to full volume rendering.
*   **Memory Complexity:** $O(1)$ additional memory per pixel for compositing state. The volume data must be resident in memory.
*   **Performance:** Interactive frame rates are achievable on GPU hardware due to the simplicity of the clipping logic and the use of 3D textures [S1, unofficial]. Performance scales with volume resolution and the size of the remaining valid interval.
*   **Strengths:** Enables interactive exploration of internal structures without pre-segmentation. Simple to implement as a shader modification.
*   **Limitations:** Only exposes structures on one side of the plane. Occlusion from the "cut" side is completely removed, which may be undesirable if context is needed.

## Background
Cutting Plane Integration is implemented in interactive visualization projects such as the SubairProject1 Volume Rendering Visualization Project, which uses Three.js for real-time rendering [S1, unofficial]. It builds on the principles of direct volume ray casting [19-direct-volume-ray-casting.md] and is closely related to Region of Interest (ROI) rendering techniques [10737-region-of-interest-roi-rendering.md].

## Sources
[S1] SubairProject1/Volume-Rendering-Visualization-Project, https://github.com/SubairProject1/Volume-Rendering-Visualization-Project, fetched 2026-10-26 (unofficial)
