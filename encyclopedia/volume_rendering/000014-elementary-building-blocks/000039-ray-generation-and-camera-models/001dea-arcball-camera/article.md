# Arcball Camera

Also called: Arcball Rotation, Screen-to-Arcball, Trackball Camera, Trackball Camera Interaction

## Overview
The Arcball Camera is an interactive camera control technique that maps 2D screen coordinates (typically mouse movements) to a 3D virtual sphere to compute smooth rotation quaternions for orbiting around a center point [S1, S2, S3, unofficial]. It solves the problem of intuitive 3D navigation by decoupling the 2D input device from 3D orientation, allowing users to rotate the view in any direction without gimbal lock or discontinuous jumps. It differs from [First-Person Camera Control](2721-first-person-camera-control.md) by orbiting a fixed center rather than moving the eye position directly, and from [Spiral Camera Path Generation](1635-spiral-camera-path-generation.md) by being interactive and non-parametric.

## How it works
The method maintains a camera state defined by an eye position $\mathbf{e}$, a target (center) point $\mathbf{c}$, and an up vector $\mathbf{up}$. It tracks the previous and current mouse positions to compute incremental rotations.

### Inputs and Outputs
*   **Inputs**:
    *   Current mouse coordinates $(x, y)$ and previous mouse coordinates $(x_{prev}, y_{prev})$ in pixel space.
    *   Viewport dimensions: width $W$, height $H$.
    *   Camera parameters: eye $\mathbf{e}$, center $\mathbf{c}$, up vector $\mathbf{up}$, zoom speed $k_z$, pan sensitivity.
    *   Current rotation quaternion $q_{curr}$ and translation vector $\mathbf{t}_{trans}$.
*   **Outputs**:
    *   Updated camera matrix $M_{cam}$ (or eye/target/up vectors).
    *   Updated rotation quaternion $q_{new}$ and translation $\mathbf{t}_{new}$.

### Data Structures
The state typically includes:
*   `width`, `height`: Integers or floats representing viewport dimensions.
*   `last_position`: A 3D vector (or list of 3 floats) storing the previous arcball vector.
*   `rotation`: A quaternion (4 floats) representing the cumulative rotation.
*   `translation`: A 4D vector or 4x4 matrix component representing zoom/pan offset.
*   `centerTranslation`: A 4x4 matrix or vector representing the pan offset relative to the center.
*   `trackball_x_form`: A 16-float array (4x4 column-major matrix) for intermediate transformations [S3, unofficial].

### Steps

1.  **Initialize Camera Basis**:
    Compute the camera basis vectors from the eye $\mathbf{e}$, center $\mathbf{c}$, and up vector $\mathbf{up}$ [S1, unofficial]:
    $$ \mathbf{z}_{axis} = \text{normalize}(\mathbf{c} - \mathbf{e}) $$
    $$ \mathbf{x}_{axis} = \text{normalize}(\text{cross}(\mathbf{z}_{axis}, \mathbf{up})) $$
    $$ \mathbf{y}_{axis} = \text{normalize}(\text{cross}(\mathbf{x}_{axis}, \mathbf{z}_{axis})) $$
    Construct the rotation matrix $R$ from these basis vectors. Convert $R$ to a quaternion $q_{curr}$ [S1, unofficial].

2.  **Map Screen to Arcball (Screen-to-Arcball)**:
    For a given mouse position $(x, y)$, compute the normalized arcball vector $\mathbf{v}$ [S1, S2, S3, unofficial]:
    
    a. Normalize coordinates to $[-1, 1]$ range. Note that screen $y$ is often inverted relative to 3D space [S2, S3, unofficial]:
    $$ v_x = \frac{2.0 \cdot x - W}{R_{ball}} $$
    $$ v_y = \frac{H - 2.0 \cdot y}{R_{ball}} $$
    Where $R_{ball}$ is the radius of the virtual trackball, typically $R_{ball} = \min(W, H) - 20$ [S3, unofficial] or simply $\min(W, H)$ [S1, S2, unofficial].
    
    b. Calculate the squared distance from the center:
    $$ d^2 = v_x^2 + v_y^2 $$
    
    c. Compute the $z$-component $v_z$ to project onto the sphere surface [S1, S2, S3, unofficial]:
    *   If $d^2 \le 1.0$ (inside the sphere):
        $$ v_z = \sqrt{1.0 - d^2} $$
    *   If $d^2 > 1.0$ (outside the sphere):
        Normalize the 2D vector to the sphere's equator:
        $$ \mathbf{v}_{2D} = (v_x, v_y) $$
        $$ \mathbf{v} = \text{normalize}(\mathbf{v}_{2D}, 0) $$
        $$ v_z = 0 $$
        *Note: Some implementations use a cosine projection for $d > 1$ to keep the point on the sphere surface, capping $d$ at 1.0 for the angle calculation [S3, unofficial].*
    
    d. The resulting vector $\mathbf{v} = (v_x, v_y, v_z)$ is the arcball vector.

3.  **Compute Incremental Rotation**:
    Let $\mathbf{v}_{prev}$ be the arcball vector from the previous frame and $\mathbf{v}_{curr}$ be the current one [S2, S3, unofficial].
    
    a. Check if movement occurred:
    $$ \text{diff} = \mathbf{v}_{curr} - \mathbf{v}_{prev} $$
    If $\|\text{diff}\| \approx 0$, skip rotation update [S3, unofficial].
    
    b. Compute the rotation axis $\mathbf{a}$ and angle $\theta$ [S2, S3, unofficial]:
    $$ \mathbf{a} = \text{cross}(\mathbf{v}_{prev}, \mathbf{v}_{curr}) $$
    $$ \theta = \arccos(\text{dot}(\mathbf{v}_{prev}, \mathbf{v}_{curr})) $$
    *Note: $\mathbf{a}$ must be normalized if $\theta$ is small to avoid precision issues.*
    
    c. Create a quaternion $q_{inc}$ from axis $\mathbf{a}$ and angle $\theta$:
    $$ q_{inc} = \text{quat}(\mathbf{a} \cdot \sin(\theta/2), \cos(\theta/2)) $$

4.  **Update Rotation**:
    Multiply the incremental rotation into the current rotation quaternion [S1, S3, unofficial]:
    $$ q_{new} = q_{inc} \otimes q_{curr} $$
    (Order depends on whether rotations are applied in local or global space; [S1, unofficial] suggests multiplying by previous and current ball quaternions in a specific sequence, but standard composition is $q_{new} = q_{inc} \cdot q_{curr}$ for global frame updates).

5.  **Handle Zoom and Pan**:
    *   **Zoom**: Compute a translation vector $\mathbf{t}_{zoom}$ along the camera's forward axis ($\mathbf{z}_{axis}$) based on scroll wheel or pinch input [S1, unofficial]:
        $$ \mathbf{t}_{zoom} = [0, 0, \text{amount} \cdot k_z, 0] $$
        Update the translation matrix: $M_{trans} = M_{trans} \cdot \text{Translate}(\mathbf{t}_{zoom})$.
        Clamp the Z-translation component (e.g., $t_{14}$ in a 4x4 matrix) to a minimum value (e.g., $-0.2$) to prevent the camera from passing through the center point [S1, unofficial].
    *   **Pan**: Compute delta from mouse movement, transform it by the inverse camera basis to get world-space delta, and update the center translation [S1, unofficial].

6.  **Update Camera Matrix**:
    Construct the final camera matrix $M_{cam}$ [S1, unofficial]:
    $$ M_{cam} = M_{trans} \cdot \text{Rotate}(q_{new}) \cdot M_{center} $$
    Where $M_{center}$ positions the camera at the center point. Compute the inverse camera matrix $M_{cam}^{-1}$ for ray generation [S1, unofficial].

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
        
        # Radius for the trackball
        self.radius = min(width, height) - 20.0
        
        # State
        self.rotation = np.array([0.0, 0.0, 0.0, 1.0]) # w, x, y, z quaternion
        self.translation = np.array([0.0, 0.0, 0.0, 0.0]) # 4D vector for translation
        self.center_translation = np.eye(4) # 4x4 matrix for panning
        
        # Previous arcball vector
        self.last_arcball = None
        
        # Compute initial basis
        self.update_basis()

    def update_basis(self):
        # Compute camera basis vectors
        z_axis = self.center - self.eye
        z_axis = z_axis / np.linalg.norm(z_axis)
        x_axis = np.cross(z_axis, self.up)
        x_axis = x_axis / np.linalg.norm(x_axis)
        y_axis = np.cross(x_axis, z_axis)
        self.x_axis = x_axis
        self.y_axis = y_axis
        self.z_axis = z_axis

    def screen_to_arcball(self, x, y):
        # Normalize to [-1, 1]
        # Invert Y for screen coordinates
        vx = (2.0 * x - self.width) / self.radius
        vy = (self.height - 2.0 * y) / self.radius
        
        dist_sq = vx*vx + vy*vy
        
        if dist_sq <= 1.0:
            vz = math.sqrt(1.0 - dist_sq)
        else:
            # Normalize to equator
            length = math.sqrt(dist_sq)
            vx /= length
            vy /= length
            vz = 0.0
            
        return np.array([vx, vy, vz])

    def quaternion_from_axis_angle(self, axis, angle):
        # axis must be normalized
        axis = axis / np.linalg.norm(axis)
        half_angle = angle / 2.0
        s = math.sin(half_angle)
        return np.array([math.cos(half_angle), axis[0]*s, axis[1]*s, axis[2]*s]) # w, x, y, z

    def quaternion_multiply(self, q1, q2):
        # q1 * q2
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
            # Compute rotation axis and angle
            axis = np.cross(self.last_arcball, curr_arcball)
            dot_prod = np.dot(self.last_arcball, curr_arcball)
            # Clamp dot product for numerical stability
            dot_prod = max(-1.0, min(1.0, dot_prod))
            angle = math.acos(dot_prod)
            
            if angle > 1e-6: # Avoid division by zero
                axis = axis / np.linalg.norm(axis)
                q_inc = self.quaternion_from_axis_angle(axis, angle)
                self.rotation = self.quaternion_multiply(q_inc, self.rotation)
        
        self.last_arcball = curr_arcball

    def zoom(self, amount, zoom_speed=0.1):
        # Translate along z-axis (forward/backward)
        # amount is positive for zoom in
        t = np.array([0.0, 0.0, -amount * zoom_speed, 0.0])
        
        # Create translation matrix
        T = np.eye(4)
        T[0:3, 3] = t
        
        # Apply translation
        # Note: Matrix multiplication order depends on convention (row/col major)
        # Assuming column-major and right-multiplication for local transforms
        # Or left-multiplication for global. Here we update the translation vector directly
        # or modify the camera matrix.
        
        # Simplified: update translation vector in camera space
        # Convert t to world space if necessary, but usually zoom is along view direction
        # Let's assume t is in camera local space for simplicity, then transform
        # Actually, standard arcball zoom moves eye along z_axis
        self.eye += self.z_axis * (-amount * zoom_speed)
        
        # Clamp zoom distance to prevent passing through center
        # Distance from eye to center
        dist = np.linalg.norm(self.center - self.eye)
        min_dist = 0.2
        if dist < min_dist:
            # Push eye back
            direction = self.center - self.eye
            direction = direction / np.linalg.norm(direction)
            self.eye = self.center - direction * min_dist

    def pan(self, dx, dy):
        # Pan moves the center point perpendicular to view direction
        # Scale by distance to center
        dist = np.linalg.norm(self.center - self.eye)
        scale = dist * 0.001 # Sensitivity factor
        
        # Compute pan vector in camera space
        pan_x = self.x_axis * (-dx * scale)
        pan_y = self.y_axis * (-dy * scale)
        
        self.center += pan_x + pan_y
        self.eye += pan_x + pan_y # Keep eye relative to center if orbiting

    def get_camera_matrix(self):
        # Construct rotation matrix from quaternion
        w, x, y, z = self.rotation
        # Rotation matrix from quaternion (column-major)
        R = np.array([
            [1 - 2*y*y - 2*z*z, 2*x*y - 2*z*w, 2*x*z + 2*y*w],
            [2*x*y + 2*z*w, 1 - 2*x*x - 2*z*z, 2*y*z - 2*x*w],
            [2*x*z - 2*y*w, 2*y*z + 2*x*w, 1 - 2*x*x - 2*y*y]
        ])
        
        # Camera matrix: M = [R | -R*eye]
        # Or M = [R | t] where t = -R * eye
        t = -R @ self.eye
        
        M = np.eye(4)
        M[0:3, 0:3] = R
        M[0:3, 3] = t
        
        return M

    def get_inverse_camera_matrix(self):
        return np.linalg.inv(self.get_camera_matrix())
```

### Parameters
*   `radius`: Radius of the virtual trackball. Typical value: $\min(W, H) - 20$ [S3, unofficial] or $\min(W, H)$ [S1, S2, unofficial].
*   `zoomSpeed`: Scalar multiplier for zoom amount. Typical value: $0.1$ [G].
*   `minZoom`: Minimum distance from center to prevent camera inversion. Typical value: $0.2$ [S1, unofficial].

### Edge Cases and Pitfalls
*   **Precision**: When the dot product of arcball vectors is close to 1 or -1, $\arccos$ becomes unstable. Clamp the dot product to $[-1, 1]$ [G].
*   **Boundaries**: If the mouse is outside the sphere ($d^2 > 1$), the vector must be normalized to the equator ($z=0$) to avoid invalid rotations [S1, S2, S3, unofficial].
*   **Zero Movement**: If the mouse does not move, the difference vector is zero, leading to division by zero in axis normalization. Check for non-zero difference before computing rotation [S3, unofficial].
*   **Zoom Clamping**: The camera must be clamped to a minimum distance from the center to prevent passing through it, which causes view inversion [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(1)$ per frame for rotation, zoom, and pan updates. Matrix inversion for ray generation is $O(1)$ for 4x4 matrices.
*   **Memory Complexity**: $O(1)$ for storing camera state (quaternion, vectors, matrices).
*   **Strengths**: Intuitive interaction, no gimbal lock (due to quaternions), smooth rotation.
*   **Limitations**: Not suitable for precise positioning or automated paths. Performance is negligible compared to ray marching.
*   **When to use**: Interactive volume rendering viewers, medical imaging software, 3D model inspection.
*   **When not to use**: Automated camera paths, non-interactive rendering, scenes requiring precise metric positioning without user input.

## Background
The Arcball technique was introduced by Ken Shoemake in 1992 [S1, S2, S3, unofficial]. It was designed to provide a natural way to rotate 3D objects on a 2D screen. It is widely used in 3D modeling software, CAD applications, and interactive volume rendering tools. The method maps 2D mouse movements to a 3D sphere, allowing for continuous rotation without singularities.

## Sources
[S1] Twinklebear/webgl-marching-cubes, https://github.com/Twinklebear/webgl-marching-cubes, fetched 2026-10-26 (unofficial)
[S2] fargiolas/qvrc, https://github.com/fargiolas/qvrc, fetched 2026-10-26 (unofficial)
[S3] faviasono/raycastingrendering, https://github.com/faviasono/raycastingrendering, fetched 2026-10-26 (unofficial)
