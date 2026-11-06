# Arcball Camera

Also called: Arcball Rotation, Screen-to-Arcball, Trackball Camera, Trackball Camera Interaction

## Overview
The Arcball Camera is an interactive camera control technique that maps 2D screen coordinates to a 3D virtual sphere to compute rotation quaternions for smooth orbiting around a center point [S1, S2, S3, S4, unofficial]. It decouples 2D input from 3D orientation, allowing intuitive rotation without gimbal lock [S2, S4, unofficial]. It differs from [First-Person Camera Control](2721-first-person-camera-control.md) by orbiting a fixed center rather than moving the eye position directly [S4, unofficial].

## How it works
The method maintains a camera state defined by an eye position $\mathbf{e}$, a target (center) point $\mathbf{c}$, and an up vector $\mathbf{up}$ [S1, unofficial]. It tracks previous and current mouse positions to compute incremental rotations [S2, S3, S4, unofficial].

### Inputs and Outputs
*   **Inputs**:
    *   Current mouse coordinates $(x, y)$ and previous mouse coordinates $(x_{\text{prev}}, y_{\text{prev}})$ in pixel space [S2, S3, S4, unofficial].
    *   Viewport dimensions: width $W$, height $H$ [S2, S3, unofficial].
    *   Camera parameters: eye $\mathbf{e}$, center $\mathbf{c}$, up vector $\mathbf{up}$, zoom speed $k_z$, pan sensitivity [S1, S4, unofficial].
    *   Current rotation quaternion $q_{\text{curr}}$ and translation vector $\mathbf{t}_{\text{trans}}$ [S1, S4, unofficial].
*   **Outputs**:
    *   Updated camera matrix $M_{\text{cam}}$ (or eye/target/up vectors) [S1, unofficial].
    *   Updated rotation quaternion $q_{\text{new}}$ and translation $\mathbf{t}_{\text{new}}$ [S1, S4, unofficial].

### Data Structures
The state typically includes:
*   `width`, `height`: Integers or floats representing viewport dimensions [S3, unofficial].
*   `last_position`: A 3D vector (list of 3 floats) storing the previous arcball vector [S3, unofficial].
*   `rotation`: A quaternion (4 floats) representing the cumulative rotation [S1, S4, unofficial].
*   `translation`: A 4D vector or 4x4 matrix component representing zoom/pan offset [S1, unofficial].
*   `centerTranslation`: A 4x4 matrix representing the pan offset relative to the center [S1, unofficial].
*   `trackball_x_form`: A 16-float array (4x4 column-major matrix) for intermediate transformations [S3, unofficial].

### Steps

1.  **Initialize Camera Basis**:
    Compute the camera basis vectors from the eye $\mathbf{e}$, center $\mathbf{c}$, and up vector $\mathbf{up}$ [S1, unofficial]:
    $$ \mathbf{z}_{\text{axis}} = \text{normalize}(\mathbf{c} - \mathbf{e}) $$
    $$ \mathbf{x}_{\text{axis}} = \text{normalize}(\text{cross}(\mathbf{z}_{\text{axis}}, \mathbf{up})) $$
    $$ \mathbf{y}_{\text{axis}} = \text{normalize}(\text{cross}(\mathbf{x}_{\text{axis}}, \mathbf{z}_{\text{axis}})) $$
    Build a rotation matrix from $\mathbf{x}_{\text{axis}}$, $\mathbf{y}_{\text{axis}}$, and $-\mathbf{z}_{\text{axis}}$, then convert it to a quaternion [S1, unofficial]. Store translation and centerTranslation matrices [S1, unofficial].

2.  **Map Screen to Arcball (Screen-to-Arcball)**:
    For a given mouse position $(x, y)$, compute the normalized arcball vector $\mathbf{v}$ [S1, S2, S3, unofficial]:
    a. Normalize coordinates to $[-1, 1]$ range, inverting screen $y$ [S2, S3, unofficial]:
    $$ v_x = \frac{2.0 \cdot x - W}{R_{\text{ball}}} $$
    $$ v_y = \frac{H - 2.0 \cdot y}{R_{\text{ball}}} $$
    Where $R_{\text{ball}} = \min(W, H) - 20$ [S3, unofficial].
    b. Calculate the squared distance from the center:
    $$ d^2 = v_x^2 + v_y^2 $$
    c. Compute the $z$-component $v_z$ to project onto the sphere surface [S1, S2, S3, unofficial]:
    *   If $d^2 \le 1.0$ (inside the sphere):
        $$ v_z = \sqrt{1.0 - d^2} $$
    *   If $d^2 > 1.0$ (outside the sphere):
        Normalize the 2D vector to the sphere's equator:
        $$ \text{len} = \sqrt{d^2} $$
        $$ v_x \leftarrow v_x / \text{len}, \quad v_y \leftarrow v_y / \text{len}, \quad v_z = 0.0 $$
        *Note: Some implementations cap $d$ at $1.0$ for a cosine projection when $d > 1.0$ [S3, unofficial].*
    d. The resulting vector $\mathbf{v} = (v_x, v_y, v_z)$ is the arcball vector [S2, unofficial].

3.  **Compute Incremental Rotation**:
    Let $\mathbf{v}_{\text{prev}}$ be the arcball vector from the previous frame and $\mathbf{v}_{\text{curr}}$ be the current one [S2, S3, unofficial].
    a. Check if movement occurred:
    $$ \text{diff} = \mathbf{v}_{\text{curr}} - \mathbf{v}_{\text{prev}} $$
    If $\|\text{diff}\| \approx 0$, skip rotation update [S3, unofficial].
    b. Compute the rotation axis $\mathbf{a}$ and angle $\theta$ [S2, S3, unofficial]:
    $$ \mathbf{a} = \text{cross}(\mathbf{v}_{\text{prev}}, \mathbf{v}_{\text{curr}}) $$
    $$ \theta = \arccos(\text{dot}(\mathbf{v}_{\text{prev}}, \mathbf{v}_{\text{curr}})) $$
    c. Create a quaternion $q_{\text{inc}}$ from axis $\mathbf{a}$ and angle $\theta$:
    $$ q_{\text{inc}} = \text{quat}(\mathbf{a} \cdot \sin(\theta/2), \cos(\theta/2)) $$

4.  **Update Rotation**:
    Multiply the incremental rotation into the current rotation quaternion [S4, unofficial]:
    $$ q_{\text{new}} = \text{normalize}(\Delta R \times R_{\text{old}}) $$
    where $\Delta R$ is $q_{\text{inc}}$ and $R_{\text{old}}$ is $q_{\text{curr}}$ [S4, unofficial].

5.  **Handle Zoom and Pan**:
    *   **Zoom**: Compute a translation vector $\mathbf{t}_{\text{zoom}}$ along the camera's forward axis ($\mathbf{z}_{\text{axis}}$) based on scroll input [S1, unofficial]:
        $$ \mathbf{t}_{\text{zoom}} = [0, 0, \text{amount} \cdot k_z, 0] $$
        Update the translation matrix: $M_{\text{trans}} = M_{\text{trans}} \cdot \text{Translate}(\mathbf{t}_{\text{zoom}})$ [S1, unofficial].
        Clamp the Z-translation component (e.g., $t_{14}$ in a 4x4 matrix) to a minimum value (e.g., $-0.2$) to prevent the camera from passing through the center point [S1, unofficial].
    *   **Pan**: Compute delta from mouse movement, transform it by the inverse camera basis to get world-space delta, and update the center translation [S1, unofficial].

6.  **Update Camera Matrix**:
    Construct the final camera matrix $M_{\text{cam}}$ [S1, unofficial]:
    $$ M_{\text{cam}} = M_{\text{trans}} \cdot \text{Rotate}(q_{\text{new}}) \cdot M_{\text{center}} $$
    Where $M_{\text{center}}$ positions the camera at the center point [S1, unofficial]. Compute the inverse camera matrix $M_{\text{cam}}^{-1}$ for ray generation [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

class ArcballCamera:
    def __init__(self, width, height, eye, center, up):
        self.width = width
        self.height = height
        self.eye = np.array(eye, dtype=float)
        self.center = np.array(center, dtype=float)
        self.up = np.array(up, dtype=float)
        
        # Radius for the trackball [S3, unofficial]
        self.radius = min(width, height) - 20.0
        
        # State [S1, S4, unofficial]
        self.rotation = np.array([0.0, 0.0, 0.0, 1.0]) # w, x, y, z quaternion
        self.translation = np.eye(4) # 4x4 matrix for zoom/pan offset
        self.center_translation = np.eye(4) # 4x4 matrix for panning
        
        # Previous arcball vector [S3, unofficial]
        self.last_arcball = None
        
        # Compute initial basis [S1, unofficial]
        self.update_basis()

    def update_basis(self):
        # Compute camera basis vectors [S1, unofficial]
        z_axis = self.center - self.eye
        z_axis = z_axis / np.linalg.norm(z_axis)
        x_axis = np.cross(z_axis, self.up)
        x_axis = x_axis / np.linalg.norm(x_axis)
        y_axis = np.cross(x_axis, z_axis)
        self.x_axis = x_axis
        self.y_axis = y_axis
        self.z_axis = z_axis

    def screen_to_arcball(self, x, y):
        # Normalize to [-1, 1] and invert Y [S2, S3, unofficial]
        vx = (2.0 * x - self.width) / self.radius
        vy = (self.height - 2.0 * y) / self.radius
        
        dist_sq = vx*vx + vy*vy
        
        if dist_sq <= 1.0:
            vz = math.sqrt(1.0 - dist_sq)
        else:
            # Normalize to equator [S1, S2, unofficial]
            length = math.sqrt(dist_sq)
            vx /= length
            vy /= length
            vz = 0.0
            
        return np.array([vx, vy, vz])

    def quaternion_from_axis_angle(self, axis, angle):
        # axis must be normalized [G]
        axis = axis / np.linalg.norm(axis)
        half_angle = angle / 2.0
        s = math.sin(half_angle)
        return np.array([math.cos(half_angle), axis[0]*s, axis[1]*s, axis[2]*s]) # w, x, y, z

    def quaternion_multiply(self, q1, q2):
        # q1 * q2 [G]
        w1, x1, y1, z1 = q1
        w2, x2, y2, z2 = q2
        w = w1*w2 - x1*x2 - y1*y2 - z1*z2
        x = w1*x2 + x1*w2 + y1*z2 - z1*y2
        y = w1*y2 - x1*z2 + y1*w2 + z1*x2
        z = w1*z2 + x1*y2 - y1*x2 + z1*w2
        return np.array([w, x, y, z])

    def rotate(self, x, y):
        curr_arcball = self.screen_to_arcball(x, y)
        
        if self.last_arcball is not None:
            # Compute rotation axis and angle [S2, S3, unofficial]
            axis = np.cross(self.last_arcball, curr_arcball)
            dot_prod = np.dot(self.last_arcball, curr_arcball)
            # Clamp dot product for numerical stability [G]
            dot_prod = max(-1.0, min(1.0, dot_prod))
            angle = math.acos(dot_prod)
            
            if angle > 1e-6: # Avoid division by zero [S3, unofficial]
                axis = axis / np.linalg.norm(axis)
                q_inc = self.quaternion_from_axis_angle(axis, angle)
                # Update rotation: q_new = normalize(delta * q_old) [S4, unofficial]
                self.rotation = self.quaternion_multiply(q_inc, self.rotation)
                self.rotation = self.rotation / np.linalg.norm(self.rotation)
        
        self.last_arcball = curr_arcball

    def zoom(self, amount, zoom_speed=0.1):
        # Translate along z-axis (forward/backward) [S1, unofficial]
        inv_screen_y = 1.0 / self.height
        t = np.array([0.0, 0.0, amount * inv_screen_y * zoom_speed, 0.0])
        
        # Create translation matrix [G]
        T = np.eye(4)
        T[0:3, 3] = t
        
        # Apply translation: translation = translation * T [S1, unofficial]
        self.translation = self.translation @ T
        
        # Clamp translation[14] (Z component) to -0.2 [S1, unofficial]
        if self.translation[14] >= -0.2:
            self.translation[14] = -0.2

    def pan(self, dx, dy):
        # Compute delta in screen space [S1, unofficial]
        inv_screen_x = 1.0 / self.width
        abs_z = abs(self.translation[14])
        delta = np.array([dx * inv_screen_x * abs_z, dy * inv_screen_y * abs_z, 0.0, 0.0])
        
        # Transform delta by inverse camera to get worldDelta [S1, unofficial]
        inv_camera = np.linalg.inv(self.get_camera_matrix())
        world_delta = inv_camera @ delta
        
        # Create translation matrix from worldDelta [S1, unofficial]
        T = np.eye(4)
        T[0:3, 3] = world_delta[0:3]
        
        # Update centerTranslation: centerTranslation = centerTranslation * T [S1, unofficial]
        self.center_translation = self.center_translation @ T

    def get_camera_matrix(self):
        # Construct rotation matrix from quaternion [G]
        w, x, y, z = self.rotation
        R = np.array([
            [1 - 2*y*y - 2*z*z, 2*x*y - 2*z*w, 2*x*z + 2*y*w],
            [2*x*y + 2*z*w, 1 - 2*x*x - 2*z*z, 2*y*z - 2*x*w],
            [2*x*z - 2*y*w, 2*y*z + 2*x*w, 1 - 2*x*x - 2*y*y]
        ])
        
        # Camera matrix: M = [R | -R*eye] [G]
        t = -R @ self.eye
        M = np.eye(4)
        M[0:3, 0:3] = R
        M[0:3, 3] = t
        
        return M

    def get_inverse_camera_matrix(self):
        return np.linalg.inv(self.get_camera_matrix())
```

### Parameters
*   `radius`: Radius of the virtual trackball. Typical value: $\min(W, H) - 20$ [S3, unofficial].
*   `zoomSpeed`: Scalar multiplier for zoom amount. Typical value: $0.1$ [G].
*   `kMinCameraDistance`: Minimum distance from center to prevent camera inversion. Typical value: $1.25$ [S4, unofficial].
*   `kMaxCameraDistance`: Maximum distance from center. Typical value: $8.0$ [S4, unofficial].
*   `kZoomStepFactor`: Zoom step scaling factor. Typical value: $0.9$ [S4, unofficial].

### Edge Cases and Pitfalls
*   **Precision**: When the dot product of arcball vectors is close to $1$ or $-1$, $\arccos$ becomes unstable. Clamp the dot product to $[-1, 1]$ [G].
*   **Boundaries**: If the mouse is outside the sphere ($d^2 > 1$), the vector must be normalized to the equator ($z=0$) to avoid invalid rotations [S1, S2, S3, unofficial].
*   **Zero Movement**: If the mouse does not move, the difference vector is zero, leading to division by zero in axis normalization. Check for non-zero difference before computing rotation [S3, unofficial].
*   **Parallel/Anti-Parallel Vectors**: When vectors are parallel or anti-parallel, the cross product yields a zero axis. Handle anti-parallel cases by finding an orthogonal axis via cross product with a fixed axis (e.g., X or Y) to avoid zero-length axis [S4, unofficial].
*   **Zoom Clamping**: The camera must be clamped to a minimum distance from the center to prevent passing through it, which causes view inversion [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(1)$ per interaction for rotation, zoom, and pan updates [G].
*   **Memory Complexity**: $O(1)$ for storing camera state (quaternion, vectors, matrices) [G].
*   **Measured Performance**: Not given in the sources.
*   **Strengths**: Intuitive interaction, no gimbal lock (due to quaternions), smooth rotation [S2, S4, unofficial].
*   **Limitations**: Not suitable for precise positioning or automated paths [G].
*   **When to use**: Interactive volume rendering viewers, medical imaging software, 3D model inspection [S4, unofficial].
*   **When not to use**: Automated camera paths, non-interactive rendering, scenes requiring precise metric positioning without user input [G].

## Background
Not given in the sources. The method is implemented in interactive medical volume viewers and 3D rendering applications [S4, unofficial].

## Sources
[S1] Twinklebear/webgl-marching-cubes, https://github.com/Twinklebear/webgl-marching-cubes, fetched 2026-10-26 (unofficial)
[S2] fargiolas/qvrc, https://github.com/fargiolas/qvrc, fetched 2026-10-26 (unofficial)
[S3] faviasono/raycastingrendering, https://github.com/faviasono/raycastingrendering, fetched 2026-10-26 (unofficial)
[S4] wpop/qt-viewer-pro, https://github.com/wpop/qt-viewer-pro, fetched 2026-10-27 (unofficial)
