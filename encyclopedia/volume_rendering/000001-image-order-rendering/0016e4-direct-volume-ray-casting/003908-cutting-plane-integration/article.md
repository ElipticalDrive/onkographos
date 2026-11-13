# Cutting Plane Integration

Also called: CuttingPlaneRender, Interactive Cutting Plane

## Overview
Cutting Plane Integration is an interactive volume rendering technique that dynamically slices a 3D dataset using a semi-transparent geometric plane to reveal internal structures [S1, unofficial]. It solves the occlusion problem where opaque outer layers prevent the visualization of internal features, such as seeds within pulp, by isolating the target volume through real-time clipping [S1, unofficial]. Unlike standard clipping which often relies on global bounding planes, this method supports arbitrary geometric envelopes and intensity-based detection [S1, unofficial][S2]. It builds on [Direct volume ray casting](19-direct-volume-ray-casting.md) by modifying the ray traversal and compositing stages to respect the cutting plane boundary.

## How it works
The method integrates a cutting plane into the ray casting loop. The plane is defined by a position and orientation, and potentially a near-clip constraint.

**Inputs and Outputs:**
*   **Input:** Volume data $V$, camera parameters ($\mathbf{o}, \mathbf{d}$), transfer function $T$, cutting plane definition (position $\mathbf{g}_0$, normal $\mathbf{n}$, rotation angles), and near-clip plane definition (optional).
*   **Output:** Final pixel color $\mathbf{C}_{out}$ and opacity $\alpha_{out}$.

**Data Structures:**
*   **Volume:** 3D grid of scalar values. In implementations like [S1, unofficial], data is loaded as a `Float32Array` normalized from `uint16` (divided by 4095.0) [S1, unofficial].
*   **Cutting Plane:** Defined by a world-space position $\mathbf{p}_{plane}$ and a normal vector $\mathbf{n}$. The position is often mapped from a normalized height parameter $h \in [0, 1]$ [S1, unofficial].
*   **Ray State:** Current position $\mathbf{p}$, accumulated color $\mathbf{C}_{acc}$, accumulated opacity $\alpha_{acc}$.

**Numbered Steps:**

1.  **Initialize Ray:** For each pixel, define ray $r(t) = \mathbf{o} + t\mathbf{d}$. Intersect with volume bounding box to find $t_{near}$ and $t_{far}$ [G].
2.  **Define Cutting Plane:**
    *   Map normalized height $h$ to world Z position: $z_{plane} = z_{min} + h \cdot (z_{max} - z_{min})$ [S1, unofficial]. Specifically, in [S1, unofficial], $z_{plane} = 200 - 400 \cdot h$.
    *   Apply rotation. If `renderAbove` is true, apply a $\pi$ radian offset to the Y-rotation [S1, unofficial].
    *   The plane equation is $\mathbf{n} \cdot (\mathbf{x} - \mathbf{g}_0) = 0$, where $\mathbf{g}_0$ is a point on the plane [S2].
3.  **Ray Marching Loop:** Iterate $i$ from $0$ to $N-1$:
    *   Calculate sample position $\mathbf{p}_i = \mathbf{o} + t_i \mathbf{d}$.
    *   **Cutting Plane Check:** Determine if the ray segment $[t_i, t_{i+1}]$ intersects the cutting plane.
        *   Let $\mathbf{g}_{in} = \mathbf{p}_i$ and $\mathbf{g}_{out} = \mathbf{p}_{i+1}$ in world space.
        *   If $(\mathbf{n} \cdot (\mathbf{g}_{in} - \mathbf{g}_0))$ and $(\mathbf{n} \cdot (\mathbf{g}_{out} - \mathbf{g}_0))$ have opposite signs, the ray intersects the plane [S2].
        *   Calculate the exact intersection point $\mathbf{g}^*$ using linear interpolation [S2].
        *   Find the corresponding parameter $t^*$ on the ray such that $r(t^*) = \mathbf{g}^*$ [S2].
        *   If the ray is "above" the cutting plane (depending on normal direction and `renderAbove` flag), stop compositing for this ray or treat samples beyond $\mathbf{g}^*$ as transparent [S1, unofficial].
    *   **Near-Clip Plane Check (Optional):**
        *   If a near-clip plane is defined, check for intersections with surface boundaries.
        *   If the number of ray-surface intersections is odd, the near plane is inside the volume [S2].
        *   Use root finding (e.g., Newton-Raphson) to solve $\phi(\mathbf{p}^*) - \mathbf{g}^* = 0$ for the precise point $\mathbf{p}^*$ corresponding to the geometric intersection $\mathbf{g}^*$ [S2].
    *   **Sampling:** Retrieve $V(\mathbf{p}_i)$ and compute $\mathbf{c}_i, \alpha_i$ using the transfer function [S2, S2789].
    *   **Compositing:** Update accumulators:
        $$
        \mathbf{C}_{acc} \leftarrow \mathbf{C}_{acc} + (1 - \alpha_{acc}) \cdot \mathbf{c}_i \cdot \alpha_i
        $$
        $$
        \alpha_{acc} \leftarrow \alpha_{acc} + (1 - \alpha_{acc}) \cdot \alpha_i
        $$
4.  **Output:** Set $\mathbf{C}_{out} = \mathbf{C}_{acc}$, $\alpha_{out} = \alpha_{acc}$.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cutting_plane_integration(volume, camera, plane_params, transfer_function):
    """
    volume: 3D array of scalars
    camera: object with origin o, direction d, t_near, t_far
    plane_params: dict with height (0-1), rotX, rotY, renderAbove (bool)
    transfer_function: function V -> (color, alpha)
    """
    # 1. Define Cutting Plane Geometry
    # Map normalized height to world Z
    z_min, z_max = volume.bounds.z_min, volume.bounds.z_max
    h = plane_params['height']
    z_plane = z_min + h * (z_max - z_min)
    
    # Calculate normal vector based on rotations
    # Initial normal along Z-axis (0, 0, 1)
    n = Vector3(0, 0, 1)
    
    # Apply rotation around X
    n = n.rotate_x(plane_params['rotX'])
    
    # Apply rotation around Y
    rot_y = plane_params['rotY']
    if plane_params['renderAbove']:
        rot_y += PI # 180 degree offset
    n = n.rotate_y(rot_y)
    
    # Plane point g0 is on the plane, e.g., (0, 0, z_plane)
    g0 = Vector3(0, 0, z_plane)
    
    # 2. Ray Casting Loop
    for pixel in image_pixels:
        ray_o = camera.origin
        ray_d = camera.direction_for_pixel(pixel)
        
        # Intersect ray with volume bounding box
        t_entry, t_exit = intersect_box(ray_o, ray_d, volume.bounds)
        
        c_acc = Vector3(0, 0, 0)
        a_acc = 0.0
        
        # Discretize ray
        steps = 100
        dt = (t_exit - t_entry) / steps
        
        for i in range(steps):
            t = t_entry + i * dt
            p_curr = ray_o + ray_d * t
            t_next = t + dt
            p_next = ray_o + ray_d * t_next
            
            # Check intersection with cutting plane
            # Evaluate plane equation: n . (p - g0)
            val_curr = dot(n, p_curr - g0)
            val_next = dot(n, p_next - g0)
            
            # If signs are opposite, intersection occurs between p_curr and p_next
            if val_curr * val_next < 0:
                # Calculate intersection parameter t_int
                # Linear interpolation
                t_int = t + dt * (val_curr / (val_curr - val_next))
                
                # Determine if we should stop or continue based on "above/below" logic
                # Assuming we want to render "below" the plane (or as defined by normal)
                # If the ray has passed the plane in the "forbidden" direction, break
                # This logic depends on specific application requirements [S1, unofficial]
                # For simplicity, if we hit the plane, we might stop or just clip
                break 
            
            # Sample volume
            v_val = volume.sample(p_curr)
            color, alpha = transfer_function(v_val)
            
            # Compositing
            c_acc = c_acc + (1.0 - a_acc) * color * alpha
            a_acc = a_acc + (1.0 - a_acc) * alpha
            
        # Set pixel color
        set_pixel_color(pixel, c_acc, a_acc)
```

**Parameters:**
*   `planeHeight`: Normalized float $[0, 1]$ mapping to the Z-axis position [S1, unofficial].
*   `planeRotX`, `planeRotY`: Rotation angles in degrees around the X and Y axes [S1, unofficial].
*   `renderAbove`: Boolean flag that flips the plane 180° around Y when true [S1, unofficial].

**Edge Cases:**
*   **Precision:** Root finding for near-clip planes requires careful initialization (e.g., $x_0 = p_{in} + \epsilon \frac{p_{out} - p_{in}}{\|p_{out} - p_{in}\|_{L2}}$) to ensure convergence [S2].
*   **Boundaries:** Large pixel frustums may cause ray-surface intersections at block boundaries; root finding methods remain pixel-accurate in these cases [S2].

## Subtopics

### [Region of Interest (ROI) Rendering](10737-region-of-interest-roi-rendering.md)
Region of Interest (ROI) rendering is a technique to visualize internal features by defining a specific region and rendering only that portion of the dataset. It solves the problem where full opacity is obtained before reaching internal structures of interest, such as seeds within pulp, by isolating the target volume. Unlike standard clipping which often uses global planes, ROI rendering supports arbitrary geometric envelopes, masks, and intensity-based automatic detection.

### Cutting and Near Clip Planes
This variant uses a root-finding based method to realize cutting planes and near-clip planes during volume rendering compositing [S2]. For near-clip planes, since rendering is water-tight, an odd number of ray-surface intersections indicates the near plane is inside the volume [S2]. The intersection point $g^* \in G$ is determined using trivial formulas, and the corresponding point $p^* \in P$ is found by solving $\phi(p^*) - g^* = 0$ using a root-finding algorithm (e.g., Newton-Raphson) [S2].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of samples per ray $N$ and the number of pixels. The root-finding step adds a small constant overhead per intersection [S2].
*   **Memory Complexity:** Linear with respect to the volume size for storage. The cutting plane itself requires minimal memory [S1, unofficial].
*   **Performance:** Interactive frame rates are achievable due to the simplicity of the plane intersection test [S1, unofficial]. Root-finding methods for near-clipping are more computationally expensive than simple plane checks but provide pixel-accurate geometry [S2].
*   **Strengths:** Real-time interactivity, simple implementation, effective for revealing internal structures [S1, unofficial].
*   **Limitations:** Limited to planar cuts (unless extended to ROI envelopes). Root-finding may fail if not initialized properly [S2].

## Background
Cutting plane integration is a standard technique in interactive volume visualization. It is widely used in medical imaging and scientific visualization to explore internal structures without removing outer layers. The specific implementation details regarding normalized height mapping and interactive plane manipulation are documented in unofficial projects [S1, unofficial]. The theoretical basis for pixel-accurate cutting and near-clipping using root-finding methods is described in research on isogeometric volume visualization [S2].

## Sources
[S1] SubairProject1/Volume-Rendering-Visualization-Project, https://github.com/SubairProject1/Volume-Rendering-Visualization-Project, fetched 2026-10-26 (unofficial)
[S2] Interactive Isogeometric Volume Visualization with Pixel-Accurate Geometry, https://doi.org/10.1109/tvcg.2015.2430337, fetched 2026-10-28
