# Pinhole Camera Ray Generation

Also called: Pinhole Camera, Pinhole Ray Casting

## Overview
The Pinhole Camera Ray Generation method computes the origin and direction vectors for viewing rays corresponding to each pixel in a raster image, assuming an ideal pinhole camera model where all rays pass through a single center point [S9, unofficial]. It solves the problem of mapping 2D screen coordinates to 3D world-space rays for volume rendering or ray tracing [S5, unofficial]. Unlike [Camera Matrix Ray Generation](2172-camera-matrix-ray-generation.md), which relies on explicit intrinsic and extrinsic matrices, this method often derives rays directly from geometric primitives such as field of view (FOV), aspect ratio, and camera basis vectors, or uses a simplified look-at formulation [S1, S2, S9, unofficial]. It provides no depth of field, ensuring all objects are in focus [S9, unofficial].

## How it works
**Inputs:**
*   Image dimensions: width $W$, height $H$ [S1, S9, unofficial].
*   Camera position (origin): $\mathbf{p}$ [S9, unofficial].
*   Camera orientation: defined by a forward vector $\mathbf{f}$ (look-at direction), a world up vector $\mathbf{u}_{world}$, and optionally a rotation matrix $\mathbf{R}$ [S1, S9, unofficial].
*   Field of View: Vertical FOV $\theta_{v}$ (in radians or degrees) or Horizontal FOV $\theta_{h}$ [S1, S9, unofficial].
*   Focal length $f$ or focal distance (distance from pinhole to image plane) [S2, S9, unofficial].

**Outputs:**
*   For each pixel $(i, j)$ where $i \in [0, W-1]$ and $j \in [0, H-1]$:
    *   Ray origin $\mathbf{o}_{i,j} = \mathbf{p}$ [S9, unofficial].
    *   Ray direction $\mathbf{d}_{i,j}$ (normalized) [S9, unofficial].

**Data Structures:**
*   Ray arrays: Two contiguous arrays of size $W \times H \times 3$ storing origins and directions as floating-point vectors [S7, unofficial].
*   Camera Basis: Three orthonormal vectors $\mathbf{w}$ (backward/forward), $\mathbf{u}$ (right), $\mathbf{v}$ (up) forming the local camera coordinate system [S2, S7, unofficial].

**Steps:**

1.  **Compute Camera Basis Vectors:**
    Define the camera's local coordinate system. The camera looks along the negative Z-axis in its local space, which maps to the forward vector in world space.
    $$ \mathbf{w} = \text{normalize}(\mathbf{p} - \mathbf{target}) \quad \text{or} \quad \mathbf{w} = -\text{normalize}(\mathbf{target} - \mathbf{p}) $$
    $$ \mathbf{u} = \text{normalize}(\text{cross}(\mathbf{u}_{world}, \mathbf{w})) $$
    $$ \mathbf{v} = \text{cross}(\mathbf{w}, \mathbf{u}) $$
    Here, $\mathbf{w}$ points from the target to the eye (or negative forward), $\mathbf{u}$ is the right vector, and $\mathbf{v}$ is the up vector [S2, S7, S9, unofficial].

2.  **Determine View Plane Dimensions:**
    Calculate the physical extent of the image plane at a unit focal distance or based on the specified FOV.
    If using FOV directly:
    $$ \text{viewport\_height} = 2 \cdot \tan\left(\frac{\theta_v}{2}\right) $$
    $$ \text{viewport\_width} = \text{viewport\_height} \cdot \frac{W}{H} $$
    Alternatively, if using focal length $f$:
    $$ \text{viewport\_height} = \frac{H}{2f} \quad \text{(scaled appropriately)} $$
    [S9, unofficial] defines the viewport at unit focal distance. [S2, unofficial] derives focal distance from FOV: $f_{dist} = \frac{1.0}{2 \tan(\theta_v \pi / 360)}$.

3.  **Compute Pixel Offsets:**
    Map pixel indices to coordinates on the view plane. Two common conventions exist:
    
    *Convention A (Centered, Normalized):*
    Map pixel $(i, j)$ to Normalized Device Coordinates (NDC) or a centered offset.
    $$ x_{ndc} = \frac{2(i + 0.5)}{W} - 1 $$
    $$ y_{ndc} = 1 - \frac{2(j + 0.5)}{H} $$
    (Note: $y$ is often inverted depending on whether image coordinates increase downwards or upwards [S7, S9, unofficial]).
    
    *Convention B (Tangent Division):*
    Sample the FOV by dividing the tangent of the half-FOV by the resolution [S1, unofficial].
    $$ \Delta x = \frac{\tan(\theta_h / 2)}{W} $$
    $$ \Delta y = \frac{\tan(\theta_v / 2)}{H} $$
    $$ x_{local} = (i - \frac{W}{2}) \cdot \Delta x $$
    $$ y_{local} = (\frac{H}{2} - j) \cdot \Delta y $$

4.  **Construct Local Ray Direction:**
    Combine the offsets with the camera basis.
    Using Convention A and Viewport dimensions:
    $$ \mathbf{d}_{local} = \mathbf{w} + x_{ndc} \cdot \frac{\text{viewport\_width}}{2} \cdot \mathbf{u} + y_{ndc} \cdot \frac{\text{viewport\_height}}{2} \cdot \mathbf{v} $$
    Using Convention B (Tangent):
    $$ \mathbf{d}_{local} = \Delta y \cdot \mathbf{v} + \Delta x \cdot \mathbf{u} + \mathbf{w} $$
    [S1, unofficial] constructs `rayd << dv * v, dh * h, 1.0` where $h, v$ are loop indices centered at 0.

5.  **Normalize and Transform:**
    Normalize the direction vector to ensure unit length. If the basis vectors are already orthonormal and the forward vector is $\mathbf{w}$, the resulting vector from the linear combination in step 4 is generally not unit length before normalization if offsets are large, or is unit length if constructed via rotation matrix [S1, unofficial].
    $$ \mathbf{d}_{world} = \text{normalize}(\mathbf{d}_{local}) $$
    If a rotation matrix $\mathbf{R}$ is provided instead of explicit basis vectors:
    $$ \mathbf{d}_{world} = \mathbf{R} \cdot \mathbf{d}_{local} $$
    [S1, unofficial] applies `rayd = rot * rayd`.

6.  **Handle Anti-Aliasing (Optional):**
    To reduce aliasing, jitter the pixel coordinates by adding a random value $\epsilon \in [0, 1)$ to $i$ and $j$ before computing offsets [S9, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def generate_pinhole_rays(width, height, position, target, world_up, fov_degrees):
    """
    Generates ray origins and directions for a pinhole camera.
    
    Args:
        width (int): Image width in pixels.
        height (int): Image height in pixels.
        position (np.array): Camera origin (3,).
        target (np.array): Look-at point (3,).
        world_up (np.array): World up vector (3,).
        fov_degrees (float): Vertical field of view in degrees.
        
    Returns:
        origins (np.array): Shape (H, W, 3).
        directions (np.array): Shape (H, W, 3).
    """
    # 1. Compute Camera Basis
    # Forward vector points from camera to target
    forward = target - position
    forward = forward / np.linalg.norm(forward)
    
    # Right vector is perpendicular to forward and world_up
    right = np.cross(forward, world_up)
    right = right / np.linalg.norm(right)
    
    # Up vector is perpendicular to forward and right
    up = np.cross(right, forward)
    
    # 2. Compute Viewport Dimensions at unit focal distance
    # Using vertical FOV
    fov_rad = math.radians(fov_degrees)
    viewport_height = 2.0 * math.tan(fov_rad / 2.0)
    aspect_ratio = width / height
    viewport_width = viewport_height * aspect_ratio
    
    # 3. Create Pixel Grid
    # Create meshgrid for x and y coordinates
    # x corresponds to columns (0 to W-1), y to rows (0 to H-1)
    # Using half-pixel offset for centering
    x_coords = np.arange(width) + 0.5
    y_coords = np.arange(height) + 0.5
    X, Y = np.meshgrid(x_coords, y_coords, indexing='xy')
    
    # 4. Map to NDC [-1, 1]
    # X: -1 to 1, Y: 1 to -1 (assuming image y goes down, but NDC y goes up)
    # Or simply map to viewport coordinates directly
    # Let's map to viewport coordinates relative to center
    # Center is at (W/2, H/2)
    
    # Offset from center in pixel units
    dx = (X - width / 2.0) / (width / 2.0) # Range approx -1 to 1
    dy = (height / 2.0 - Y) / (height / 2.0) # Range approx -1 to 1 (inverted Y)
    
    # Scale by viewport dimensions
    # The factor 0.5 is included in viewport_width/height definition? 
    # viewport_height = 2 * tan(fov/2). So tan(fov/2) is half-height.
    # dx * (viewport_width / 2) gives the offset in world units at focal distance 1?
    # Let's use the explicit formula from [S9, unofficial]:
    # ray_dir = normalize(forward + ndc.x * (viewport_width / 2.0) * right + ndc.y * (viewport_height / 2.0) * up)
    
    ndc_x = (2.0 * X / width) - 1.0
    ndc_y = 1.0 - (2.0 * Y / height)
    
    # 5. Compute Directions
    # Vector addition in world space
    # Note: forward is the Z-axis of the camera (looking down -Z in local, but here forward is the direction vector)
    # In the basis {right, up, forward}, the ray direction is:
    # d = forward + ndc_x * (vw/2) * right + ndc_y * (vh/2) * up
    
    dir_x = ndc_x * (viewport_width / 2.0) * right
    dir_y = ndc_y * (viewport_height / 2.0) * up
    dir_z = forward
    
    directions = dir_x + dir_y + dir_z
    directions = directions / np.linalg.norm(directions, axis=-1, keepdims=True)
    
    # 6. Origins
    # All rays originate from the camera position
    origins = np.broadcast_to(position, (height, width, 3))
    
    return origins, directions
```

**Parameters with Typical Values:**
*   `fov_degrees`: 60° (telephoto/normal), 90° (wide), 120° (very wide) [S9, unofficial].
*   `near`, `far`: Depth bounds for ray marching, typically `near=2.0`, `far=6.0` for synthetic datasets [S4, unofficial].
*   `world_up`: Usually `(0, 1, 0)` [S9, unofficial].

**Edge Cases and Pitfalls:**
*   **Degenerate Up Vector:** If `world_up` is parallel to `forward`, the cross product for `right` becomes zero. Guard against this by checking the dot product or using a default up vector if alignment is detected [G].
*   **Precision:** On some mobile Intel Graphics 6xx chipsets, compute shaders generating these rays may fail silently or throw AccessViolationException, resulting in a purple screen [S5, unofficial].
*   **FOV Definition:** Sources disagree on whether FOV is horizontal or vertical. Most modern engines (e.g., Godot, [S9, unofficial]) use Vertical FOV. Ensure consistency with the rendering API [S9, unofficial].

## Subtopics

### Pinhole Camera with Depth of Field
Also known as Thin Lens Camera. This variant introduces a finite aperture to simulate realistic lens behavior, creating a focal plane where objects are sharp and others are blurred (bokeh).
*   **What it is:** Extends the pinhole model by sampling ray origins from a disk (the lens) rather than a single point.
*   **What sets it apart:** It requires additional parameters: `aperture` (lens diameter) and `focus_distance`. The ray origin is offset by a random vector on the lens disk, and the direction is calculated towards a focus point on the focal plane.
*   **When to choose it:** When realistic blur effects are required for depth perception or artistic style.
*   **Link:** [Pinhole Camera with Depth of Field](3222-pinhole-camera-with-depth-of-field.md) (Hypothetical link as no article exists, but described here).

## Operating characteristics
*   **Time Complexity:** $O(W \cdot H)$ to generate rays for all pixels. The computation per pixel is constant time (vector operations) [S1, S9, unofficial].
*   **Memory Complexity:** $O(W \cdot H)$ to store the ray origins and directions arrays [S7, unofficial].
*   **Measured Performance:** Generating rays is generally fast and CPU/GPU parallelizable. However, implementation errors (e.g., incorrect basis vectors) lead to completely incorrect rendering. The method is efficient compared to matrix inversion methods if the basis is precomputed [S3, unofficial].
*   **Strengths:** Simple to implement; intuitive geometric interpretation; no need for explicit intrinsic matrix inversion if FOV is known [S9, unofficial].
*   **Limitations:** Does not account for lens distortion, chromatic aberration, or depth of field (unless extended) [S9, unofficial].
*   **When to use:** Standard volume rendering, ray marching, and simple ray tracing where ideal perspective projection is sufficient.
*   **When not:** When using calibrated real-world camera data with known intrinsics/extrinsics (use [Camera Matrix Ray Generation](2172-camera-matrix-ray-generation.md)) or when depth of field is required.

## Background
The pinhole camera model is the simplest mathematical model of image formation, dating back to the camera obscura [S9, unofficial]. In computer graphics, it is the standard assumption for perspective projection. The specific implementation of generating rays by sampling the tangent of the FOV equally [S1, unofficial] is a common technique in ray tracers to ensure uniform angular sampling. The relationship between focal length and FOV, $fov = 2 \cdot \arctan(\frac{sensor\_height}{2 \cdot focal\_length})$, is derived from basic trigonometry [S9, unofficial]. A 50mm lens on a 35mm sensor yields approximately 39.6° FOV, considered a "normal" lens [S9, unofficial].

## Sources
[S1] RMonica/basic_next_best_view, https://github.com/RMonica/basic_next_best_view, fetched 2026-10-24 (unofficial)
[S2] CSE-333-Computer-Graphics-2020/VolumeRayCasting, https://github.com/CSE-333-Computer-Graphics-2020/VolumeRayCasting, fetched 2026-10-26 (unofficial)
[S3] google-research/multinerf, https://github.com/google-research/multinerf, fetched 2026-10-26 (unofficial)
[S4] kwea123/nerf_pl, https://github.com/kwea123/nerf_pl, fetched 2026-10-26 (unofficial)
[S5] Gomosuza/ray-marching, https://github.com/Gomosuza/ray-marching, fetched 2026-10-26 (unofficial)
[S6] AlexSheldrick/density_bounded_ray_weights, https://github.com/AlexSheldrick/density_bounded_ray_weights, fetched 2026-10-27 (unofficial)
[S7] YueJiang-nj/CVPR2020-SDFDiff, https://github.com/YueJiang-nj/CVPR2020-SDFDiff, fetched 2026-10-27 (unofficial)
[S8] ziplab/VolSplat, https://github.com/ziplab/VolSplat, fetched 2026-10-27 (unofficial)
[S9] Camera Model in Path Tracing - Pinhole, Lens, and DOF Guide | Dev ..., https://code-note-vr.vercel.app/pathtracer-learning-camera-model, fetched 2026-10-29 (unofficial)
[S10] GitHub - apchenstu/TensoRF: [ECCV 2022] Tensorial Radiance Fields, a ..., https://github.com/apchenstu/TensoRF, fetched 2026-10-29 (unofficial)
[S11] hexhowells/GoTracer, https://github.com/hexhowells/GoTracer, fetched 2026-10-26 (unofficial)
