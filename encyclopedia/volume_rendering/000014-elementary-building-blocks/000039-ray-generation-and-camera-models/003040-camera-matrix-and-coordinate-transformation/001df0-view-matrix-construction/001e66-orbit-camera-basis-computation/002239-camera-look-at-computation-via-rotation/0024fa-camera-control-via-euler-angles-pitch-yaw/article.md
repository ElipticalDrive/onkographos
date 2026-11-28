# Camera Control via Euler Angles (Pitch/Yaw)
Also called: 2-axis aircraft rotation, Camera Basis Vector Update from Euler Angles, Camera View Matrix and Vector Generation, Pitch/Yaw Camera Control, Scene Rotation via Elevation and Azimuth, forward-vec, rotate-point

## Overview
This method computes a camera's orientation and view matrix from two rotation angles: yaw (azimuth) and pitch (elevation), supporting movement along local axes [S2, unofficial]. It differs from full 3-axis rotation by excluding roll, reducing computational cost [S4, unofficial]. It builds on [Camera Look-at Computation via Rotation](2696-camera-look-at-computation-via-rotation.md) by explicitly deriving the basis vectors from spherical coordinate angles rather than arbitrary rotation matrices [S2, unofficial][S3, unofficial].

## How it works
**Inputs:**
- Camera position $\mathbf{p} = (p_x, p_y, p_z)$ [G]
- Yaw angle $\theta_y$ (rotation about vertical/world Y axis) [G]
- Pitch angle $\theta_p$ (rotation about local right axis) [S4, unofficial]
- World up vector $\mathbf{u}_{world} = (0, 1, 0)$ [S3, unofficial]
- Optional: Scene center $(x_{cen}, y_{cen}, z_{cen})$ for scene-rotation variants [S1, unofficial]

**Outputs:**
- Forward vector $\mathbf{f}$
- Right vector $\mathbf{r}$
- Up vector $\mathbf{u}$
- View matrix $\mathbf{M}_{view}$

**Data Structures:**
- Vectors are 3D floating-point tuples.
- Angles are typically in degrees, converted to radians for trigonometric functions [G].

**Steps:**

1. **Clamp Pitch:** Limit pitch to prevent gimbal lock or view inversion. Typical range: $[-89^\circ, 89^\circ]$ [S2, unofficial][S4, unofficial].
   $$ \theta_p \leftarrow \text{clamp}(\theta_p, -89^\circ, 89^\circ) $$

2. **Convert to Radians:**
   $$ \theta_y \leftarrow \theta_y \times \frac{\pi}{180} $$
   $$ \theta_p \leftarrow \theta_p \times \frac{\pi}{180} $$

3. **Compute Forward Vector:**
   The forward vector is derived from spherical coordinates, constrained to the XZ plane for yaw and lifted by pitch [S3, unofficial].
   $$ f_x = \cos(\theta_y) \cos(\theta_p) $$
   $$ f_y = \sin(\theta_p) $$
   $$ f_z = \sin(\theta_y) \cos(\theta_p) $$
   $$ \mathbf{f} = (f_x, f_y, f_z) $$
   Normalize $\mathbf{f}$ [S2, unofficial][S3, unofficial].
   *Note: Some implementations define yaw as rotation from the Z-axis towards X, resulting in $f_x = \sin(\theta_y)$ and $f_z = \cos(\theta_y)$ before normalization [S3, unofficial].*

4. **Compute Right Vector:**
   $$ \mathbf{r} = \frac{\mathbf{f} \times \mathbf{u}_{world}}{\|\mathbf{f} \times \mathbf{u}_{world}\|} $$
   If $\|\mathbf{f} \times \mathbf{u}_{world}\|$ is near zero (camera pointing straight up/down), handle degenerate case [G].

5. **Compute Up Vector:**
   $$ \mathbf{u} = \frac{\mathbf{r} \times \mathbf{f}}{\|\mathbf{r} \times \mathbf{f}\|} $$

6. **Construct View Matrix:**
   Using the look-at function:
   $$ \mathbf{M}_{view} = \text{lookAt}(\mathbf{p}, \mathbf{p} + \mathbf{f}, \mathbf{u}) $$

**Alternative: Scene Rotation Variant**
For rotating the scene around a center point $(x_{cen}, y_{cen}, z_{cen})$ using elevation ($\alpha$) and azimuth ($\beta$) [S1, unofficial]:
1. Translate scene center to origin: $\mathbf{v}' = \mathbf{v} - (x_{cen}, y_{cen}, z_{cen})$ [S1, unofficial].
2. Rotate about YZ axis by elevation $\alpha$ using axis $(\cos\beta, -\sin\beta, 1.0)$ [S1, unofficial].
3. Rotate about XY axis by azimuth $\beta$ using axis $(0.0, 0.0, 1.0)$ [S1, unofficial].
4. Translate back: $\mathbf{v}_{final} = \mathbf{v}' + (x_{cen}, y_{cen}, z_{cen})$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_camera_basis(yaw_deg, pitch_deg, world_up=(0, 1, 0)):
    # 1. Clamp pitch to prevent gimbal lock [S2, unofficial][S4, unofficial]
    pitch_deg = max(-89.0, min(89.0, pitch_deg))
    
    # 2. Convert to radians [G]
    yaw_rad = math.radians(yaw_deg)
    pitch_rad = math.radians(pitch_deg)
    
    # 3. Compute forward vector [S2, unofficial]
    cos_y = math.cos(yaw_rad)
    sin_y = math.sin(yaw_rad)
    cos_p = math.cos(pitch_rad)
    sin_p = math.sin(pitch_rad)
    
    fx = cos_y * cos_p
    fy = sin_p
    fz = sin_y * cos_p
    
    # Normalize forward [S2, unofficial][S3, unofficial]
    f_len = math.sqrt(fx**2 + fy**2 + fz**2)
    if f_len == 0:
        fx, fy, fz = 0, 1, 0 # [G] Default up if degenerate
    else:
        fx /= f_len
        fy /= f_len
        fz /= f_len
    forward = (fx, fy, fz)
    
    # 4. Compute right vector [S2, unofficial][S3, unofficial]
    # r = normalize(cross(forward, world_up))
    rx = fy * world_up[2] - fz * world_up[1]
    ry = fz * world_up[0] - fx * world_up[2]
    rz = fx * world_up[1] - fy * world_up[0]
    
    r_len = math.sqrt(rx**2 + ry**2 + rz**2)
    if r_len == 0:
        right = (1, 0, 0) # [G] Handle singularity when forward is parallel to world up
    else:
        right = (rx/r_len, ry/r_len, rz/r_len)
        
    # 5. Compute up vector [S2, unofficial][S3, unofficial]
    # u = normalize(cross(right, forward))
    ux = right[1] * fz - right[2] * fy
    uy = right[2] * fx - right[0] * fz
    uz = right[0] * fy - right[1] * fx
    
    u_len = math.sqrt(ux**2 + uy**2 + uz**2)
    if u_len == 0:
        up = (0, 1, 0)
    else:
        up = (ux/u_len, uy/u_len, uz/u_len)
        
    return forward, right, up

def look_at(position, target, up):
    # Standard look-at matrix construction [G]
    fx = target[0] - position[0]
    fy = target[1] - position[1]
    fz = target[2] - position[2]
    f_len = math.sqrt(fx**2 + fy**2 + fz**2)
    if f_len == 0:
        fx, fy, fz = 0, 0, 0
    else:
        fx /= f_len
        fy /= f_len
        fz /= f_len
        
    rx = up[1] * fz - up[2] * fy
    ry = up[2] * fx - up[0] * fz
    rz = up[0] * fy - up[1] * fx
    r_len = math.sqrt(rx**2 + ry**2 + rz**2)
    if r_len == 0:
        rx, ry, rz = 0, 0, 0
    else:
        rx /= r_len
        ry /= r_len
        rz /= r_len
        
    ux = fy * rz - fz * ry
    uy = fz * rx - fx * rz
    uz = fx * ry - fy * rx
    
    # Return basis vectors and translation for view matrix
    return (
        (rx, ry, rz, 0),
        (ux, uy, uz, 0),
        (-fx, -fy, -fz, 0),
        (rx*position[0] + ry*position[1] + rz*position[2],
         ux*position[0] + uy*position[1] + uz*position[2],
         -fx*position[0] - fy*position[1] - fz*position[2],
         1)
    )
```

**Edge Cases and Pitfalls:**
- **Gimbal Lock:** Pitch near $\pm 90^\circ$ causes the right vector to become undefined or flip [S2, unofficial][S4, unofficial]. Clamping pitch to $\pm 89^\circ$ avoids this [S2, unofficial].
- **Pitch Normalization Bug:** A known bug in some implementations sets `pitch = 360 - abs(yaw)` instead of `abs(pitch)`, causing pitch to mirror yaw [S3, unofficial].
- **Coordinate System Handedness:** The sign of sine/cosine components depends on whether the coordinate system is left-handed or right-handed and the definition of yaw (from X or Z axis) [S3, unofficial].

## Subtopics

### [Camera Orientation Setup with Gimbal Lock Prevention](6974-camera-orientation-setup-with-gimbal-lock-prevention.md)
Computes camera position, look-at target, and up-vector from spherical coordinates, explicitly handling the singularity where the camera points downwards or backwards to prevent gimbal lock and inverted views. It constructs a 3x3 rotation matrix representing the camera's orientation using up, right, and forward vectors. Choose this when robust orientation handling is required for complex camera paths. Link: [Camera Orientation Setup with Gimbal Lock Prevention](6974-camera-orientation-setup-with-gimbal-lock-prevention.md).

### Camera Control via Quaternions (Pitch/Yaw)
Updates camera orientation vectors (_forward, _right, _up) using quaternion rotation for yaw and pitch, avoiding gimbal lock [S5, unofficial]. It applies yaw rotation around the world Y-axis and pitch rotation around the local right axis using quaternion conjugation, then recomputes the up vector via cross product [S5, unofficial]. Choose this when smooth interpolation or avoiding singularities is critical.

### First-Person Camera Movement and Orientation Update
Updates camera position and orientation vectors based on keyboard input and mouse delta, recalculating view matrix and frustum [S6, unofficial]. Calculates sensitivity based on frame time (`0.5f / e.Time`), updates orientation vectors `_front`, `_right`, `_up` based on yaw and pitch, and moves along local axes by `SPEED * e.Time` (SPEED=10f) [S6, unofficial]. Handles mouse yaw/pitch updates, clamps pitch to $[-89^\circ, 89^\circ]$, and regenerates frustum on movement [S6, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ per frame for basis vector computation [G].
- **Memory Complexity:** $O(1)$ for storing angles and basis vectors [G].
- **Performance:** ~25 FPS at 256×224 resolution for a single object in a raymarcher using this method on a single thread with Lua/Luajit [S4, unofficial].
- **Strengths:** Simple to implement, intuitive for users (aircraft-style control), computationally cheap [S4, unofficial].
- **Limitations:** Susceptible to gimbal lock if not clamped, less flexible for complex camera paths compared to quaternions [S5, unofficial].
- **When to Use:** First-person or third-person camera controls in games and viewers where roll is not needed [S4, unofficial].
- **When Not to Use:** When smooth interpolation between arbitrary orientations is required, or when roll is necessary [G].

## Background
Originated in 3D visualization tools like Smokeview for fire dynamics simulation data, where scene rotation about vertical and horizontal axes through the scene center is used for navigation [S1, unofficial]. It is widely used in game engines (e.g., LÖVE with Fennel/Lua) for raymarching camera navigation [S4, unofficial] and in voxel rendering engines [S2, unofficial][S3, unofficial].

## Sources
[S1] Smokeview (Version 6) A Tool for Visualizing Fire Dynamics Simulation Data Volume II : Technical Reference Guide, https://doi.org/10.6028/nist.sp.1017-2, fetched 2026-10-26 (unofficial)
[S2] StanislavPetrovV/Minecraft, https://github.com/StanislavPetrovV/Minecraft, fetched 2026-10-26 (unofficial)
[S3] anb2473/Python-Voxel-Rendering-Engine, https://github.com/anb2473/Python-Voxel-Rendering-Engine, fetched 2026-10-27 (unofficial)
[S4] Raymarching with Fennel and LÖVE · Andrey Listopadov, https://andreyor.st/posts/2020-10-15-raymarching-with-fennel-and-love/, fetched 2026-10-30 (unofficial)
[S5] Onlinerocker/SVO, https://github.com/Onlinerocker/SVO, fetched 2026-10-24 (unofficial)
[S6] MProgramingUnityCsharp/VoxelCoreEngine, https://github.com/MProgramingUnityCsharp/VoxelCoreEngine, fetched 2026-10-27 (unofficial)
