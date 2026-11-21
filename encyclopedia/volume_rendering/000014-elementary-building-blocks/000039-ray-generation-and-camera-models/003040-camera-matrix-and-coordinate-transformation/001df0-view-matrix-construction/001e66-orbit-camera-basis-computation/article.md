# Orbit Camera Basis Computation

Also called: Orbit Camera, Orbit Camera Control, Orbit Camera Position Calculation, Spinny Camera

## Overview
Orbit Camera Basis Computation calculates an orthonormal camera basis (eye position, forward, right, and up vectors) for a camera that orbits around a fixed target center using spherical coordinates (yaw, pitch, and radius) [S1, unofficial]. It solves the problem of determining the viewing direction and position for interactive 3D navigation, ensuring the camera maintains a stable orientation relative to the world up vector while avoiding singularities [S1, unofficial]. It differs from standard look-at methods by explicitly deriving the eye position from parametric spherical angles rather than accepting a pre-computed position, and from arcball controls by using direct angle-to-position mapping instead of quaternion rotations [S1, unofficial].

## How it works
**Inputs:**
- `target`: 3D vector representing the center of orbit (pivot point).
- `yaw` (or `azimuth`): Horizontal rotation angle in radians.
- `pitch` (or `polar`): Vertical rotation angle in radians.
- `radius` (or `distance`): Distance from `target` to `eye`.
- `worldUp`: Fixed up vector for the world coordinate system (typically `[0, 1, 0]`).
- `pitchLimit`: Maximum absolute pitch angle to prevent gimbal lock or inversion.

**Outputs:**
- `eye`: 3D vector for camera position.
- `forward`: Normalized vector from `eye` to `target`.
- `right`: Normalized vector perpendicular to `forward` and `worldUp`.
- `up`: Normalized vector perpendicular to `forward` and `right`.

**Data Structures:**
- Vectors are 3-component floating-point arrays or structs (`vec3`).
- No persistent state is required for the basis computation itself, though interactive controllers maintain state for `yaw`, `pitch`, and `radius` [S3, unofficial].

**Parameters and Typical Values:**
- `worldUp`: `[0, 1, 0]` [S1, unofficial].
- `pitchLimit`: `1.5533430342749535` radians (~89 degrees) [S1, unofficial].
- `minDistance`: `0.01` [S3, unofficial].
- `maxDistance`: `20.0` [S3, unofficial].
- `initialAzimuth`: `Math.PI / 4` [S3, unofficial].
- `initialPolar`: `Math.PI / 2.6` [S3, unofficial].
- `initialDistance`: `2.4` [S3, unofficial].

**Steps:**

1.  **Clamp Pitch:** Restrict `pitch` to the range `[-pitchLimit, pitchLimit]` to prevent the camera from flipping upside down when crossing the poles [S1, unofficial].
    $$ p_{clamped} = \max(-\text{pitchLimit}, \min(\text{pitchLimit}, \text{pitch})) $$

2.  **Compute Offset Vector:** Calculate the displacement from the target to the eye using spherical coordinate conversion. The offset is derived from `radius`, `yaw`, and `p_clamped` [S1, unofficial].
    $$ \text{offset}_x = \text{radius} \cdot \cos(p_{clamped}) \cdot \sin(\text{yaw}) $$
    $$ \text{offset}_y = \text{radius} \cdot \sin(p_{clamped}) $$
    $$ \text{offset}_z = \text{radius} \cdot \cos(p_{clamped}) \cdot \cos(\text{yaw}) $$
    $$ \text{offset} = [\text{offset}_x, \text{offset}_y, \text{offset}_z] $$

3.  **Compute Eye Position:** Add the offset to the target center [S1, unofficial].
    $$ \text{eye} = \text{target} + \text{offset} $$

4.  **Compute Forward Vector:** Calculate the direction from the eye to the target and normalize it [S1, unofficial].
    $$ \text{forward} = \text{normalize}(\text{target} - \text{eye}) $$

5.  **Compute Right Vector:** Calculate the cross product of `forward` and `worldUp` [S1, unofficial].
    $$ \text{right}_{raw} = \text{cross}(\text{forward}, \text{worldUp}) $$

6.  **Handle Singularity (Pitfall):** If the magnitude of `right_raw` is below a small epsilon (e.g., $10^{-6}$), the forward vector is parallel to the world up vector, causing a zero-length right vector. In this case, set `right` to a stable arbitrary vector, typically `[1, 0, 0]` [S1, unofficial].
    $$ \text{if } \|\text{right}_{raw}\| < 10^{-6}: \quad \text{right} = [1, 0, 0] $$
    $$ \text{else}: \quad \text{right} = \text{normalize}(\text{right}_{raw}) $$

7.  **Compute Up Vector:** Calculate the cross product of `right` and `forward` and normalize it [S1, unofficial].
    $$ \text{up} = \text{normalize}(\text{cross}(\text{right}, \text{forward})) $$

8.  **Return Basis:** Return the tuple `(eye, forward, right, up)` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def orbit_camera_basis(target, yaw, pitch, radius, world_up=None, pitch_limit=None):
    """
    Computes the orthonormal camera basis for an orbit camera.
    
    Args:
        target (tuple): (x, y, z) center of orbit.
        yaw (float): Horizontal angle in radians.
        pitch (float): Vertical angle in radians.
        radius (float): Distance from target to eye.
        world_up (tuple): World up vector. Defaults to (0, 1, 0).
        pitch_limit (float): Max pitch in radians. Defaults to ~89 degrees.
        
    Returns:
        dict: Contains 'eye', 'forward', 'right', 'up' as tuples.
    """
    if world_up is None:
        world_up = (0.0, 1.0, 0.0)
    if pitch_limit is None:
        pitch_limit = 1.5533430342749535  # ~89 degrees

    # Step 1: Clamp pitch
    p = max(-pitch_limit, min(pitch_limit, pitch))
    
    # Step 2: Compute offset vector
    cos_p = math.cos(p)
    sin_p = math.sin(p)
    sin_yaw = math.sin(yaw)
    cos_yaw = math.cos(yaw)
    
    offset_x = radius * cos_p * sin_yaw
    offset_y = radius * sin_p
    offset_z = radius * cos_p * cos_yaw
    
    # Step 3: Compute eye position
    eye_x = target[0] + offset_x
    eye_y = target[1] + offset_y
    eye_z = target[2] + offset_z
    eye = (eye_x, eye_y, eye_z)
    
    # Step 4: Compute forward vector
    fwd_x = target[0] - eye_x
    fwd_y = target[1] - eye_y
    fwd_z = target[2] - eye_z
    
    fwd_len = math.sqrt(fwd_x**2 + fwd_y**2 + fwd_z**2)
    if fwd_len == 0:
        # Pitfall: Eye is at target. Return identity basis or handle error.
        # For robustness, assume forward is -Z or similar, but standard orbit
        # implies distance > 0.
        forward = (0.0, 0.0, -1.0) 
    else:
        forward = (fwd_x / fwd_len, fwd_y / fwd_len, fwd_z / fwd_len)
        
    # Step 5: Compute right vector
    # cross(forward, world_up)
    right_x = forward[1] * world_up[2] - forward[2] * world_up[1]
    right_y = forward[2] * world_up[0] - forward[0] * world_up[2]
    right_z = forward[0] * world_up[1] - forward[1] * world_up[0]
    
    right_len = math.sqrt(right_x**2 + right_y**2 + right_z**2)
    
    # Step 6: Handle singularity
    if right_len < 1e-6:
        right = (1.0, 0.0, 0.0)
    else:
        right = (right_x / right_len, right_y / right_len, right_z / right_len)
        
    # Step 7: Compute up vector
    # cross(right, forward)
    up_x = right[1] * forward[2] - right[2] * forward[1]
    up_y = right[2] * forward[0] - right[0] * forward[2]
    up_z = right[0] * forward[1] - right[1] * forward[0]
    
    up_len = math.sqrt(up_x**2 + up_y**2 + up_z**2)
    if up_len == 0:
        up = (0.0, 1.0, 0.0) # Fallback
    else:
        up = (up_x / up_len, up_y / up_len, up_z / up_len)
        
    return {
        "eye": eye,
        "forward": forward,
        "right": right,
        "up": up
    }
```

## Subtopics

### [Arcball Camera View & Projection Matrix Computation](2210-arcball-camera-view-projection-matrix-computation.md)
Computes the view and perspective projection matrices for an arcball-style camera, handling rotation around a pivot point and avoiding pitch flip artifacts. It differs from standard look-at methods by explicitly managing the camera position via sequential yaw and pitch rotations around a fixed target.

### [Camera Look-at Computation via Rotation](2696-camera-look-at-computation-via-rotation.md)
Computes a camera's look-at point by rotating a forward vector around the camera's local X and Z axes using spherical coordinate angles, then adding the result to the camera position. It solves the problem of determining the viewing direction for an orbit camera based on user input angles, enabling navigation in 3D space.

### [Orbit Camera Controls](3262-orbit-camera-controls.md)
An interactive camera controller that manipulates a camera's pose using spherical coordinates (distance, theta, phi) relative to a target center, enabling orbiting, panning, and zooming operations. It differs from [Arcball Camera View & Projection Matrix Computation](2210-arcball-camera-view-projection-matrix-computation.md) by using explicit spherical angles rather than quaternion-based rotation from mouse vectors, and from [Camera Look-at Computation via Rotation](2696-camera-look-at-computation-via-rotation.md) by maintaining a persistent target center that can be panned independently of the orbit angles [G]. It builds on [Orbit Camera Basis Computation](1749-orbit-camera-basis-computation.md) to derive the final view matrix from the computed pose [G].

### [Spherical Camera Path Generation](3934-spherical-camera-path-generation.md)
Spherical Camera Path Generation creates a sequence of camera poses arranged in a circular orbit around a central target (typically the origin) for 360° video rendering or volume visualization. It differs from general orbit controls by pre-computing a fixed, smooth trajectory of camera-to-world matrices rather than responding to interactive input. It builds on [Orbit Camera Basis Computation](1749-orbit-camera-basis-computation.md) by automating the generation of the full camera-to-world transformation matrix for each frame in the path.

### [Volume Visualization Camera](4116-volume-visualization-camera.md)
The Volume Visualization Camera is a camera controller for volume visualization that computes a view matrix by applying volume scaling, centering, rotation, zoom, and translation transformations. It solves the problem of maintaining user attention on specific focus parts of the volume by dynamically adjusting the model-view matrix based on interaction inputs. Unlike standard zooming which typically scales uniformly around the view center or object origin, this model emphasizes the region near the initial button click by translating the view. It builds on [Orbit Camera Basis Computation](1749-orbit-camera-basis-computation.md) for basis generation but extends it with specific volume-centric transformation logic.

### Auto-fit Camera Distance Calculation
Calculates the camera distance based on the stack aspect ratio and field of view to ensure the entire volume fits in the view [S5, unofficial]. The distance is computed as `half_max_xy / tan(radians(fov_deg * 0.5)) * 1.25`, where `half_max_xy` is half the maximum dimension of the aspect ratio [S5, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ per frame. The computation involves a constant number of trigonometric functions, vector operations, and comparisons.
- **Memory Complexity:** $O(1)$. No large data structures are allocated; only temporary vectors for the basis are needed.
- **Performance:** Extremely fast, suitable for real-time interactive applications. The primary cost is the trigonometric evaluation, which is negligible on modern CPUs/GPUs.
- **Strengths:** Intuitive control for users (orbit, zoom, pan); stable orientation via world-up vector; avoids gimbal lock within the clamped pitch range.
- **Limitations:** Singularities at the poles (pitch $\pm 90^\circ$) require explicit handling (clamping or fallback vectors) [S1, unofficial]. Not suitable for cameras that need to look directly up or down without constraints.
- **When to use:** Interactive 3D viewers, volume visualization tools, and any application requiring a camera that orbits a specific point of interest.
- **When not to use:** First-person shooters (where rotation is local), cinematic camera paths requiring complex keyframing (use [Spherical Camera Path Generation](3934-spherical-camera-path-generation.md) instead), or when the camera needs to pass through the target point.

## Background
The orbit camera model is a standard technique in 3D graphics and visualization, widely used in scientific visualization, CAD viewers, and game engines. It allows users to inspect 3D objects from any angle by rotating around a central pivot. The specific implementation details, such as the pitch limit of ~89 degrees to prevent inversion, are common conventions in interactive viewers [S1, unofficial]. The method is foundational for higher-level camera controllers like those found in React Three Fiber ecosystems [S3, unofficial] and volume rendering libraries [S2, unofficial].

## Sources
[S1] MohammadHossinzehi/sdf-raymarcher, https://github.com/MohammadHossinzehi/sdf-raymarcher, fetched 2026-10-26 (unofficial)
[S2] marceline-cramer/svo-cpu, https://github.com/marceline-cramer/svo-cpu, fetched 2026-10-27 (unofficial)
[S3] volume-rendering.patch, https://github.com/user-attachments/files/28030354/volume-rendering.patch, fetched 2026-10-28 (unofficial)
[S4] Ray Marching and Signed Distance Functions — GLSL in the..., https://www.mysimulator.uk/content/articles/ray-marching-sdf.html, fetched 2026-10-29 (unofficial)
[S5] xwghua/clearvolume-plus, https://github.com/xwghua/clearvolume-plus, fetched 2026-10-26 (unofficial)
