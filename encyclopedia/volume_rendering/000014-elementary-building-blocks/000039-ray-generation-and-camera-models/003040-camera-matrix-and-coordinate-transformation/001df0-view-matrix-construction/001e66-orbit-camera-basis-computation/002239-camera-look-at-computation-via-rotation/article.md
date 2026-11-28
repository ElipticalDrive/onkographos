# Camera Look-at Computation via Rotation

Also called: Camera Transform Calculation, Euler Angle Camera Look-at, Spherical Coordinate Camera Orientation

## Overview
This method computes a camera's view matrix by deriving an orthonormal basis from either spherical rotation angles or a position and look direction [S1, unofficial][S2, unofficial]. It solves the problem of defining a camera's orientation in 3D space for ray generation, specifically for orbit cameras that rotate around a target [S2, unofficial]. It differs from [Arcball Camera View & Projection Matrix Computation](2210-arcball-camera-view-projection-matrix-computation.md) by using explicit Euler angles or direct vector construction rather than quaternion interpolation [S2, unofficial], and from [Orbit Camera Basis Computation](1749-orbit-camera-basis-computation.md) by providing the concrete implementation details for the basis derivation and matrix assembly [S1, unofficial].

## How it works
The method provides two primary approaches: computing a look-at point from spherical coordinates (Euler angles) or constructing the full view matrix from a position and look direction [S1, unofficial][S2, unofficial].

### Inputs and Outputs
*   **Inputs:**
    *   Camera position $P = [ax, ay, az]$ [S2, unofficial].
    *   Rotation angles: $x\text{-}angle$ and $z\text{-}angle$ in degrees [S2, unofficial].
    *   Camera distance $d$ from the target (default 2) [S1, unofficial].
    *   Optional: Look direction vector and reference up vector $\vec{u}_{ref} = [0, 0, -1]$ [S1, unofficial].
*   **Outputs:**
    *   Look-at point $L_{at}$ (intermediate) [S2, unofficial].
    *   4x4 Camera Transformation Matrix $M_{cam}$ [S1, unofficial].

### Data Structures
*   **Rotation Matrix:** 4x4 matrix, typically initialized as identity [S1, unofficial].
*   **Vectors:** 3D floating-point vectors [S1, unofficial][S2, unofficial].
*   **Constants:**
    *   $\text{RAD} = \pi / 180$ (conversion factor from degrees to radians) [S2, unofficial].

### Steps

#### Method A: Look-at Point from Spherical Coordinates
This method calculates where the camera is looking based on its position and rotation angles [S2, unofficial].

1.  **Convert Angles:** Convert $x\text{-}angle$ and $z\text{-}angle$ from degrees to radians:
    $$ x_{\text{rad}} = x\text{-}angle \cdot \text{RAD} $$
    $$ z_{\text{rad}} = z\text{-}angle \cdot \text{RAD} $$
2.  **Compute Trigonometry:** Calculate $\cos(x_{\text{rad}})$, $\sin(x_{\text{rad}})$, $\cos(z_{\text{rad}})$, and $\sin(z_{\text{rad}})$ [S2, unofficial].
3.  **Rotate Reference Vector:** Apply the rotations to a base direction vector $(x,y,z)$ [S2, unofficial]:
    $$ x' = \cos(x_{\text{rad}})\cos(z_{\text{rad}})x - \sin(x_{\text{rad}})y + \cos(x_{\text{rad}})\sin(z_{\text{rad}})z $$
    $$ y' = \cos(x_{\text{rad}})\sin(z_{\text{rad}})x + \sin(x_{\text{rad}})y + \cos(x_{\text{rad}})\cos(z_{\text{rad}})z $$
    The sources do not provide the $z'$ formula [G].
4.  **Calculate Look-at Point:** Add the rotated vector to the camera position $P$:
    $$ L_{at} = P + [x', y', z'] $$

#### Method B: Full View Matrix Construction
This method constructs the complete 4x4 view matrix from a position and a look direction [S1, unofficial].

1.  **Compute Forward Vector:**
    $$ \vec{f} = -\frac{\text{look\_direction}}{\|\text{look\_direction}\|} $$
2.  **Compute Right Vector:**
    $$ \vec{r} = \vec{f} \times [0, 0, -1] $$
3.  **Normalize Right Vector:**
    $$ \vec{r} = \frac{\vec{r}}{\|\vec{r}\|} $$
4.  **Compute Up Vector:**
    $$ \vec{u} = \vec{f} \times \vec{r} $$
5.  **Normalize Up Vector:**
    $$ \vec{u} = \frac{\vec{u}}{\|\vec{u}\|} $$
6.  **Construct Rotation Matrix:** Create a 3x3 rotation matrix $R$ using $\vec{r}, \vec{u}, \vec{f}$ as columns 0, 1, and 2 respectively [S1, unofficial].
    $$ R = \begin{bmatrix} r_x & u_x & f_x \\ r_y & u_y & f_y \\ r_z & u_z & f_z \end{bmatrix} $$
7.  **Construct Translation Matrix:** Create a 4x4 translation matrix $T$ with the camera position in the fourth column [S1, unofficial].
8.  **Combine:** Multiply translation by rotation:
    $$ M_{cam} = T \cdot R $$

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def rotation_matrix(axis, angle):
    """Returns a 4x4 rotation matrix for the given axis and angle."""
    c = math.cos(angle)
    s = math.sin(angle)
    if axis == 'x':
        return np.array([
            [1, 0, 0, 0],
            [0, c, -s, 0],
            [0, s, c, 0],
            [0, 0, 0, 1]
        ])
    elif axis == 'y':
        return np.array([
            [c, 0, s, 0],
            [0, 1, 0, 0],
            [-s, 0, c, 0],
            [0, 0, 0, 1]
        ])
    elif axis == 'z':
        return np.array([
            [c, -s, 0, 0],
            [s, c, 0, 0],
            [0, 0, 1, 0],
            [0, 0, 0, 1]
        ])
    raise ValueError("Unsupported axis")

def get_camera_transform_looking_at_origin(camera_pos, x_angle_deg, z_angle_deg, camera_distance=2):
    """Computes camera transform by rotating a forward vector based on Euler angles [S1, unofficial]."""
    camera_transform = np.eye(4)
    camera_transform[2, 3] = camera_distance
    x_rad = x_angle_deg * (math.pi / 180)
    z_rad = z_angle_deg * (math.pi / 180)
    camera_transform = rotation_matrix('x', x_rad) @ camera_transform
    camera_transform = rotation_matrix('y', z_rad) @ camera_transform
    return camera_transform

def get_camera_transform(position, look_direction, up_vector=np.array([0, 0, -1])):
    """Computes camera transform from position and look direction [S1, unofficial]."""
    forward = -look_direction / np.linalg.norm(look_direction)
    right = np.cross(forward, up_vector)
    norm_right = np.linalg.norm(right)
    if norm_right < 0.5:
        right = np.array([0, 1, 0])
    else:
        right = right / norm_right
    up = np.cross(forward, right)
    norm_up = np.linalg.norm(up)
    if norm_up < 0.5:
        up = np.array([0, 1, 0])
    else:
        up = up / norm_up
    rotation = np.eye(4)
    rotation[0, :3] = right
    rotation[1, :3] = up
    rotation[2, :3] = forward
    translation = np.eye(4)
    translation[0, 3] = position[0]
    translation[1, 3] = position[1]
    translation[2, 3] = position[2]
    return translation @ rotation

def compute_look_at_point(camera_pos, x_angle_deg, z_angle_deg):
    """Computes look-at point using spherical coordinate rotation [S2, unofficial]."""
    x_rad = x_angle_deg * (math.pi / 180)
    z_rad = z_angle_deg * (math.pi / 180)
    cos_x = math.cos(x_rad)
    sin_x = math.sin(x_rad)
    cos_z = math.cos(z_rad)
    sin_z = math.sin(z_rad)
    dx, dy, dz = 1.0, 0.0, 0.0
    new_x = cos_x * cos_z * dx - sin_x * dy + cos_x * sin_z * dz
    new_y = cos_x * sin_z * dx + sin_x * dy + cos_x * cos_z * dz
    # z' formula is not provided in the sources [G]
    new_z = 0.0
    return np.array(camera_pos) + np.array([new_x, new_y, new_z])
```

### Parameters
*   `camera_distance`: Distance of the camera from the origin. Default: 2 [S1, unofficial].
*   `x-rotate`, `z-rotate`: Euler angles in degrees. Default: 255 and 15 respectively [S2, unofficial].
*   `RAD`: $\pi/180$ [S2, unofficial].

### Edge Cases and Pitfalls
*   **Degenerate Basis:** If the norm of the computed `camera_right` vector is less than 0.5, the basis is degenerate (camera is looking parallel to the up vector). Replace `camera_right` with $[0, 1, 0]$ [S1, unofficial].
*   **Coordinate System:** The specific order of rotation (X then Y in [S1, unofficial] vs X then Z in [S2, unofficial]) and the definition of the "forward" vector axis depend on the local coordinate system (Left-Handed vs Right-Handed) [S1, unofficial][S2, unofficial]. Implementers must verify the axis convention of their renderer [G].

## Subtopics

### [Camera Control via Euler Angles (Pitch/Yaw)](3431-camera-control-via-euler-angles-pitch-yaw.md)
Computes orientation from yaw and pitch, excluding roll. Chosen for intuitive navigation and lower computational cost. Builds on this method by deriving basis vectors from spherical angles.

### [Orbit Camera Basis Computation](1749-orbit-camera-basis-computation.md)
Computes the orthonormal basis for an orbit camera. This method provides the specific look-at calculation used within that basis.

## Operating characteristics
*   **Time Complexity:** $O(1)$ for fixed-size matrix operations [G].
*   **Memory Complexity:** $O(1)$, storing only a few vectors and a 4x4 matrix [G].
*   **Strengths:** Simple to implement; provides intuitive control via angles; robust against gimbal lock if only two axes are used (pitch/yaw) [S2, unofficial][G].
*   **Limitations:** Euler angles can suffer from gimbal lock if all three axes are used; order of rotation matters significantly; requires careful handling of degenerate cases (up vector alignment) [S1, unofficial].
*   **When to use:** Orbit cameras, simple orbiters, or when explicit control over pitch and yaw is required [S2, unofficial].
*   **When not to use:** When smooth interpolation between arbitrary orientations is needed (use Quaternions/Arcball); when roll is required [G].

## Background
The method originates from practical implementations in raymarching and 3D visualization tools [S1, unofficial][S2, unofficial]. [S1, unofficial] presents it in the context of a mesh-to-SDF tool, providing generic functions for camera transforms. [S2, unofficial] details its application in a LÖVE-based raymarcher, using spherical coordinates to allow user navigation. It is a standard technique in computer graphics for converting angular input into view matrices [G].

## Sources
[S1] mesh_to_sdf, https://github.com/marian42/mesh_to_sdf, fetched 2026-10-28 (unofficial)
[S2] Raymarching with Fennel and LÖVE · Andrey Listopadov, https://andreyor.st/posts/2020-10-15-raymarching-with-fennel-and-love/, fetched 2026-10-30 (unofficial)
