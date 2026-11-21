# NDC-to-World Ray Generation

Also called: Camera Raycasting, View Space Ray Origin

## Overview
NDC-to-World Ray Generation computes the world-space origin of a ray for a given Normalized Device Coordinate (NDC) pixel, assuming a pinhole camera model where the camera is at the origin in view space, facing the positive Z axis, with a near plane at $Z=1$ [S1, unofficial]. It differs from sibling methods like [Pinhole Camera Projection (Pixel to Camera)](3162-pinhole-camera-projection-pixel-to-camera.md) by explicitly constructing the ray origin in world space based on the near-plane geometry defined by the field of view and aspect ratio, rather than just transforming pixel coordinates to camera space [S1, unofficial]. A common pitfall in implementation is incorrectly offsetting the ray origin by screen-space coordinates, which causes field-of-view distortion and curvature artifacts [S2, unofficial].

## How it works
**Inputs:**
- `ndc_x`, `ndc_y`: Normalized Device Coordinates of the pixel, typically in the range $[0, 1]$ or $[-1, 1]$. The source assumes $[0, 1]$ based on the formula $(ndc\_x - 0.5)$ [S1, unofficial].
- `half_fov_rad_x`: Horizontal half-field of view in radians [S1, unofficial].
- `aspect`: Screen aspect ratio (width / height) [S1, unofficial].
- `camera_pos`: World-space position of the camera [S1, unofficial].
- `camera_basis`: Camera orientation defined by forward, up, and right vectors [S1, unofficial].

**Outputs:**
- `world_pos`: The world-space origin of the ray [S1, unofficial].
- `ray_dir`: The direction of the ray in world space (implied by the context of ray generation, though the specific formula in [S1, unofficial] focuses on the origin; the direction is typically normalized from the view-space direction).

**Data Structures:**
- `Vector3d`: A 3D vector structure with fields `x`, `y`, `z` [S1, unofficial].
- `Camera`: Contains `camera_pos` and basis vectors `forward`, `up`, `right` [S1, unofficial].

**Steps:**
1. **Initialize Camera Basis:** Define the default camera basis vectors in view space:
   - `forward = (0, 0, 1)`
   - `up = (0, 1, 0)`
   - `right = cross(up, forward)` [S1, unofficial]
2. **Rotate Basis (Optional):** If the camera is rotated, update the basis vectors by rotating them around the specified axis by the given angle in degrees. Each vector is updated independently using the corresponding rotation function [S1, unofficial].
3. **Calculate Near Plane Dimensions:**
   - `near_plane_width = 2.0 * tan(half_fov_rad_x)`
   - `near_plane_height = near_plane_width / aspect` [S1, unofficial]
4. **Calculate View-Space Position:**
   - `view_pos = forward + right * (ndc_x - 0.5) * near_plane_width + up * (ndc_y - 0.5) * near_plane_height` [S1, unofficial]
   - Note: This formula assumes the NDC coordinates are in $[0, 1]$. If NDC is in $[-1, 1]$, the offset term $(ndc - 0.5)$ should be adjusted to $(ndc / 2.0)$.
5. **Transform to World Space:**
   - `world_pos = view_pos + camera_pos` [S1, unofficial]
   - Note: This assumes `view_pos` is already in world space or that the basis vectors are in world space. If `view_pos` is in view space, it must be rotated by the camera's orientation matrix before adding `camera_pos`. The source [S1, unofficial] implies `view_pos` is added directly to `camera_pos`, suggesting `forward`, `right`, and `up` are world-space vectors after rotation.

**Formulas:**
- $near\_plane\_width = 2.0 \cdot \tan(\text{half\_fov\_rad\_x})$
- $near\_plane\_height = \frac{near\_plane\_width}{\text{aspect}}$
- $\text{view\_pos} = \text{forward} + \text{right} \cdot (\text{ndc\_x} - 0.5) \cdot near\_plane\_width + \text{up} \cdot (\text{ndc\_y} - 0.5) \cdot near\_plane\_height$
- $\text{world\_pos} = \text{view\_pos} + \text{camera\_pos}$ [S1, unofficial]

**Parameters:**
- `half_fov_rad_x`: Horizontal half-field of view in radians. Derived from degrees via $(\text{deg} \cdot \pi) / 180.0$ [S1, unofficial].
- `aspect`: Screen aspect ratio. Used to scale near plane height: $near\_plane\_height = near\_plane\_width \cdot (1.0 / \text{aspect})$ [S1, unofficial].

**Edge Cases and Pitfalls:**
- **NDC Range:** The formula assumes NDC coordinates are in $[0, 1]$. If the NDC range is $[-1, 1]$, the offset $(ndc - 0.5)$ must be changed to $(ndc / 2.0)$ [S1, unofficial].
- **Ray Origin Distortion:** A common error is adding screen-space coordinates (e.g., `vec3(x, y, 0)`) to the ray origin in addition to the direction vector. This results in an incorrect effective FOV that varies with pixel position, leading to distortion and curvature artifacts, especially when the camera is close to objects [S2, unofficial]. The correct approach is to keep the ray origin at the camera position (or world origin relative to camera) and vary only the direction based on fragment coordinates [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def ndc_to_world_ray(ndc_x, ndc_y, half_fov_rad_x, aspect, camera_pos, camera_forward, camera_up, camera_right):
    """
    Computes the world-space ray origin for a given NDC pixel.
    
    Args:
        ndc_x (float): NDC x coordinate in [0, 1].
        ndc_y (float): NDC y coordinate in [0, 1].
        half_fov_rad_x (float): Horizontal half-field of view in radians.
        aspect (float): Screen aspect ratio (width / height).
        camera_pos (Vector3): World-space camera position.
        camera_forward (Vector3): World-space forward vector.
        camera_up (Vector3): World-space up vector.
        camera_right (Vector3): World-space right vector.
        
    Returns:
        Vector3: World-space ray origin.
    """
    # Step 1: Calculate near plane dimensions
    near_plane_width = 2.0 * math.tan(half_fov_rad_x)
    near_plane_height = near_plane_width / aspect
    
    # Step 2: Calculate view-space position relative to camera center
    # Assuming NDC is in [0, 1], center is at (0.5, 0.5)
    view_pos = (camera_forward + 
                camera_right * (ndc_x - 0.5) * near_plane_width + 
                camera_up * (ndc_y - 0.5) * near_plane_height)
    
    # Step 3: Transform to world space
    world_pos = view_pos + camera_pos
    
    return world_pos
```

## Subtopics

### Correct Camera Ray Origin
A correction for raymarching where the ray origin is incorrectly initialized with screen-space pixel coordinates, causing field-of-view distortion and curvature artifacts when the camera is close to objects [S2, unofficial]. The correct method initializes the ray origin `ray_pos` to the camera position `cam_pos` only, without adding screen-space coordinates [S2, unofficial]. Specifically, one should not add the screen-space vector `vec3(x, y, 0)` to the ray origin before adding the ray direction `r` [S2, unofficial]. Adding `vec3(x, y, 0)` to the ray origin while also adding the direction vector `r` (which already contains screen-space information) results in an incorrect effective FOV that varies with pixel position, leading to distortion [S2, unofficial]. Standard raymarching leaves the ray origin at the camera position (or world origin relative to camera) and varies only the direction based on fragment coordinates, whereas the flawed approach adds pixel offsets to the origin [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ per ray, as it involves a constant number of arithmetic operations [S1, unofficial].
- **Memory Complexity:** $O(1)$, requiring only a few temporary variables for the calculation [S1, unofficial].
- **Strengths:** Simple and efficient calculation for generating ray origins in a pinhole camera model [S1, unofficial].
- **Limitations:** Assumes a specific NDC range ($[0, 1]$) and camera orientation (positive Z) [S1, unofficial]. Incorrect implementation can lead to significant visual artifacts [S2, unofficial].
- **When to Use:** When implementing a ray tracer or ray marcher with a pinhole camera model and NDC coordinates in the range $[0, 1]$ [S1, unofficial].
- **When Not to Use:** When the camera model is not a pinhole camera, or when NDC coordinates are in a different range (e.g., $[-1, 1]$) without adjustment [S1, unofficial].

## Background
The method is derived from the implementation of a terminal-based signed distance field renderer [S1, unofficial]. The specific pitfall of adding screen-space coordinates to the ray origin was identified in a discussion on avoiding curvature artifacts in raytracers [S2, unofficial]. The source [S1, unofficial] provides the core formula for calculating the ray origin based on NDC coordinates, field of view, and aspect ratio. The source [S2, unofficial] highlights the common mistake of offsetting the ray origin, which leads to distortion.

## Sources
[S1] JamesRicher/TerminalSignedDistanceFieldRenderer, https://github.com/JamesRicher/TerminalSignedDistanceFieldRenderer, fetched 2026-10-27 (unofficial)
[S2] How to avoid curvature artifact in raytracer?, https://computergraphics.stackexchange.com/questions/11072/how-to-avoid-curvature-artifact-in-raytracer, fetched 2026-10-27 (unofficial)
