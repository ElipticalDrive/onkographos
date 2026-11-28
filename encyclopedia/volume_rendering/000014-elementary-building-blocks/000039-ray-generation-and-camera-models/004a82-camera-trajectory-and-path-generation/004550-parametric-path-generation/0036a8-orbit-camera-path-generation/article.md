# Orbit Camera Path Generation

Also called: Camera Orbit Animation, Camera Orbit Generation, Circular Camera Path, Orbit Camera Trajectory Generation, Rotating Mesh Video Rendering, Rotation animation camera setup, Turntable Mode

## Overview
Orbit Camera Path Generation creates a sequence of rendered frames by incrementally rotating the camera around a fixed focal point, producing a smooth circular visualization effect [S2, unofficial][S3]. It differs from its siblings, Spiral and Elliptical path generation, by constraining the camera trajectory to a single plane perpendicular to the rotation axis at a constant radius, ensuring uniform angular velocity and no change in elevation or distance [S1, unofficial][S2, unofficial].

## How it works
**Inputs:**
- `active_camera`: The camera object or state to be transformed [S1, unofficial][S2, unofficial].
- `frame_count`: Total number of frames to generate (default 36) [S1, unofficial].
- `azimuth_increment`: Degrees to rotate the camera around the view-up axis per frame (default 10) [S1, unofficial].
- `rotate_axis`: Integer specifying the rotation axis: 0 for Elevation, 1 for Roll, 2 for Azimuth [S2, unofficial].
- `rotate_angles`: Optional list of float angles in degrees to rotate the camera at each step [S2, unofficial].
- `resolution`: Output image dimensions (default [1000, 800]) [S1, unofficial].
- `output_dir`: Directory path for saving rendered frames (default `outputs/preview`) [S1, unofficial].
- **Outputs:** A sequence of rendered frames saved as a PNG image sequence or a 4D NumPy array [S1, unofficial][S2, unofficial].

**Data Structures:**
- `rendered_image`: A 4D NumPy array of shape `(frame_count, height, width, 3)` with dtype `uint8`, initialized to zeros [S2, unofficial].
- `image_filter`: A VTK `vtkWindowToImageFilter` instance used to convert the render window to an image buffer [S2, unofficial].

**Steps:**
1. **Initialize**: Obtain the active camera object and configure the rendering window with the specified `resolution` [S1, unofficial][S2, unofficial].
2. **Loop**: Iterate `i` from `0` to `frame_count - 1` [S1, unofficial].
3. **Compute Angle**: If `rotate_angles` is provided, set `angle = rotate_angles[i]` [S2, unofficial]. Otherwise, set `angle = azimuth_increment` [S1, unofficial].
4. **Apply Rotation**: Rotate the camera using the `rotate_axis` specification. For azimuth rotation (axis 2), call `camera.Azimuth(angle)` [S1, unofficial][S2, unofficial].
5. **Render**: Trigger the rendering pipeline. If using VTK, call `image_filter.Modified()` and `image_filter.Update()`, then `window.Render()` [S2, unofficial].
6. **Capture**: Extract the rendered image data. In VTK, convert `image_filter.GetOutput()` to a NumPy array [S2, unofficial].
7. **Store**: Assign the captured image to `rendered_image[i, :, :, :]` [S2, unofficial].
8. **Reset (Optional)**: Apply the inverse rotation `camera.Azimuth(-angle)` after capture to prevent cumulative drift [S2, unofficial].
9. **Save**: Write the frame to `output_dir` with a sequential filename (e.g., `frame_%03d.png`) [S1, unofficial].

**Formulas:**
For a circular orbit in the XY plane around a center $\mathbf{c} = (c_x, c_y, c_z)$ at radius $r$:
$$ \theta_i = \frac{2\pi \cdot i}{\text{num\_frames}} $$
$$ x_i = c_x + r \cdot \cos(\theta_i) $$
$$ y_i = c_y + r \cdot \sin(\theta_i) $$
$$ z_i = c_z + \text{elevation} $$
The look-at direction is $\mathbf{d} = \mathbf{c} - \mathbf{p}_i$, where $\mathbf{p}_i = (x_i, y_i, z_i)$ [S5, unofficial]. Camera orientation angles are computed as $\text{rot}_z = \arctan2(d_y, d_x) - \frac{\pi}{2}$ and $\text{rot}_x = -\arctan2(d_z, \sqrt{d_x^2 + d_y^2})$ [S5, unofficial]. The optimal focal length is $f = \max(f_{\text{base}}, r \cdot 0.5)$ [S5, unofficial].

**Parameters:**
- `azimuth_increment`: Typical value 10 degrees [S1, unofficial].
- `frame_count`: Typical value 36 [S1, unofficial].
- `resolution`: Typical value [1000, 800] [S1, unofficial].
- `rotate_axis`: 0 (Elevation), 1 (Roll), 2 (Azimuth) [S2, unofficial].

**Edge Cases:**
- **Cumulative Error**: Repeated floating-point rotations can accumulate orientation drift [S2, unofficial]. Using `rotate_angles` with explicit values or resetting the camera orientation periodically mitigates this [S2, unofficial].
- **Axis Ambiguity**: The `rotate_axis` parameter must be consistently interpreted by the camera API [S2, unofficial]. VTK uses 0, 1, 2 for Elevation, Roll, Azimuth respectively [S2, unofficial].
- **Empty Grid**: If the input grid is empty, the function returns early to prevent invalid camera setup or division by zero [S5, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import os

def generate_orbit_frames(active_camera, frame_count, azimuth_increment, 
                          rotate_axis, resolution, output_dir, rotate_angles=None):
    """Generates orbit camera frames."""
    os.makedirs(output_dir, exist_ok=True)
    height, width = resolution
    rendered_image = np.zeros((frame_count, height, width, 3), dtype=np.uint8)

    for i in range(frame_count):
        if rotate_angles is not None:
            angle = rotate_angles[i]
        else:
            angle = azimuth_increment

        if rotate_axis == 2:
            active_camera.Azimuth(angle)
        elif rotate_axis == 0:
            active_camera.Elevation(angle)
        elif rotate_axis == 1:
            active_camera.Roll(angle)

        # [G] Trigger renderer and extract pixel buffer to NumPy array
        frame = np.zeros((height, width, 3), dtype=np.uint8)
        rendered_image[i] = frame

        # [G] Write frame to disk using standard image library
        filename = os.path.join(output_dir, f"frame_{i:03d}.png")

        # Reset rotation to prevent cumulative drift
        if rotate_axis == 2:
            active_camera.Azimuth(-angle)
        elif rotate_axis == 0:
            active_camera.Elevation(-angle)
        elif rotate_axis == 1:
            active_camera.Roll(-angle)

    return rendered_image
```

## Subtopics

### [IM-3D](8092-im-3d.md)
A method for 3D reconstruction that uses a fine-tuned image-to-video model to generate turn-table style video sequences (orbit paths) from a single image, which are then used for 3D Gaussian Splatting. It differs by generating the path content via a video model rather than geometric camera transforms.

## Operating characteristics
- **Time Complexity**: $O(F)$, where $F$ is the number of frames, as each frame requires one rotation and one render operation [S1, unofficial][S2, unofficial].
- **Memory Complexity**: $O(F \cdot H \cdot W \cdot C)$ when pre-allocating the frame buffer [S2, unofficial].
- **Performance**: Rendering time depends on the scene complexity and resolution [S1, unofficial]. Typical resolution [1000, 800] with 36 frames is feasible for real-time preview but may require batch processing for high-res volume rendering [S1, unofficial].
- **Strengths**: Simple to implement; produces smooth, continuous rotation ideal for visualizing symmetric or centrally focused data [S1, unofficial].
- **Limitations**: Does not provide elevation changes (unlike Spiral) or aspect variation (unlike Elliptical), potentially hiding features located above or below the equatorial plane [S2, unofficial].

## Background
The method is commonly implemented in scientific visualization pipelines using VTK and ParaView [S2, unofficial][S6, unofficial]. It is utilized for exporting animation sequences (e.g., PNG frames) that can be stitched into videos using tools like ffmpeg [S1, unofficial]. The approach is foundational in volume rendering workflows for creating turn-table style visualizations [S2, unofficial].

## Sources
[S1] ashawkey/stable-dreamfusion, https://github.com/ashawkey/stable-dreamfusion, fetched 2026-10-26 (unofficial)
[S2] cjcamp16/sciviz-paraview, https://github.com/cjcamp16/sciviz-paraview, fetched 2026-10-26 (unofficial)
[S3] MagicaVoxel .vox, https://ephtracy.github.io/, fetched 2026-10-26
[S4] Laityperfect7/GaussianScan-Recon, https://github.com/Laityperfect7/GaussianScan-Recon, fetched 2026-10-27 (unofficial)
[S5] nath54/NaxelEditor, https://github.com/nath54/NaxelEditor, fetched 2026-10-27 (unofficial)
[S6] Volume Renderer with VTK - yuta-hi, https://github.com/yuta-hi/volume-renderer, fetched 2026-10-29 (unofficial)
