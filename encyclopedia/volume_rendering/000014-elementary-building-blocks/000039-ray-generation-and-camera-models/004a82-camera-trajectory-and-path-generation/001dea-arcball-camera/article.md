# Arcball Camera

Also called: Arcball Camera Rotation, Arcball Rotation, Screen-to-Arcball, Screen-to-Arcball mapping, Trackball Camera, Trackball Camera Interaction, Virtual trackball rotation

## Overview

The Arcball Camera is a camera control technique that maps 2D screen coordinates to a 3D virtual sphere to compute rotation quaternions for smooth orbiting around a center point [S1, S2, S3, S4, S5, unofficial]. It differs from [First-Person Camera Control](2721-first-person-camera-control.md) by decoupling rotation from the camera's local forward vector, instead rotating the view around a fixed center [G]. It builds on [Camera Matrix and Coordinate Transformation](6231-camera-matrix-and-coordinate-transformation.md) to apply the resulting rotation to the view matrix [G].

## How it works

### Inputs and Outputs
**Inputs:**
- Mouse position $(x, y)$ in screen/pixel coordinates [S1, S5, unofficial].
- Viewport dimensions: width $W$, height $H$ [S1, S5, unofficial].
- Previous mouse position $(x_{prev}, y_{prev})$ and previous arcball state (rotation quaternion $q_{old}$) [S4, unofficial].
- Camera configuration: eye position $\mathbf{e}$, center point $\mathbf{c}$, up vector $\mathbf{u}$, zoom speed $k_{zoom}$, minimum/maximum camera distance $d_{min}, d_{max}$ [S1, S4, S5, unofficial].

**Outputs:**
- Updated camera view matrix $M_{cam}$ [S1, S5, unofficial].
- Updated rotation quaternion $q_{new}$ [S4, unofficial].
- Updated translation vector for zoom/pan [S1, unofficial].

### Data Structures

**ArcballState**
- `rotation`: Quaternion $q$ (w, x, y, z), normalized [S5, unofficial].
- `translation`: 4x4 matrix $T$ or vector $\mathbf{t}$, representing zoom/pan offset [S1, S5, unofficial].
- `centerTranslation`: 4x4 matrix $T_c$, representing the translation to the orbit center [S1, S5, unofficial].
- `invCamera`: 4x4 matrix, inverse of the current camera matrix [S1, S5, unofficial].
- `invScreen`: Array $[1/W, 1/H]$ used to normalize pixel coordinates [S1, S5, unofficial].
- `lastPosition`: 2D vector $(x_{prev}, y_{prev})$ [S3, S5, unofficial].
- `isRotating`: Boolean flag [S3, unofficial].

**CameraConfig**
- `eye`: 3D vector $\mathbf{e}$ [S5, unofficial].
- `center`: 3D vector $\mathbf{c}$ [S5, unofficial].
- `up`: 3D vector $\mathbf{u}$ (typically $[0, 1, 0]$) [S1, S5, unofficial].
- `zoomSpeed`: Scalar $k_{zoom}$ (e.g., 0.1) [S1, S5, unofficial].
- `minDist`: Float $d_{min}$ (e.g., 1.25) [S4, unofficial].
- `maxDist`: Float $d_{max}$ (e.g., 8.0) [S4, unofficial].

### Numbered Steps

1. **Initialize Camera Basis:**
   Compute the camera's local coordinate system [S1, unofficial].
   $$ \mathbf{z}_{axis} = \text{normalize}(\mathbf{c} - \mathbf{e}) $$
   $$ \mathbf{x}_{axis} = \text{normalize}(\text{cross}(\mathbf{z}_{axis}, \mathbf{u})) $$
   $$ \mathbf{y}_{axis} = \text{normalize}(\text{cross}(\mathbf{x}_{axis}, \mathbf{z}_{axis})) $$
   Construct the rotation matrix $R$ from these axes and convert to initial quaternion $q$ [S1, unofficial]. Store translation $\mathbf{t} = \mathbf{e} - \mathbf{c}$ [S1, unofficial].

2. **Normalize Mouse Coordinates:**
   Convert screen pixel coordinates to normalized device coordinates in the range $[-1, 1]$ [S1, S5, unofficial].
   $$ u = \text{clamp}\left( \frac{2x}{W} - 1, -1, 1 \right) $$
   $$ v = \text{clamp}\left( 1 - \frac{2y}{H}, -1, 1 \right) $$
   The y-axis inversion accounts for top-left origin screen coordinates [S2, S5, unofficial].

3. **Project to Arcball Surface (Screen-to-Arcball):**
   Map the 2D point $(u, v)$ to a 3D point $\mathbf{p}$ on the virtual trackball [S1, S5, unofficial].
   Calculate squared distance from center: $d^2 = u^2 + v^2$ [S1, S5, unofficial].
   - If $d^2 \le 1.0$: The point is inside the unit circle [S1, S5, unofficial]. Project onto the upper hemisphere of the sphere:
     $$ p_z = \sqrt{1.0 - d^2} $$
     $$ \mathbf{p} = (u, v, p_z) $$
   - If $d^2 > 1.0$: The point is outside the unit circle [S1, S5, unofficial]. Normalize the XY components and set $z = 0$ [S1, S5, unofficial].
     $$ \mathbf{p}_{xy} = \text{normalize}((u, v)) $$
     $$ p_z = 0 $$
     $$ \mathbf{p} = \mathbf{p}_{xy} $$
   Some implementations use a spherical cap projection where $z = \cos(\frac{\pi}{2} \min(d, 1))$ [S3, unofficial], or a hyperboloid fallback $z = 1/\sqrt{u^2+v^2}$ for constant angular velocity [S5, unofficial].

4. **Compute Rotation Delta:**
   Let $\mathbf{p}_{prev}$ be the arcball point from the previous frame and $\mathbf{p}_{curr}$ be the current point [S4, unofficial].
   Compute the rotation axis $\mathbf{a}$ and angle $\theta$ between these two vectors [S2, S4, unofficial].
   $$ \mathbf{a} = \text{cross}(\mathbf{p}_{prev}, \mathbf{p}_{curr}) $$
   $$ d = \text{dot}(\mathbf{p}_{prev}, \mathbf{p}_{curr}) $$
   **Edge Case Handling:**
   - If $d > 1 - \epsilon$: Vectors are parallel [S4, unofficial]. Return identity quaternion.
   - If $d < -1 + \epsilon$: Vectors are anti-parallel [S4, unofficial]. Find an orthogonal axis to $\mathbf{p}_{prev}$ by crossing with $[1,0,0]$ or $[0,1,0]$, normalize, and create a 180-degree quaternion [S4, unofficial].
   - Else: Normalize $\mathbf{a}$ and construct delta quaternion $q_{\Delta} = [1+d, \mathbf{a}]$, then normalize $q_{\Delta}$ [S4, unofficial].

5. **Update Rotation Quaternion:**
   $$ q_{new} = q_{\Delta} \times q_{old} $$
   Normalize $q_{new}$ [S4, unofficial].

6. **Update Camera Matrix:**
   Convert $q_{new}$ to rotation matrix $R_{quat}$ [S1, unofficial].
   The camera matrix $M_{cam}$ is constructed as:
   $$ M_{cam} = T_{trans} \cdot R_{quat} \cdot T_{center}^{-1} $$
   Where $T_{center}$ translates the center $\mathbf{c}$ to the origin, $R_{quat}$ applies the rotation, and $T_{trans}$ translates the eye back to its position relative to the center [S1, S5, unofficial].

7. **Handle Zoom and Pan:**
   - **Zoom:** Compute translation vector $\mathbf{v}_{zoom} = [0, 0, \text{amount} \cdot k_{zoom}]$ [S1, unofficial]. Update $T_{trans}$ and clamp the distance component to $[d_{min}, d_{max}]$ to prevent passing through the center [S1, S4, unofficial].
   - **Pan:** Compute delta in screen space, transform to world space using $M_{cam}^{-1}$, and update $T_{center}$ or the eye position relative to the center [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def normalize(v):
    l = math.sqrt(v[0]**2 + v[1]**2 + v[2]**2)
    if l < 1e-6:
        return [0.0, 0.0, 0.0]
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
    u = max(-1.0, min(1.0, 2.0 * x / width - 1.0))
    v = max(-1.0, min(1.0, 1.0 - 2.0 * y / height))
    dist_sq = u*u + v*v
    if dist_sq <= 1.0:
        z = math.sqrt(1.0 - dist_sq)
    else:
        length = math.sqrt(dist_sq)
        u /= length
        v /= length
        z = 0.0
    return [u, v, z]

def rotation_between_vectors(v1, v2):
    v1 = normalize(v1)
    v2 = normalize(v2)
    d = dot(v1, v2)
    if d > 1.0 - 1e-6:
        return [1.0, 0.0, 0.0, 0.0]
    if d < -1.0 + 1e-6:
        axis = cross(v1, [1.0, 0.0, 0.0])
        if math.sqrt(axis[0]**2 + axis[1]**2 + axis[2]**2) < 1e-6:
            axis = cross(v1, [0.0, 1.0, 0.0])
        axis = normalize(axis)
        return [0.0, axis[0], axis[1], axis[2]]
    axis = cross(v1, v2)
    quat = [1.0 + d, axis[0], axis[1], axis[2]]
    l = math.sqrt(quat[0]**2 + quat[1]**2 + quat[2]**2 + quat[3]**2)
    return [quat[0]/l, quat[1]/l, quat[2]/l, quat[3]/l]

def mat4_mul(A, B):
    C = [[0.0]*4 for _ in range(4)]
    for i in range(4):
        for j in range(4):
            for k in range(4):
                C[i][j] += A[i][k] * B[k][j]
    return C

class ArcballCamera:
    def __init__(self, eye, center, up, width, height):
        self.eye = eye
        self.center = center
        self.up = up
        self.width = width
        self.height = height
        z_axis = normalize([center[0]-eye[0], center[1]-eye[1], center[2]-eye[2]])
        x_axis = normalize(cross(z_axis, up))
        y_axis = normalize(cross(x_axis, z_axis))
        self.rotation = [1.0, 0.0, 0.0, 0.0]
        self.translation = [eye[0]-center[0], eye[1]-center[1], eye[2]-center[2]]
        self.last_mouse = [0, 0]
        self.last_arcball = screen_to_arcball(0, 0, width, height)

    def update_rotation(self, x, y):
        curr_arcball = screen_to_arcball(x, y, self.width, self.height)
        delta_quat = rotation_between_vectors(self.last_arcball, curr_arcball)
        self.rotation = quat_mul(delta_quat, self.rotation)
        self.last_mouse = [x, y]
        self.last_arcball = curr_arcball

    def update_zoom(self, amount, zoom_speed, min_dist, max_dist):
        zoom_factor = amount * zoom_speed
        view_dir = normalize([self.center[0]-self.eye[0], self.center[1]-self.eye[1], self.center[2]-self.eye[2]])
        new_eye = [
            self.eye[0] + view_dir[0] * zoom_factor,
            self.eye[1] + view_dir[1] * zoom_factor,
            self.eye[2] + view_dir[2] * zoom_factor
        ]
        dist = math.sqrt((new_eye[0]-self.center[0])**2 + (new_eye[1]-self.center[1])**2 + (new_eye[2]-self.center[2])**2)
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
        self.eye = new_eye
        self.translation = [new_eye[0]-self.center[0], new_eye[1]-self.center[1], new_eye[2]-self.center[2]]

    def get_camera_matrix(self):
        R = quat_to_mat4(self.rotation)
        T = [
            [1, 0, 0, self.translation[0]],
            [0, 1, 0, self.translation[1]],
            [0, 0, 1, self.translation[2]],
            [0, 0, 0, 1]
        ]
        Tc_inv = [
            [1, 0, 0, -self.center[0]],
            [0, 1, 0, -self.center[1]],
            [0, 0, 1, -self.center[2]],
            [0, 0, 0, 1]
        ]
        return mat4_mul(T, mat4_mul(R, Tc_inv))
```

### Parameters
- `width`, `height`: Viewport dimensions in pixels [S1, S5, unofficial].
- `zoomSpeed`: Scalar multiplier for zoom magnitude (e.g., 0.1) [S1, S5, unofficial].
- `minDist`, `maxDist`: Minimum and maximum camera distance from center (e.g., 1.25, 8.0) [S4, unofficial].
- `eye`, `center`, `up`: Initial camera position, target, and up vector [S1, S5, unofficial].

### Edge Cases and Pitfalls
- **Gimbal Lock:** Avoided by using quaternions instead of Euler angles [S1, S4, unofficial].
- **Parallel/Anti-parallel Vectors:** When the mouse moves such that the arcball vectors are parallel or anti-parallel, the cross product is zero [S4, unofficial]. Handle by checking the dot product against thresholds ($1-\epsilon$, $-1+\epsilon$) and using a fallback orthogonal axis for 180-degree rotations [S4, unofficial].
- **Boundary Clamping:** Mouse coordinates must be clamped to $[-1, 1]$ before projection to prevent artifacts at screen edges [S1, S5, unofficial].
- **Zoom Clamping:** The camera distance must be clamped to prevent the eye from passing through the center point [S1, S4, unofficial].
- **Projection Boundary:** For points outside the unit circle, the provided sources [S1, S5, unofficial] normalize the XY components and set $z=0$ in their code [S1, S5, unofficial]. Alternative projections use a spherical cap $z = \cos(\frac{\pi}{2} \min(d, 1))$ [S3, unofficial] or a hyperboloid $z = 1/\sqrt{u^2+v^2}$ [S5, unofficial].

## Operating characteristics

- **Time Complexity:** $O(1)$ per frame for rotation update [G].
- **Memory Complexity:** $O(1)$ for state storage (quaternion, translation, matrices) [G].
- **Performance:** Very fast, suitable for real-time interaction [G].
- **Strengths:** Intuitive control, no gimbal lock, smooth rotation [G].
- **Limitations:** Can be disorienting if the center point is not visible or if the rotation axis is not aligned with user expectations [G].
- **When to use:** Interactive volume exploration and medical imaging viewers [S2, S4, unofficial].
- **When not to use:** When precise, axis-aligned rotation is required, or when the camera needs to follow a specific path [G].

## Background

The Arcball Camera is implemented in various volume rendering and 3D visualization applications [S1, S2, S3, S4, S5, unofficial]. It maps 2D mouse movements to a 3D sphere, allowing users to rotate the view as if interacting with a physical trackball [S2, S4, unofficial]. The method is used in web-based marching cubes renderers [S1, unofficial], Qt-based medical volume viewers [S4, unofficial], and WebGL raycasting implementations [S3, S5, unofficial].

## Sources

[S1] Twinklebear/webgl-marching-cubes, https://github.com/Twinklebear/webgl-marching-cubes, fetched 2026-10-26 (unofficial)
[S2] fargiolas/qvrc, https://github.com/fargiolas/qvrc, fetched 2026-10-26 (unofficial)
[S3] faviasono/raycastingrendering, https://github.com/faviasono/raycastingrendering, fetched 2026-10-26 (unofficial)
[S4] wpop/qt-viewer-pro, https://github.com/wpop/qt-viewer-pro, fetched 2026-10-27 (unofficial)
[S5] sunjianxin/rendner_webgl, https://github.com/sunjianxin/rendner_webgl, fetched 2026-10-27 (unofficial)
