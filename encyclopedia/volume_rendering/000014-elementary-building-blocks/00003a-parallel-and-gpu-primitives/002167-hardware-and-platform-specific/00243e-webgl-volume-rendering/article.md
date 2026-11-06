# WebGL Volume Rendering

Also called: Direct Volume Rendering in WebGL, Ray Casting in Fragment Shader

## Overview
WebGL Volume Rendering performs direct volume ray casting within a web browser using the WebGL API, typically leveraging the Three.js library for scene management and shader execution [S1, unofficial]. It solves the problem of interactive 3D visualization of scalar volumes (such as microscopy or medical data) in client-side applications without requiring native plugins or server-side rendering [S1, unofficial]. It is distinguished from other GPU-based methods by its specific implementation constraints and optimizations for the web ecosystem, including the use of `SharedArrayBuffer` for high-performance data transfer and specific handling of cross-origin isolation requirements [S1, unofficial]. Unlike CUDA-based approaches, it runs on the client's GPU via JavaScript bindings, trading some raw performance for accessibility and ease of deployment [S1, unofficial].

## How it works

### Inputs and Outputs
**Inputs:**
*   `volumeDataUint8`: A JavaScript `Uint8Array` containing the scalar volume data, with one 8-bit value per voxel [S1, unofficial].
*   `volumeSize`: An array `[x, y, z]` specifying the dimensions of the volume in voxels [S1, unofficial].
*   `voxelSize`: An array `[x, y, z]` specifying the physical dimensions of each voxel in consistent units (e.g., microns) [S1, unofficial].
*   `transferFunctionTex`: A Three.js `DataTexture` with width 256 and height 1, using `THREE.RGBAFormat`. The data is a `Uint8Array` of size $256 \times 4$, mapping each 8-bit input value to a 4-byte RGBA color [S1, unofficial].
*   Optional parameters: `cameraPosition`, `cameraTarget`, `cameraUp`, `cameraFovDegrees`, `orbitZoomSpeed`, `alphaScale`, `dtScale`, `finalGamma`, `useLighting`, `useVolumeMirrorX`, `useSurface`, `surfaceMesh`, `surfaceColor`, `interactionSpeedup`, `heightCorrection` [S1, unofficial].

**Outputs:**
*   A rendered 2D image displayed in the WebGL canvas [S1, unofficial].
*   Optional callbacks: `onCameraChange`, `onWebGLRender`, `onRenderedImage` [S1, unofficial].

### Data Structures
The volume data is stored in a flat `Uint8Array` [S1, unofficial]. The transfer function is stored in a 1D texture (`DataTexture`) of size $256 \times 1$ [S1, unofficial]. The camera state is managed by a Three.js `PerspectiveCamera` and `OrbitUnlimitedControls` instance [S1, unofficial].

### Steps
1.  **Initialization**: Create a scene, camera, and WebGL renderer [S1, unofficial]. Initialize the `DataTexture` for the transfer function with the provided RGBA mapping [S1, unofficial].
2.  **Camera Setup**: Configure the camera using `cameraPosition`, `cameraTarget`, `cameraUp`, and `cameraFovDegrees` [S1, unofficial]. Set up orbit controls with `orbitZoomSpeed` [S1, unofficial].
3.  **Ray Casting Loop**: For each pixel in the viewport:
    a.  Compute the ray origin and direction based on the camera matrix and pixel coordinates [S1, unofficial].
    b.  Determine the entry and exit points of the ray into the volume bounding box [S1, unofficial].
    c.  Sample the volume along the ray using a step size scaled by `dtScale` [S1, unofficial].
    d.  For each sample, retrieve the scalar value from `volumeDataUint8` using `volumeSize` and `voxelSize` [S1, unofficial].
    e.  Apply the transfer function from `transferFunctionTex` to get color $C$ and opacity $\alpha$ [S1, unofficial].
    f.  Scale the opacity by `alphaScale` [S1, unofficial].
    g.  Apply front-to-back compositing:
        $$ C_{out} = C_{in} + C_{sample} \cdot \alpha_{sample} \cdot (1 - \alpha_{in}) $$
        $$ \alpha_{out} = \alpha_{in} + \alpha_{sample} \cdot (1 - \alpha_{in}) $$
        where $C_{in}$ and $\alpha_{in}$ are the accumulated color and opacity from previous samples [S1, unofficial].
    h.  If `useLighting` is true, compute lighting based on gradients in the volume as surface normals [S1, unofficial].
4.  **Post-Processing**: Apply the final gamma correction to the accumulated color components $c \in [0, 1]$:
    $$ c_{final} = c^{(1.0 / \text{finalGamma})} $$
    [S1, unofficial].
5.  **Surface Rendering**: If `useSurface` is true and `surfaceMesh` is provided, render the mesh within the volume [S1, unofficial]. The mesh color is defined by `surfaceColor` and must be fully opaque (no alpha) [S1, unofficial].
6.  **Mirroring**: If `useVolumeMirrorX` is true, mirror the volume data along the x-axis during sampling [S1, unofficial].
7.  **Interaction Speedup**: If `interactionSpeedup` > 1, reduce the rendering resolution during camera manipulation to improve performance [S1, unofficial].
8.  **Callbacks**: Trigger `onCameraChange` when the camera moves, `onWebGLRender` after each render, and `onRenderedImage` with the pixel data [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cast_ray(ray_origin, ray_direction, volume_data, volume_size, voxel_size, tf_data, dt_scale, alpha_scale, final_gamma, mirror_x):
    # Volume bounds (centered at origin)
    min_x, max_x = -volume_size[0] * voxel_size[0] / 2.0, volume_size[0] * voxel_size[0] / 2.0
    min_y, max_y = -volume_size[1] * voxel_size[1] / 2.0, volume_size[1] * voxel_size[1] / 2.0
    min_z, max_z = -volume_size[2] * voxel_size[2] / 2.0, volume_size[2] * voxel_size[2] / 2.0

    # Ray-box intersection (slab method)
    t_min = -float('inf')
    t_max = float('inf')
    
    bounds = [(min_x, max_x), (min_y, max_y), (min_z, max_z)]
    for i in range(3):
        d = ray_direction[i]
        if abs(d) < 1e-8:
            if ray_origin[i] < bounds[i][0] or ray_origin[i] > bounds[i][1]:
                return [0.0, 0.0, 0.0]
        else:
            t1 = (bounds[i][0] - ray_origin[i]) / d
            t2 = (bounds[i][1] - ray_origin[i]) / d
            t_min = max(t_min, min(t1, t2))
            t_max = min(t_max, max(t1, t2))
            
    if t_min > t_max or t_max < 0:
        return [0.0, 0.0, 0.0]

    step_size = dt_scale
    t = t_min
    acc_color = [0.0, 0.0, 0.0]
    acc_alpha = 0.0

    while t < t_max:
        # Sample position
        p = [ray_origin[i] + ray_direction[i] * t for i in range(3)]
        
        # Handle mirroring along X
        if mirror_x:
            p[0] = -p[0]
            
        # Convert to voxel indices
        ix = int((p[0] - min_x) / voxel_size[0])
        iy = int((p[1] - min_y) / voxel_size[1])
        iz = int((p[2] - min_z) / voxel_size[2])
        
        # Boundary check
        if 0 <= ix < volume_size[0] and 0 <= iy < volume_size[1] and 0 <= iz < volume_size[2]:
            # Flat index calculation
            idx = ix + iy * volume_size[0] + iz * volume_size[0] * volume_size[1]
            scalar = volume_data[idx]
            
            # Transfer function lookup
            tf_idx = max(0, min(255, int(scalar)))
            r, g, b, a = tf_data[tf_idx * 4], tf_data[tf_idx * 4 + 1], tf_data[tf_idx * 4 + 2], tf_data[tf_idx * 4 + 3]
            color = [r / 255.0, g / 255.0, b / 255.0]
            alpha = a / 255.0
            
            # Scale opacity
            alpha *= alpha_scale
            
            # Front-to-back compositing
            weight = alpha * (1.0 - acc_alpha)
            acc_color = [acc_color[i] + color[i] * weight for i in range(3)]
            acc_alpha += weight
            
        t += step_size

    # Final gamma correction
    final_color = [c ** (1.0 / final_gamma) for c in acc_color]
    return final_color
```

### Parameters
*   `alphaScale` (default: 1): Scales opacity at each sample [S1, unofficial].
*   `dtScale` (default: 1): Scales step size; higher values increase performance but decrease quality [S1, unofficial].
*   `finalGamma` (default: 4.5): Gamma correction applied to accumulated color; increases visibility of faint details [S1, unofficial].
*   `useLighting` (default: true): Enables lighting using gradients as normals; if false, resembles maximum intensity projection [S1, unofficial].
*   `useVolumeMirrorX` (default: false): Mirrors volume data along the x-axis [S1, unofficial].
*   `interactionSpeedup` (default: 1): Reduces rendering resolution during camera interaction [S1, unofficial].
*   `cameraPosition` (default: [0, 0, -2]): Initial camera position relative to the volume box (centered at origin, longest dimension -0.5 to 0.5) [S1, unofficial].
*   `cameraTarget` (default: [0, 0, 0]): Initial camera target [S1, unofficial].
*   `cameraUp` (default: [0, -1, 0]): Initial camera up direction [S1, unofficial].
*   `cameraFovDegrees` (default: 45.0): Vertical field of view in degrees [S1, unofficial].
*   `orbitZoomSpeed` (default: 0.15): Speed of camera zoom on mouse-wheel event [S1, unofficial].
*   `useSurface` (default: false): Controls visibility of the surface mesh [S1, unofficial].
*   `surfaceColor` (default: '#00ff00'): Color of the opaque surface mesh; no alpha allowed [S1, unofficial].
*   `heightCorrection` (default: true): Applies correction when resizing DOM item height [S1, unofficial].

### Edge Cases and Pitfalls
*   **SharedArrayBuffer Error**: Opening H5J files may produce a "SharedArrayBuffer is not defined" error if cross-origin isolation is not set up properly [S1, unofficial].
*   **Surface Mesh Alpha**: The surface mesh color must be fully opaque; alpha is not supported for the mesh [S1, unofficial].
*   **Data Peak and Gamma**: The opacity falloff is controlled by the "data peak" and "data $\gamma$" parameters, which affect the transfer function application [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: Linear with respect to the number of pixels and the number of samples per ray [S1, unofficial].
*   **Memory Complexity**: Linear with respect to the volume size and the transfer function size [S1, unofficial].
*   **Performance**: Performance can be improved by increasing `dtScale` or `interactionSpeedup`, at the cost of quality and interactivity [S1, unofficial].
*   **Strengths**: Runs in the browser, no plugin required, interactive camera controls [S1, unofficial].
*   **Limitations**: Limited by WebGL capabilities and browser security restrictions (e.g., cross-origin isolation) [S1, unofficial].
*   **When to Use**: Client-side visualization of scalar volumes in web applications [S1, unofficial].
*   **When Not to Use**: When the browser environment cannot be configured for cross-origin isolation, which causes `SharedArrayBuffer` errors [S1, unofficial].

## Background
WebGL Volume Rendering is implemented in the `@janelia/web-vol-viewer` package by JaneliaSciComp [S1, unofficial]. It provides a reusable component for integrating volume rendering into web applications, including React functional components [S1, unofficial]. The package includes a basic UI for controlling rendering parameters and supports keyboard shortcuts for toggling surface visibility, lighting, and mirroring [S1, unofficial].

## Sources
[S1] JaneliaSciComp/web-vol-viewer, https://github.com/JaneliaSciComp/web-vol-viewer, fetched 2026-10-26 (unofficial)
