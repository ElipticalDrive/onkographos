# Arcball Camera

Also called: Arcball Camera Rotation, Arcball Rotation, Screen-to-Arcball, Screen-to-Arcball mapping, Trackball Camera, Trackball Camera Interaction, Virtual trackball rotation

## Overview

The Arcball Camera is a camera control technique that maps 2D screen coordinates (typically mouse movements) to a 3D virtual sphere to compute rotation quaternions, allowing intuitive orbiting around a center point without gimbal lock [S1, S2, S3, S4, S5, unofficial]. It differs from [First-Person Camera Control](2721-first-person-camera-control.md) by decoupling rotation from the camera's local forward vector, instead rotating the view around a fixed center, and from [Camera Trajectory and Path Generation](6410-camera-trajectory-and-path-generation.md) by being an interactive, direct-manipulation method rather than a scripted path. It builds on [Camera Matrix and Coordinate Transformation](6231-camera-matrix-and-coordinate-transformation.md) to apply the resulting rotation to the view matrix.

## How it works

### Inputs and Outputs
**Inputs:**
- Mouse position $(x, y)$ in screen/pixel coordinates.
- Viewport dimensions: width $W$, height $H$.
- Previous mouse position $(x_{prev}, y_{prev})$ and previous arcball state (rotation quaternion $q_{old}$).
- Camera configuration: eye position $\mathbf{e}$, center point $\mathbf{c}$, up vector $\mathbf{u}$, zoom speed $k_{zoom}$, minimum/maximum camera distance $d_{min}, d_{max}$.

**Outputs:**
- Updated camera view matrix $M_{cam}$.
- Updated rotation quaternion $q_{new}$.
- Updated translation vector for zoom/pan.

### Data Structures

**ArcballState**
- `rotation`: Quaternion $q$ (w, x, y, z), normalized.
- `translation`: 4x4 matrix $T$ or vector $\mathbf{t}$, representing zoom/pan offset.
- `centerTranslation`: 4x4 matrix $T_c$, representing the translation to the orbit center.
- `invCamera`: 4x4 matrix, inverse of the current camera matrix.
- `invScreen`: Array $[1/W, 1/H]$.
- `lastPosition`: 2D vector $(x_{prev}, y_{prev})$.
- `isRotating`: Boolean flag.

**CameraConfig**
- `eye`: 3D vector $\mathbf{e}$.
- `center`: 3D vector $\mathbf{c}$.
- `up`: 3D vector $\mathbf{u}$ (typically $[0, 1, 0]$).
- `zoomSpeed`: Scalar $k_{zoom}$ (e.g., 0.1).
- `minDist`: Float $d_{min}$ (e.g., 1.25).
- `maxDist`: Float $d_{max}$ (e.g., 8.0).

### Numbered Steps

1. **Initialize Camera Basis:**
   Compute the camera's local coordinate system.
   $$ \mathbf{z}_{axis} = \text{normalize}(\mathbf{c} - \mathbf{e}) $$
   $$ \mathbf{x}_{axis} = \text{normalize}(\text{cross}(\mathbf{z}_{axis}, \mathbf{u})) $$
   $$ \mathbf{y}_{axis} = \text{normalize}(\text{cross}(\mathbf{x}_{axis}, \mathbf{z}_{axis})) $$
   Construct the rotation matrix $R$ from these axes. Convert $R$ to initial quaternion $q$. Store translation $\mathbf{t} = \mathbf{e} - \mathbf{c}$ (or similar relative offset).

2. **Normalize Mouse Coordinates:**
   Convert screen pixel coordinates to normalized device coordinates (NDC) in the range $[-1, 1]$.
   $$ u = \text{clamp}\left( \frac{2x}{W} - 1, -1, 1 \right) $$
   $$ v = \text{clamp}\left( 1 - \frac{2y}{H}, -1, 1 \right) $$
   Note: The y-axis inversion depends on the screen coordinate system (top-left origin vs. bottom-left). [S2, S5, unofficial] use $1 - 2y/H$. [S3, unofficial] uses $(H - 2y)/radius$.

3. **Project to Arcball Surface (Screen-to-Arcball):**
   Map the 2D point $(u, v)$ to a 3D point $\mathbf{p}$ on the virtual trackball.
   Calculate squared distance from center: $d^2 = u^2 + v^2$.
   - If $d^2 \le 1.0$: The point is inside the unit circle. Project onto the upper hemisphere of the sphere.
     $$ p_z = \sqrt{1.0 - d^2} $$
     $$ \mathbf{p} = (u, v, p_z) $$
   - If $d^2 > 1.0$: The point is outside the unit circle. Project onto the hyperboloid surface to maintain constant angular velocity and avoid gimbal lock [S5, G, unofficial].
     $$ \mathbf{p}_{xy} = \text{normalize}((u, v)) $$
     $$ p_z = 0 $$
     $$ \mathbf{p} = \mathbf{p}_{xy} $$
     *Note: Some implementations use $p_z = 1/\sqrt{u^2+v^2}$ for the hyperboloid [G], but [S5, unofficial] and [S1, unofficial] imply $p_z=0$ for the "else" branch in the provided code snippets, effectively capping the projection to the equator. The standard arcball uses the hyperboloid for smoother rotation outside the sphere. We follow the explicit code in [S1, S5, unofficial] which sets $z=0$ for $d>1$, but acknowledge the hyperboloid alternative [G].*

4. **Compute Rotation Delta:**
   Let $\mathbf{p}_{prev}$ be the arcball point from the previous frame and $\mathbf{p}_{curr}$ be the current point.
   Compute the rotation axis $\mathbf{a}$ and angle $\theta$ between these two vectors.
   $$ \mathbf{a} = \text{cross}(\mathbf{p}_{prev}, \mathbf{p}_{curr}) $$
   $$ \cos \theta = \text{dot}(\mathbf{p}_{prev}, \mathbf{p}_{curr}) $$
   $$ \sin \theta = \|\mathbf{a}\| $$
   
   **Edge Case Handling:**
   - If $\|\mathbf{a}\| < \epsilon$ (vectors are parallel or anti-parallel):
     - If $\cos \theta > 1 - \epsilon$: No rotation ($\theta \approx 0$). Return identity quaternion.
     - If $\cos \theta < -1 + \epsilon$: 180-degree rotation. Find an orthogonal axis to $\mathbf{p}_{prev}$ by crossing with a non-parallel vector (e.g., $[1,0,0]$ or $[0,1,0]$). Normalize and create quaternion from axis-angle.
   - Else:
     $$ \mathbf{a} = \text{normalize}(\mathbf{a}) $$
     $$ \theta = \text{atan2}(\sin \theta, \cos \theta) $$
     Construct delta quaternion $q_{\Delta}$ from axis $\mathbf{a}$ and angle $\theta$.

5. **Update Rotation Quaternion:**
   $$ q_{new} = q_{\Delta} \times q_{old} $$
   Normalize $q_{new}$.

6. **Update Camera Matrix:**
   Convert $q_{new}$ to rotation matrix $R_{new}$.
   The camera matrix $M_{cam}$ is constructed as:
   $$ M_{cam} = T_{trans} \cdot R_{quat} \cdot T_{center}^{-1} $$
   Where:
   - $T_{center}$ translates the center $\mathbf{c}$ to the origin.
   - $R_{quat}$ applies the rotation.
   - $T_{trans}$ translates the eye back to its position relative to the center (zoom/pan).
   *Note: [S1, unofficial] uses `camera = mat4.mul(camera, rotMat, centerTranslation)` and `camera = mat4.mul(camera, translation, camera)`. This implies a specific multiplication order. Standard OpenGL column-major: $M = T \cdot R \cdot T_c^{-1}$. [S5, unofficial] confirms $M_{cam} = T_{trans} \cdot R_{quat} \cdot T_{center}^{-1}$.*

7. **Handle Zoom and Pan:**
   - **Zoom:** Based on mouse wheel or right-click drag. Compute translation vector $\mathbf{v}_{zoom} = [0, 0, \text{amount} \cdot k_{zoom}]$. Update $T_{trans}$. Clamp the distance component to $[d_{min}, d_{max}]$ to prevent passing through the center [S1, S4, unofficial].
   - **Pan:** Based on middle-click drag. Compute delta in screen space, transform to world space using $M_{cam}^{-1}$, and update $T_{center}$ or the eye position relative to the center.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def normalize(v):
    l = math.sqrt(v[0]**2 + v[1]**2 + v[2]**2)
    if l < 1e-6:
        return [0, 0, 0]
    return [v[0]/l, v[1]/l, v[2]/l]

def dot(v1, v2):
    return v1[0]*v2[0] + v1[1]*v2[1] + v1[2]*v2[2]

def cross(v1, v2):
    return [
        v1[1]*v2[2] - v1[2]*v2[1],
        v1[2]*v2[0] - v1[0]*v2[2],
        v1[0]*v2[1] - v1[1]*v2[0]
    ]

def quat_mul(q1, q2):
    # q = [w, x, y, z]
    w1, x1, y1, z1 = q1
    w2, x2, y2, z2 = q2
    return [
        w1*w2 - x1*x2 - y1*y2 - z1*z2,
        w1*x2 + x1*w2 + y1*z2 - z1*y2,
        w1*y2 - x1*z2 + y1*w2 + z1*x2,
        w1*z2 + x1*y2 - y1*x2 + z1*w2
    ]

def quat_to_mat4(q):
    w, x, y, z = q
    # Standard quaternion to 4x4 rotation matrix
    xx, yy, zz = x*x, y*y, z*z
    xy, xz, yz = x*y, x*z, y*z
    wx, wy, wz = w*x, w*y, w*z
    return [
        [1 - 2*(yy + zz), 2*(xy - wz), 2*(xz + wy), 0],
        [2*(xy + wz), 1 - 2*(xx + zz), 2*(yz - wx), 0],
        [2*(xz - wy), 2*(yz + wx), 1 - 2*(xx + yy), 0],
        [0, 0, 0, 1]
    ]

def screen_to_arcball(x, y, width, height):
    # Normalize to [-1, 1]
    u = max(-1.0, min(1.0, 2.0 * x / width - 1.0))
    v = max(-1.0, min(1.0, 1.0 - 2.0 * y / height))
    
    dist_sq = u*u + v*v
    if dist_sq <= 1.0:
        z = math.sqrt(1.0 - dist_sq)
    else:
        # Project to hyperboloid or equator. 
        # Sources [S1, S5, unofficial] use equator (z=0) for d>1 in code, 
        # but standard arcball uses hyperboloid for smoothness.
        # We follow the explicit "else" logic in [S5, unofficial] which normalizes xy and sets z=0.
        length = math.sqrt(dist_sq)
        u /= length
        v /= length
        z = 0.0
    return [u, v, z]

def rotation_between_vectors(v1, v2):
    # Returns quaternion representing rotation from v1 to v2
    v1 = normalize(v1)
    v2 = normalize(v2)
    
    d = dot(v1, v2)
    
    # Parallel
    if d > 1.0 - 1e-6:
        return [1.0, 0.0, 0.0, 0.0]
    
    # Anti-parallel
    if d < -1.0 + 1e-6:
        # Find orthogonal axis
        axis = cross(v1, [1.0, 0.0, 0.0])
        if math.sqrt(axis[0]**2 + axis[1]**2 + axis[2]**2) < 1e-6:
            axis = cross(v1, [0.0, 1.0, 0.0])
        axis = normalize(axis)
        # 180 degree rotation
        return [0.0, axis[0], axis[1], axis[2]]
    
    # General case
    axis = cross(v1, v2)
    w = 1.0 + d
    quat = [w, axis[0], axis[1], axis[2]]
    # Normalize
    l = math.sqrt(quat[0]**2 + quat[1]**2 + quat[2]**2 + quat[3]**2)
    return [quat[0]/l, quat[1]/l, quat[2]/l, quat[3]/l]

class ArcballCamera:
    def __init__(self, eye, center, up, width, height):
        self.eye = eye
        self.center = center
        self.up = up
        self.width = width
        self.height = height
        
        # Compute initial basis
        z_axis = normalize([center[0]-eye[0], center[1]-eye[1], center[2]-eye[2]])
        x_axis = normalize(cross(z_axis, up))
        y_axis = normalize(cross(x_axis, z_axis))
        
        # Initial rotation is identity relative to basis, or derived from basis
        self.rotation = [1.0, 0.0, 0.0, 0.0]
        
        # Translation: vector from center to eye
        self.translation = [eye[0]-center[0], eye[1]-center[1], eye[2]-center[2]]
        
        # Center translation matrix (inverse)
        self.center_translation = [
            [1, 0, 0, -center[0]],
            [0, 1, 0, -center[1]],
            [0, 0, 1, -center[2]],
            [0, 0, 0, 1]
        ]
        
        self.last_mouse = [0, 0]
        self.last_arcball = screen_to_arcball(0, 0, width, height)
        
    def update_rotation(self, x, y):
        curr_arcball = screen_to_arcball(x, y, self.width, self.height)
        
        # Compute delta rotation
        delta_quat = rotation_between_vectors(self.last_arcball, curr_arcball)
        
        # Update rotation: q_new = delta * q_old
        self.rotation = quat_mul(delta_quat, self.rotation)
        
        self.last_mouse = [x, y]
        self.last_arcball = curr_arcball
        
    def update_zoom(self, amount):
        # amount is typically mouse wheel delta
        zoom_factor = amount * 0.1 # zoomSpeed
        # Translate along the local Z axis (view direction)
        # In camera space, Z is forward. 
        # We need to rotate the translation vector by the current rotation
        # to get the world-space zoom direction.
        # Simplified: adjust distance from center.
        
        # Current eye position relative to center
        # Apply rotation to the translation vector
        # Note: This is a simplification. Full implementation requires matrix mult.
        # For brevity, we assume translation is stored in camera space and rotated.
        
        # Let's use the matrix approach for correctness
        rot_mat = quat_to_mat4(self.rotation)
        
        # Current eye position
        eye = [
            self.translation[0] + self.center[0],
            self.translation[1] + self.center[1],
            self.translation[2] + self.center[2]
        ]
        
        # View direction
        view_dir = normalize([self.center[0]-eye[0], self.center[1]-eye[1], self.center[2]-eye[2]])
        
        # Move eye along view direction
        new_eye = [
            eye[0] + view_dir[0] * zoom_factor,
            eye[1] + view_dir[1] * zoom_factor,
            eye[2] + view_dir[2] * zoom_factor
        ]
        
        # Clamp distance
        dist = math.sqrt((new_eye[0]-self.center[0])**2 + (new_eye[1]-self.center[1])**2 + (new_eye[2]-self.center[2])**2)
        min_dist = 1.25
        max_dist = 8.0
        
        if dist < min_dist:
            scale = min_dist / dist
            new_eye = [self.center[0] + (new_eye[0]-self.center[0])*scale,
                       self.center[1] + (new_eye[1]-self.center[1])*scale,
                       self.center[2] + (new_eye[2]-self.center[2])*scale]
        elif dist > max_dist:
            scale = max_dist / dist
            new_eye = [self.center[0] + (new_eye[0]-self.center[0])*scale,
                       self.center[1] + (new_eye[1]-self.center[1])*scale,
                       self.center[2] + (new_eye[2]-self.center[2])*scale]
                       
        self.translation = [new_eye[0]-self.center[0], new_eye[1]-self.center[1], new_eye[2]-self.center[2]]

    def get_camera_matrix(self):
        # M_cam = T_trans * R * T_center_inv
        rot_mat = quat_to_mat4(self.rotation)
        
        # Translation matrix for eye offset
        t_mat = [
            [1, 0, 0, self.translation[0]],
            [0, 1, 0, self.translation[1]],
            [0, 0, 1, self.translation[2]],
            [0, 0, 0, 1]
        ]
        
        # Multiply T * R
        tr_mat = [
            [rot_mat[0][0], rot_mat[0][1], rot_mat[0][2], t_mat[0][3]],
            [rot_mat[1][0], rot_mat[1][1], rot_mat[1][2], t_mat[1][3]],
            [rot_mat[2][0], rot_mat[2][1], rot_mat[2][2], t_mat[2][3]],
            [0, 0, 0, 1]
        ]
        
        # Multiply (T*R) * T_center_inv
        # T_center_inv translates by -center
        final_mat = [
            [tr_mat[0][0], tr_mat[0][1], tr_mat[0][2], tr_mat[0][3] - tr_mat[0][0]*self.center[0] - tr_mat[0][1]*self.center[1] - tr_mat[0][2]*self.center[2]],
            [tr_mat[1][0], tr_mat[1][1], tr_mat[1][2], tr_mat[1][3] - tr_mat[1][0]*self.center[0] - tr_mat[1][1]*self.center[1] - tr_mat[1][2]*self.center[2]],
            [tr_mat[2][0], tr_mat[2][1], tr_mat[2][2], tr_mat[2][3] - tr_mat[2][0]*self.center[0] - tr_mat[2][1]*self.center[1] - tr_mat[2][2]*self.center[2]],
            [0, 0, 0, 1]
        ]
        
        return final_mat
```

### Parameters
- `width`, `height`: Viewport dimensions in pixels.
- `zoomSpeed`: Scalar multiplier for zoom magnitude (e.g., 0.1).
- `minDist`, `maxDist`: Minimum and maximum camera distance from center (e.g., 1.25, 8.0) [S4, unofficial].
- `eye`, `center`, `up`: Initial camera position, target, and up vector.

### Edge Cases and Pitfalls
- **Gimbal Lock:** Avoided by using quaternions instead of Euler angles [S1, S4, unofficial].
- **Parallel/Anti-parallel Vectors:** When the mouse moves such that the arcball vectors are parallel or anti-parallel, the cross product is zero. Handle by checking the dot product against thresholds ($1-\epsilon$, $-1+\epsilon$) and using a fallback orthogonal axis for 180-degree rotations [S4, unofficial].
- **Boundary Clamping:** Mouse coordinates must be clamped to $[-1, 1]$ before projection to prevent artifacts at screen edges [S1, S5, unofficial].
- **Zoom Clamping:** The camera distance must be clamped to prevent the eye from passing through the center point or moving too far away [S1, S4, unofficial].
- **Hyperboloid vs. Equator:** For points outside the unit circle, some implementations use a hyperboloid projection for smoother rotation, while others cap to the equator ($z=0$). The provided sources [S1, S5, unofficial] use the equator cap in code, but the hyperboloid is a common alternative [G].

## Operating characteristics

- **Time Complexity:** $O(1)$ per frame for rotation update. Matrix multiplications are constant time (4x4).
- **Memory Complexity:** $O(1)$ for state storage (quaternion, translation, matrices).
- **Performance:** Very fast, suitable for real-time interaction. The main cost is the matrix multiplication for the view matrix update, which is negligible on modern hardware.
- **Strengths:** Intuitive control, no gimbal lock, smooth rotation.
- **Limitations:** Can be disorienting if the center point is not visible or if the rotation axis is not aligned with user expectations. Not suitable for precise positioning compared to first-person controls.
- **When to use:** Interactive volume exploration, medical imaging viewers, 3D model inspection.
- **When not to use:** When precise, axis-aligned rotation is required, or when the camera needs to follow a specific path.

## Background

The Arcball Camera was introduced by Ken Shoemake in "Quaternion Calculus and Fast Animation" (1992) [S1, S3, unofficial]. It was designed to provide a more intuitive way to rotate objects in 3D space compared to Euler angles, avoiding gimbal lock. It has been widely adopted in 3D modeling software, CAD applications, and volume rendering viewers [S2, S4, unofficial]. The method maps 2D mouse movements to a 3D sphere, allowing users to "grab" the object and rotate it as if it were on a physical trackball.

## Sources

[S1] Twinklebear/webgl-marching-cubes, https://github.com/Twinklebear/webgl-marching-cubes, fetched 2026-10-26 (unofficial)
[S2] fargiolas/qvrc, https://github.com/fargiolas/qvrc, fetched 2026-10-26 (unofficial)
[S3] faviasono/raycastingrendering, https://github.com/faviasono/raycastingrendering, fetched 2026-10-26 (unofficial)
[S4] wpop/qt-viewer-pro, https://github.com/wpop/qt-viewer-pro, fetched 2026-10-27 (unofficial)
[S5] sunjianxin/rendner_webgl, https://github.com/sunjianxin/rendner_webgl, fetched 2026-10-27 (unofficial)
