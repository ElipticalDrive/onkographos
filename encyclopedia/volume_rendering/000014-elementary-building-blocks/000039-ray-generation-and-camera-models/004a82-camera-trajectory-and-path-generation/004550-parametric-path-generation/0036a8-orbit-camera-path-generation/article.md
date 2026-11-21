# Orbit Camera Path Generation

Also called: Camera Orbit Animation, Circular Camera Path, Cyclic rotation rendering, Orbit rendering, Orbiting Render, Rotation Rendering

## Overview
Orbit Camera Path Generation creates a sequence of rendered frames by incrementally rotating the camera around a fixed focal point, producing a smooth circular visualization effect [S1, unofficial][S2, unofficial]. It differs from its siblings, Spiral and Elliptical path generation, by constraining the camera trajectory to a single plane perpendicular to the rotation axis at a constant radius, ensuring uniform angular velocity and no change in elevation or distance [S1, unofficial][S2, unofficial].

## How it works
**Inputs:**
- `active_camera`: The camera object or state to be transformed [S1, unofficial][S2, unofficial].
- `frame_count`: Total number of frames to generate (e.g., 36) [S1, unofficial].
- `azimuth_increment`: Degrees to rotate the camera around the view-up axis per frame (e.g., 10 degrees) [S1, unofficial].
- `rotate_axis`: Integer specifying the rotation axis: 0 for Elevation, 1 for Roll, 2 for Azimuth [S2, unofficial].
- `rotate_angles`: Optional list of float angles in degrees to rotate the camera at each step, overriding `azimuth_increment` if provided [S2, unofficial].
- `resolution`: Output image dimensions (e.g., [1000, 800]) [S1, unofficial].
- `output_dir`: Directory path for saving rendered frames [S1, unofficial].

**Data Structures:**
- `rendered_image`: A 4D NumPy array of shape `(frame_count, height, width, 3)` with dtype `uint8`, initialized to zeros [S2, unofficial].
- `image_filter`: A VTK `vtkWindowToImageFilter` instance used to convert the render window to an image buffer [S2, unofficial].

**Steps:**
1. **Initialize**: Obtain the active camera object and configure the rendering window with the specified `resolution` [S1, unofficial][S2, unofficial].
2. **Loop**: Iterate `i` from `0` to `frame_count - 1` [S1, unofficial].
3. **Rotate**:
   - If `rotate_angles` is provided, retrieve `angle = rotate_angles[i]` [S2, unofficial].
   - Otherwise, set `angle = azimuth_increment` [S1, unofficial].
   - Apply the rotation to the camera using the `rotate_axis` specification. For azimuth rotation (axis 2), call `camera.Azimuth(angle)` [S1, unofficial][S2, unofficial].
4. **Render**: Trigger the rendering pipeline. If using VTK, call `image_filter.Modified()` and `image_filter.Update()` to refresh the image buffer, then `window.Render()` [S2, unofficial].
5. **Capture**: Extract the rendered image data. In VTK, convert the output of `image_filter.GetOutput()` to a NumPy array [S2, unofficial].
6. **Store**: Assign the captured image to `rendered_image[i, :, :, :]` [S2, unofficial].
7. **Cleanup (Optional)**: If the rotation is cumulative and needs to be reset for the next frame without accumulating error, apply the inverse rotation `camera.Azimuth(-angle)` after capture [S2, unofficial]. *Note: In standard orbit generation, rotations are often cumulative, so this step may be omitted depending on the desired continuity.*
8. **Save**: Write the frame to `output_dir` with a sequential filename (e.g., `frame_%03d.png`) [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import os

def generate_orbit_frames(active_camera, frame_count, azimuth_increment, 
                          rotate_axis, resolution, output_dir, rotate_angles=None):
    """
    Generates orbit camera frames.
    active_camera: Object with Azimuth(degrees) method.
    frame_count: int
    azimuth_increment: float (degrees)
    rotate_axis: int (0=Elevation, 1=Roll, 2=Azimuth)
    resolution: tuple (width, height)
    output_dir: str
    rotate_angles: list of floats or None
    """
    os.makedirs(output_dir, exist_ok=True)
    
    # Initialize image buffer if using VTK-style capture
    # Note: Actual capture mechanism depends on the renderer API.
    # This pseudocode assumes a generic capture function `capture_frame`.
    
    for i in range(frame_count):
        if rotate_angles is not None:
            angle = rotate_angles[i]
        else:
            angle = azimuth_increment
            
        # Apply rotation based on axis
        # Simplified for Azimuth (axis 2) as per common usage [S1, unofficial][S2, unofficial]
        if rotate_axis == 2:
            active_camera.Azimuth(angle)
        elif rotate_axis == 0:
            active_camera.Elevation(angle)
        elif rotate_axis == 1:
            active_camera.Roll(angle)
            
        # Render and Capture
        # [G] Implementation detail: Trigger renderer and extract pixel buffer
        frame = capture_frame(resolution)
        
        # Save frame
        filename = os.path.join(output_dir, f"frame_{i:03d}.png")
        save_image(frame, filename)
        
        # Optional: Reset rotation if non-cumulative behavior is required [S2, unofficial]
        # active_camera.Azimuth(-angle) 
```

**Parameters:**
- `azimuth_increment`: Typical value 10 degrees [S1, unofficial].
- `frame_count`: Typical value 36 [S1, unofficial].
- `resolution`: Typical value [1000, 800] [S1, unofficial].
- `rotate_axis`: 0 (Elevation), 1 (Roll), 2 (Azimuth) [S2, unofficial].

**Edge Cases:**
- **Cumulative Error**: Repeated floating-point rotations can accumulate orientation drift. Using `rotate_angles` with explicit values or resetting the camera orientation periodically mitigates this [S2, unofficial].
- **Axis Ambiguity**: The `rotate_axis` parameter must be consistently interpreted by the camera API. VTK uses 0, 1, 2 for Elevation, Roll, Azimuth respectively [S2, unofficial].

## Subtopics

### [IM-3D](8092-im-3d.md)
A method for 3D reconstruction that uses a fine-tuned image-to-video model to generate turn-table style video sequences (orbit paths) from a single image, which are then used for 3D Gaussian Splatting. It differs by generating the path content via a video model rather than geometric camera transforms.

## Operating characteristics
- **Time Complexity**: $O(F)$, where $F$ is the number of frames, as each frame requires one rotation and one render operation [S1, unofficial][S2, unofficial].
- **Memory Complexity**: $O(F \cdot H \cdot W \cdot C)$ if storing all frames in memory, or $O(H \cdot W \cdot C)$ if streaming to disk [S2, unofficial].
- **Performance**: Rendering time depends on the scene complexity and resolution. Typical resolution [1000, 800] with 36 frames is feasible for real-time preview but may require batch processing for high-res volume rendering [S1, unofficial].
- **Strengths**: Simple to implement; produces smooth, continuous rotation ideal for visualizing symmetric or centrally focused data [S1, unofficial].
- **Limitations**: Does not provide elevation changes (unlike Spiral) or aspect variation (unlike Elliptical), potentially hiding features located above or below the equatorial plane [S2, unofficial].

## Background
The method is commonly implemented in scientific visualization pipelines using VTK and ParaView. It is utilized for exporting animation sequences (e.g., PNG frames) that can be stitched into videos using tools like ffmpeg [S1, unofficial]. The approach is foundational in volume rendering workflows for creating turn-table style visualizations [S2, unofficial].

## Sources
[S1] cjcamp16/sciviz-paraview, https://github.com/cjcamp16/sciviz-paraview, fetched 2026-10-26 (unofficial)
[S2] Volume Renderer with VTK - yuta-hi, https://github.com/yuta-hi/volume-renderer, fetched 2026-10-29 (unofficial)
