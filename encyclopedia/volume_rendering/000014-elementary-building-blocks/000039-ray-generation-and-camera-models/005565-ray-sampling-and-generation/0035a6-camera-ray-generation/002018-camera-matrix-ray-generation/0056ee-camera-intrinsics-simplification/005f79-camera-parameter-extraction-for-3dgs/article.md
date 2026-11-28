# Camera Parameter Extraction for 3DGS


## Overview
This method extracts the intrinsic camera parameters required for 3D Gaussian Splatting (3DGS) rendering from a source scene's camera definition. It converts raw sensor data, lens focal length, and resolution settings into the canonical 3DGS format: horizontal and vertical field of view angles ($\alpha_x, \alpha_y$), focal lengths in pixels ($f_x, f_y$), and principal point offsets ($c_x, c_y$). It differs from sibling methods by handling the full pipeline of sensor fit mode resolution and pixel aspect ratio correction, which are prerequisites for the simplified FOV calculations performed by [Camera Intrinsics Simplification](2271-camera-intrinsics-simplification.md).

## How it works
**Inputs:**
*   `scene`: Object containing render settings (`resolution_x`, `resolution_y`, `resolution_percentage`, `pixel_aspect_x`, `pixel_aspect_y`).
*   `camera`: Object containing camera settings (`lens`, `sensor_width`, `sensor_height`, `sensor_fit`, `shift_x`, `shift_y`, `type`).

**Outputs:**
*   Dictionary with keys: `camera_angle_x` (float), `camera_angle_y` (float), `fl_x` (float), `fl_y` (float), `cx` (float), `cy` (float), `w` (int), `h` (int).

**Data Structures:**
*   The output is a flat dictionary. All float values are standard 64-bit floating-point numbers. Integers are standard 32-bit integers.

**Steps:**

1.  **Validate Camera Type:**
    Check if `camera.type` is `"PERSP"`. If not, raise a `DatasetBuildError` with the message `"3DGS export requires a perspective camera."` and abort execution [S1, unofficial].

2.  **Compute Resolution and Aspect Ratio:**
    Calculate the effective image width $w$ and height $h$ in pixels, accounting for the resolution percentage scale factor:
    $$ w = \left\lfloor \text{scene.render.resolution\_x} \times \frac{\text{scene.render.resolution\_percentage}}{100} \right\rfloor $$
    $$ h = \left\lfloor \text{scene.render.resolution\_y} \times \frac{\text{scene.render.resolution\_percentage}}{100} \right\rfloor $$
    Calculate the pixel aspect ratio $p$:
    $$ p = \frac{\text{scene.render.pixel\_aspect\_x}}{\text{scene.render.pixel\_aspect\_y}} $$

3.  **Resolve Sensor Fit Mode:**
    Determine the sensor fit mode $m$. If `camera.sensor_fit` is `"AUTO"`, set $m$ to `"HORIZONTAL"` if $w \times p \ge h$, otherwise set $m$ to `"VERTICAL"`. If `camera.sensor_fit` is explicitly `"HORIZONTAL"` or `"VERTICAL"`, use that value [S1, unofficial].

4.  **Compute Focal Lengths:**
    Let $f$ be `camera.lens`, $s_w$ be `camera.sensor_width`, and $s_h$ be `camera.sensor_height`.
    
    If $m$ is `"VERTICAL"`:
    $$ f_x = \frac{f}{s_h} \times h \times p $$
    $$ f_y = \frac{f}{s_h} \times h $$
    
    Otherwise (if $m$ is `"HORIZONTAL"`):
    $$ f_x = \frac{f}{s_w} \times w $$
    $$ f_y = \frac{f}{s_w} \times \frac{w}{p} $$

5.  **Compute Field of View Angles:**
    Calculate the horizontal field of view $\alpha_x$ and vertical field of view $\alpha_y$ using the arctangent function:
    $$ \alpha_x = 2.0 \times \arctan\left(\frac{w}{2.0 \times f_x}\right) $$
    $$ \alpha_y = 2.0 \times \arctan\left(\frac{h}{2.0 \times f_y}\right) $$

6.  **Compute Principal Point:**
    Calculate the principal point coordinates $(c_x, c_y)$ relative to the image center, incorporating camera shifts:
    $$ c_x = \frac{w}{2.0} - (\text{camera.shift\_x} \times w) $$
    $$ c_y = \frac{h}{2.0} + (\text{camera.shift\_y} \times h) $$

7.  **Return Results:**
    Return the dictionary containing $\alpha_x, \alpha_y, f_x, f_y, c_x, c_y, w, h$.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def build_camera_info(scene, camera):
    # Step 1: Validate Camera Type
    if camera.type != "PERSP":
        raise DatasetBuildError("3DGS export requires a perspective camera.")

    # Step 2: Compute Resolution and Aspect Ratio
    w = int(scene.render.resolution_x * scene.render.resolution_percentage / 100)
    h = int(scene.render.resolution_y * scene.render.resolution_percentage / 100)
    pixel_aspect = scene.render.pixel_aspect_x / scene.render.pixel_aspect_y

    # Step 3: Resolve Sensor Fit Mode
    if camera.sensor_fit == "AUTO":
        if w * pixel_aspect >= h:
            sensor_fit = "HORIZONTAL"
        else:
            sensor_fit = "VERTICAL"
    else:
        sensor_fit = camera.sensor_fit

    # Step 4: Compute Focal Lengths
    fl_x = 0.0
    fl_y = 0.0
    
    if sensor_fit == "VERTICAL":
        fl_x = (camera.lens / camera.sensor_height) * h * pixel_aspect
        fl_y = (camera.lens / camera.sensor_height) * h
    else:
        fl_x = (camera.lens / camera.sensor_width) * w
        fl_y = (camera.lens / camera.sensor_width) * (w / pixel_aspect)

    # Step 5: Compute Field of View Angles
    camera_angle_x = 2.0 * math.atan(w / (2.0 * fl_x))
    camera_angle_y = 2.0 * math.atan(h / (2.0 * fl_y))

    # Step 6: Compute Principal Point
    cx = (w / 2.0) - (camera.shift_x * w)
    cy = (h / 2.0) + (camera.shift_y * h)

    # Step 7: Return Results
    return {
        "camera_angle_x": camera_angle_x,
        "camera_angle_y": camera_angle_y,
        "fl_x": fl_x,
        "fl_y": fl_y,
        "cx": cx,
        "cy": cy,
        "w": w,
        "h": h
    }
```

**Parameters:**
*   `sensor_fit`: String. Values: `"AUTO"`, `"HORIZONTAL"`, `"VERTICAL"`. Determines which sensor dimension scales the lens focal length. Default behavior in `"AUTO"` mode depends on the aspect ratio of the output resolution [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Non-Perspective Cameras:** The method strictly requires a perspective camera. Orthographic or other projection types cause immediate failure [S1, unofficial].
*   **Zero Dimensions:** If `resolution_percentage` is 0 or sensor dimensions are 0, division by zero or zero-width/height images may occur. The sources do not specify explicit guards for zero resolution, but standard floating-point behavior applies [S1, unofficial].
*   **Pixel Aspect Ratio:** If `pixel_aspect_y` is 0, division by zero occurs in step 2. This is not explicitly handled in the source notes [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$. The method performs a fixed number of arithmetic operations and comparisons.
*   **Memory Complexity:** $O(1)$. It returns a small dictionary with 8 fields.
*   **Strengths:** It correctly handles Blender's specific sensor fit logic and pixel aspect ratios, ensuring that the resulting 3DGS camera parameters match the intended field of view and aspect ratio of the source scene.
*   **Limitations:** It is specific to the 3DGS data format. It does not generate extrinsic parameters (position/orientation), which are handled separately.
*   **When to use:** When exporting a dataset for 3D Gaussian Splatting from a 3D software package (specifically Blender, based on the source) that uses sensor-based camera models.
*   **When not to use:** When the camera is not perspective, or when the target rendering engine uses a different intrinsic parameterization (e.g., direct focal length in mm without conversion to pixels/FOV).

## Background
This method is part of the 3DGS dataset building pipeline, specifically designed to bridge the gap between 3D software camera models (like Blender's) and the canonical camera representation used by 3D Gaussian Splatting. The implementation is found in the `3dgs-dataset-builder` repository.

## Sources
[S1] yumehiko/3dgs-dataset-builder, https://github.com/yumehiko/3dgs-dataset-builder, fetched 2026-10-27 (unofficial)
