# Arcball Camera View & Projection Matrix Computation
Also called: Arcball Camera Matrices, Camera View & Projection Matrix Computation, Perspective View Setup

## Overview
Computes the view and perspective projection matrices for an arcball-style camera, handling rotation around a pivot point and avoiding pitch flip artifacts [S1, unofficial]. It differs from standard look-at methods by explicitly managing the camera position via sequential yaw and pitch rotations around a fixed target [S1, unofficial].

## How it works
**Inputs:**
- Camera position $P$, target $T$, and up vector $U$ [S1, unofficial].
- Yaw angle $\theta_{yaw}$ and pitch angle $\theta_{pitch}$ [S1, unofficial].
- Pivot point $C$, typically the target $T$ [S1, unofficial].
- Screen aspect ratio $w/h$, vertical field of view $\phi$, near plane $z_{near}$, and far plane $z_{far}$ [S1, unofficial].

**Outputs:**
- View matrix $M_{view}$ [S1, unofficial].
- Projection matrix $M_{proj}$ [S1, unofficial].

**Data Structures:**
- Vectors are 3D Cartesian coordinates [G].
- Matrices are 4x4 column-major [G].
- Rotation matrices are constructed from axis-angle representation [G].

**Parameters:**
- $z_{near} = 0.01$ [S1, unofficial].
- $z_{far} = 100.0$ [S1, unofficial].
- $\phi = \pi / 2.0$ (90 degrees) [S1, unofficial].

**Steps:**
1. **Compute View Direction:** Calculate the normalized vector from the target to the current position: $V = \text{normalize}(P - T)$ [S1, unofficial].
2. **Gimbal Lock Check:** Compute the dot product $d = V \cdot U$. If $d \cdot \text{sign}(\theta_{pitch}) > 0.99$, set $\theta_{pitch} = 0.0$ to prevent pitch flip [S1, unofficial].
3. **Yaw Rotation:** Rotate the position $P$ around the pivot $C$ by $\theta_{yaw}$ using $U$ as the rotation axis. Let the result be $P_{yaw}$:
   $$ P_{yaw} = R(U, \theta_{yaw})(P - C) + C $$
4. **Right Vector:** Compute the right vector $R_{vec}$ from the rotated position:
   $$ R_{vec} = \text{normalize}(U \times P_{yaw}) $$
5. **Pitch Rotation:** Rotate $P_{yaw}$ around the pivot $C$ by $\theta_{pitch}$ using $R_{vec}$ as the rotation axis. Let the result be $P_{final}$:
   $$ P_{final} = R(R_{vec}, \theta_{pitch})(P_{yaw} - C) + C $$
6. **View Matrix:** Construct $M_{view}$ using a look-at function with eye position $P_{final}$, target $T$, and up vector $U$ [S1, unofficial].
7. **Projection Matrix:** Construct $M_{proj}$ using perspective projection with aspect ratio $w/h$, vertical FOV $\phi$, $z_{near}$, and $z_{far}$ [S1, unofficial].

**Edge Cases and Pitfalls:**
- When the camera direction aligns closely with the up vector, pitch rotation causes gimbal lock or flipping [S1, unofficial].
- Handled by clamping pitch to $0.0$ when $\cos(\text{angle}) \cdot \text{sign}(\text{pitch}) > 0.99$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def arcball_camera_matrices(position, target, up, yaw, pitch, pivot, width, height, znear=0.01, zfar=100.0, fovy=math.pi/2.0):
    """
    Computes view and projection matrices for an arcball camera.
    """
    # Step 1: Compute view direction
    view_dir = position - target
    view_dir_norm = view_dir / np.linalg.norm(view_dir)
    
    # Step 2: Gimbal lock check
    cos_angle = np.dot(view_dir_norm, up)
    pitch_sign = 1.0 if pitch >= 0 else -1.0
    if cos_angle * pitch_sign > 0.99:
        pitch = 0.0
        
    # Step 3: Yaw rotation around up vector
    position_rel = position - pivot
    rotation_x = rotation_matrix(up, yaw)
    position_rotated = np.dot(rotation_x, position_rel) + pivot
    
    # Step 4: Compute right vector
    right_vec = np.cross(up, position_rotated)
    right_vec_norm = right_vec / np.linalg.norm(right_vec)
    
    # Step 5: Pitch rotation around right vector
    position_rel_rotated = position_rotated - pivot
    rotation_y = rotation_matrix(right_vec_norm, pitch)
    final_position = np.dot(rotation_y, position_rel_rotated) + pivot
    
    # Step 6: View matrix (LookAt)
    view_matrix = look_at(final_position, target, up)
    
    # Step 7: Projection matrix (Perspective)
    aspect = width / height
    proj_matrix = perspective(fovy, aspect, znear, zfar)
    
    return view_matrix, proj_matrix

def rotation_matrix(axis, angle):
    """
    Computes a 4x4 rotation matrix for rotating around a given axis by a given angle.
    Axis is assumed to be normalized. Uses Rodrigues' rotation formula [G].
    """
    axis = axis / np.linalg.norm(axis)
    x, y, z = axis
    c = math.cos(angle)
    s = math.sin(angle)
    t = 1 - c
    
    mat = np.eye(4)
    mat[0, 0] = t * x * x + c
    mat[0, 1] = t * x * y - s * z
    mat[0, 2] = t * x * z + s * y
    
    mat[1, 0] = t * x * y + s * z
    mat[1, 1] = t * y * y + c
    mat[1, 2] = t * y * z - s * x
    
    mat[2, 0] = t * x * z - s * y
    mat[2, 1] = t * y * z + s * x
    mat[2, 2] = t * z * z + c
    
    return mat

def look_at(eye, target, up):
    """
    Computes a 4x4 view matrix (LookAt) following OpenGL conventions [G].
    """
    z_axis = eye - target
    z_axis = z_axis / np.linalg.norm(z_axis)
    
    x_axis = np.cross(up, z_axis)
    x_axis = x_axis / np.linalg.norm(x_axis)
    
    y_axis = np.cross(z_axis, x_axis)
    
    view = np.eye(4)
    view[0, 0] = x_axis[0]
    view[0, 1] = x_axis[1]
    view[0, 2] = x_axis[2]
    view[0, 3] = -np.dot(x_axis, eye)
    
    view[1, 0] = y_axis[0]
    view[1, 1] = y_axis[1]
    view[1, 2] = y_axis[2]
    view[1, 3] = -np.dot(y_axis, eye)
    
    view[2, 0] = z_axis[0]
    view[2, 1] = z_axis[1]
    view[2, 2] = z_axis[2]
    view[2, 3] = -np.dot(z_axis, eye)
    
    return view

def perspective(fovy, aspect, znear, zfar):
    """
    Computes a 4x4 perspective projection matrix following OpenGL conventions [G].
    """
    f = 1.0 / math.tan(fovy / 2.0)
    
    proj = np.zeros((4, 4))
    proj[0, 0] = f / aspect
    proj[1, 1] = f
    proj[2, 2] = (zfar + znear) / (znear - zfar)
    proj[2, 3] = -1.0
    proj[3, 2] = (2 * zfar * znear) / (znear - zfar)
    proj[3, 3] = 0.0
    
    return proj
```

## Subtopics

### [Camera Look-at Computation via Rotation](2696-camera-look-at-computation-via-rotation.md)
Computes view matrix from rotation matrices. Choose when you need explicit control over the view matrix construction from basis vectors.

### [Orbit Camera Controls](3262-orbit-camera-controls.md)
Handles user input for orbit cameras. Choose when you need to map mouse/keyboard events to yaw/pitch changes.

### [Spherical Camera Path Generation](3934-spherical-camera-path-generation.md)
Generates smooth camera paths on a sphere. Choose when you need automated camera movement rather than interactive control.

### [Volume Visualization Camera](4116-volume-visualization-camera.md)
Specialized camera setup for volume rendering. Choose when you need specific clipping or projection adjustments for volumetric data.

## Operating characteristics
- Time and memory complexity: Not given in the sources.
- Measured performance on specific hardware and resolution: Not given in the sources.
- Strengths: Handles rotation around a pivot point and avoids pitch flip artifacts [S1, unofficial].
- Limitations: The pitch flip check is a heuristic threshold (0.99) and may not cover all edge cases [S1, unofficial].
- When to use: Interactive 3D viewers and orbit cameras where the camera rotates around a central object [S1, unofficial].
- When not to use: First-person shooters or flight simulators requiring continuous pitch rotation without limits [S1, unofficial].

## Background
The method is implemented in the 3D Gaussian Splatting reference code by Thomas Antony [S1, unofficial].

## Sources
[S1] thomasantony/splat, https://github.com/thomasantony/splat, fetched 2026-10-27 (unofficial)
