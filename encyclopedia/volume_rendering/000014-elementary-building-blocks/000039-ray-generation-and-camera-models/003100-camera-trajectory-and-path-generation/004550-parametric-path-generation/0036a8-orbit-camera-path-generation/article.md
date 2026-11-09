# Orbit Camera Path Generation

Also called: Cyclic rotation rendering, Orbit rendering, Rotation Rendering

## Overview
Orbit Camera Path Generation creates a sequence of camera poses by rotating the viewpoint around a fixed axis to capture an object from multiple angles, primarily for video export or turn-table visualization [S1, unofficial]. It differs from [Spiral Camera Path Generation](1635-spiral-camera-path-generation.md) and [Elliptical Camera Path Generation](7627-elliptical-camera-path-generation.md) by maintaining a constant radius and a single-axis rotation constraint, ensuring a planar circular trajectory rather than a helical or elliptical one.

## How it works
**Inputs and Outputs:**
*   **Input:** A rendering window/camera context, a list of rotation angles `rotate_angles` (floats in degrees), and an axis selector `rotate_axis` (integer 0, 1, or 2).
*   **Output:** A 4D NumPy array `rendered_image` of shape `(N, height, width, 3)` containing uint8 RGB frames, where $N$ is the number of angles.

**Data Structures:**
*   `rotate_axis`: Integer. 0 = Elevation, 1 = Roll, 2 = Azimuth [S1, unofficial].
*   `rotate_angles`: List of floats.
*   `rendered_image`: `np.zeros((len(rotate_angles), size[1], size[0], 3), np.uint8)` [S1, unofficial].

**Steps:**
1.  Initialize a `vtkWindowToImageFilter` instance connected to the rendering window.
2.  Iterate through each angle $\alpha$ in `rotate_angles` with index $i$:
    a. Apply rotation to the camera using `self._camera.Azimuth(angle)` (Note: The source code uses `Azimuth` regardless of the `rotate_axis` definition, implying a specific implementation detail or potential bug in the reference [S1, unofficial]).
    b. Trigger a render update via `image_filter.Modified()` and `self._window.Render()`.
    c. Extract the image buffer using `image_filter.GetOutput()`.
    d. Convert the VTK output to a NumPy array `_image_to_numpy(...)`.
    e. Store the result in `rendered_image[i, :]`.
    f. **Crucial Step:** Reverse the rotation immediately using `self._camera.Azimuth(-angle)` to reset the camera state for the next iteration [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import vtk

def generate_orbit_path(camera, window, rotate_angles, size):
    """
    Generates orbit camera path frames.
    Note: This implementation follows the specific logic of [S1, unofficial], 
    which uses Azimuth for rotation and resets the camera after each frame.
    """
    # Initialize filter
    image_filter = vtk.vtkWindowToImageFilter()
    image_filter.SetInput(window)
    image_filter.Update()

    # Allocate output array: (num_frames, height, width, channels)
    # size[0] is width, size[1] is height
    rendered_image = np.zeros((len(rotate_angles), size[1], size[0], 3), dtype=np.uint8)

    for i, angle in enumerate(rotate_angles):
        # Rotate camera
        # Note: Source [S1, unofficial] uses Azimuth for all rotations despite rotate_axis definition
        camera.Azimuth(angle)
        
        # Render
        image_filter.Modified()
        window.Render()
        
        # Extract image
        # Helper function _image_to_numpy is assumed to convert VTK image to numpy array
        frame = _image_to_numpy(image_filter.GetOutput())
        
        # Store frame
        rendered_image[i, :] = frame
        
        # Reset camera to original orientation for next step
        camera.Azimuth(-angle)

    return rendered_image

def _image_to_numpy(vtk_image):
    """
    Converts a VTK image data object to a numpy array.
    Implementation details omitted as per [S1, unofficial] reference.
    """
    # [G] Standard VTK to NumPy conversion:
    # 1. Get scalar data
    # 2. Reshape to (height, width, channels)
    # 3. Cast to uint8
    pass
```

**Edge Cases and Pitfalls:**
*   **Camera State Reset:** The source explicitly resets the camera rotation after each frame (`Azimuth(-angle)`) [S1, unofficial]. Failure to do so will cause cumulative rotation, breaking the orbit path.
*   **Axis Mismatch:** The source defines `rotate_axis` (0=Elevation, 1=Roll, 2=Azimuth) but the code snippet only calls `Azimuth` [S1, unofficial]. Implementers must map `rotate_axis` to the correct VTK method (`Elevation`, `Roll`, or `Azimuth`) if generalizing [S1, unofficial].
*   **Duplicate Points:** As per [Parametric Path Generation](11476-parametric-path-generation.md) principles, the final angle should not duplicate the start angle to avoid redundant frames in a closed loop.

## Subtopics

### IM-3D
A method that fine-tunes an image-to-video model based on Emu to generate turn-table-like videos rotating around an object, which are then used as input for 3D Gaussian Splatting (3DGS) reconstruction [S2]. It differs from standard orbit rendering by generating synthetic multi-view images via diffusion models rather than rendering a pre-existing 3D scene [S2].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot R)$, where $N$ is the number of frames and $R$ is the cost of a single render pass.
*   **Memory Complexity:** $O(N \cdot H \cdot W \cdot C)$ for storing the output video frames [S1, unofficial].
*   **Strengths:** Simple to implement; produces consistent, planar rotation ideal for object inspection [S1, unofficial].
*   **Limitations:** Limited to single-axis rotation; cannot capture complex viewpoints like spirals or ellipses [S1, unofficial].
*   **Hardware:** Dependent on the rendering backend (VTK in source [S1, unofficial]).

## Background
Originating from VTK-based volume rendering implementations, this method is used for generating turn-table videos for visualization and 3D reconstruction pipelines [S1, unofficial]. It has been adapted in modern 3D generation workflows, such as IM-3D, to synthesize training data for Gaussian Splatting [S2].

## Sources
[S1] Volume Renderer with VTK - yuta-hi, https://github.com/yuta-hi/volume-renderer, fetched 2026-10-29 (unofficial)
[S2] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
