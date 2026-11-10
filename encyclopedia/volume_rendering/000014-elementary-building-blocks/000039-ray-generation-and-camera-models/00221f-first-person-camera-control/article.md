# First-Person Camera Control

Also called: Camera Control, Camera Controller, Camera State & GPU Data Packing, Input Handler, Mouse and Keyboard Input Controller

## Overview
First-Person Camera Control manages the position, orientation (yaw/pitch), and projection parameters of a virtual camera, mapping user input (mouse/keyboard) to transformations for 3D rendering [S1, unofficial][S2, unofficial][S3, unofficial]. It differs from spherical or arcball models by maintaining a fixed "up" vector and allowing free translation along the camera's local axes, which is essential for immersive exploration or precise volumetric inspection [S1, unofficial][S3, unofficial]. It builds on [Ray generation and camera models](770-ray-generation-and-camera-models.md) by providing the specific $\mathbf{o}$ and $\mathbf{d}$ generation logic for a perspective view.

## How it works
**Inputs and Outputs:**
*   **Inputs:** Mouse movement deltas ($\Delta x, \Delta y$), keyboard state (active keys), time delta ($\Delta t$), and initial camera state (position, yaw, pitch).
*   **Outputs:** Updated camera position, orientation vectors (front, right, up), and a packed GPU buffer containing these parameters for ray generation shaders.

**Data Structures:**
The camera state is typically maintained in CPU memory for logic and packed into a GPU-friendly struct for rendering passes [S1, unofficial].

*   **CPU State (S3, unofficial):**
    *   `_position`: `vec3` (x, y, z)
    *   `_front`: `vec3` (normalized direction vector)
    *   `_right`: `vec3` (normalized right vector)
    *   `_up`: `vec3` (normalized up vector)
    *   `_yaw`: `float` (horizontal rotation, default $\pi/2$)
    *   `_pitch`: `float` (vertical rotation, default $0.0$)
    *   `_speed`: `float` (movement speed)
    *   `_sensitivity`: `float` (mouse sensitivity)
    *   `_zoom`: `float` (field of view factor)
    *   `_aspectRatio`: `float`
    *   `_nearPlane`: `float`
    *   `_farPlane`: `float`
    *   `_worldUp`: `vec3` (default $(0, 1, 0)$)

*   **GPU Struct (S1, unofficial):**
    *   `position`: `array<f32, 4>` (xyz position, w unused or padding)
    *   `rot`: `array<array<f32, 4>, 3>` (3 rows of 4 floats, representing the rotation matrix columns or basis vectors)
    *   `sin_half_vfov`: `f32`
    *   `frame_count`: `u32`
    *   `width`: `f32`
    *   `height`: `f32`
    *   `prev_position`: `array<f32, 4>`

**Parameters (Typical Values):**
*   Vertical Field of View (`vfov`): $\pi/2$ radians ($90^\circ$) [S1, unofficial].
*   Movement Speed: $1.0$ units/sec [S1, unofficial].
*   Mouse Sensitivity/Rotation Factor: $0.01$ or $0.0025$ [S1, unofficial][S2, unofficial].
*   Pan Factor: $1.0$ [S2, unofficial].
*   Zoom Factor: $0.0025$ [S2, unofficial].

**Algorithm Steps:**

1.  **Process Input:**
    *   Read mouse deltas ($\Delta x, \Delta y$) and active keyboard keys [S2, unofficial][S3, unofficial].
    *   If mouse button is left and Control key is held, update lighting parameters (if applicable) [S2, unofficial].
    *   If mouse button is left and Control is not held, update rotation:
        $$ \Delta \text{yaw} = \Delta x \times \text{ROTATION\_FACTOR} $$
        $$ \Delta \text{pitch} = \Delta y \times \text{ROTATION\_FACTOR} $$
    *   If mouse button is right, update pan (position offset):
        $$ \Delta \text{pan}_x = \Delta x \times \text{PAN\_FACTOR} $$
        $$ \Delta \text{pan}_y = \Delta y \times \text{PAN\_FACTOR} $$
    *   Store active keys on key-down; clear on key-up [S2, unofficial].

2.  **Update Orientation (Yaw/Pitch):**
    *   Apply rotation deltas to `_yaw` and `_pitch` [S3, unofficial].
    *   Clamp `_pitch` to $[-\pi/2 + \epsilon, \pi/2 - \epsilon]$ to prevent gimbal lock or flipping [G].
    *   Recalculate basis vectors using the new angles [S1, unofficial][S3, unofficial]:
        $$ \text{front}_x = \cos(\text{pitch}) \sin(\text{yaw}) $$
        $$ \text{front}_y = -\sin(\text{pitch}) $$
        $$ \text{front}_z = \cos(\text{pitch}) \cos(\text{yaw}) $$
        $$ \text{front} = \text{normalize}(\text{front}) $$
        $$ \text{right} = \text{normalize}(\text{cross}(\text{front}, \text{worldUp})) $$
        $$ \text{up} = \text{cross}(\text{right}, \text{front}) $$
    *   *Note:* Some implementations compute `vec_x` and `vec_y` from `vec_z` (front) using cross products with a fixed up vector [S1, unofficial].

3.  **Update Position:**
    *   Determine movement direction based on active keys (WASD/Arrows) [S3, unofficial].
    *   Calculate displacement vector $\mathbf{v}$:
        $$ \mathbf{v} = (\text{forward} \times \text{front} + \text{right} \times \text{right} + \text{up} \times \text{up}) \times \text{speed} \times \Delta t $$
    *   Update position:
        $$ \text{position} = \text{position} + \mathbf{v} $$
    *   *Note:* If pan mode is active (right mouse), apply $\Delta \text{pan}$ to position directly or relative to view plane [S2, unofficial].

4.  **Pack GPU Data:**
    *   Compute `sin_half_vfov = sin(0.5 * vfov)` [S1, unofficial].
    *   Construct the rotation matrix rows/columns from `front`, `right`, `up` [S1, unofficial].
    *   Fill `CameraGpuData` struct with `position`, `rot`, `sin_half_vfov`, `frame_count`, `width`, `height`, and `prev_position` [S1, unofficial].
    *   Upload to GPU buffer (e.g., Uniform Buffer Object or Storage Buffer) [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

class FirstPersonCamera:
    def __init__(self, position=[0.0, 0.0, 0.0], yaw=math.pi / 2, pitch=0.0):
        self.position = list(position)
        self.yaw = yaw
        self.pitch = pitch
        self.world_up = [0.0, 1.0, 0.0]
        self.front = [0.0, 0.0, 0.0]
        self.right = [0.0, 0.0, 0.0]
        self.up = [0.0, 0.0, 0.0]
        self.speed = 1.0
        self.sensitivity = 0.01
        self.vfov = math.pi / 2
        
        self._update_vectors()

    def _update_vectors(self):
        # Calculate front vector
        x = math.cos(self.pitch) * math.sin(self.yaw)
        y = -math.sin(self.pitch)
        z = math.cos(self.pitch) * math.cos(self.yaw)
        self.front = self._normalize([x, y, z])
        
        # Calculate right and up vectors
        self.right = self._normalize(self._cross(self.front, self.world_up))
        self.up = self._normalize(self._cross(self.right, self.front))

    def _normalize(self, v):
        length = math.sqrt(v[0]**2 + v[1]**2 + v[2]**2)
        if length == 0: return [0, 0, 0]
        return [v[0]/length, v[1]/length, v[2]/length]

    def _cross(self, a, b):
        return [
            a[1]*b[2] - a[2]*b[1],
            a[2]*b[0] - a[0]*b[2],
            a[0]*b[1] - a[1]*b[0]
        ]

    def process_mouse(self, dx, dy):
        self.yaw += dx * self.sensitivity
        self.pitch += dy * self.sensitivity
        # Clamp pitch to avoid flipping
        if self.pitch > 89.0 * math.pi / 180.0:
            self.pitch = 89.0 * math.pi / 180.0
        if self.pitch < -89.0 * math.pi / 180.0:
            self.pitch = -89.0 * math.pi / 180.0
        self._update_vectors()

    def process_keyboard(self, direction, delta_time):
        velocity = self.speed * delta_time
        if direction == 'FORWARD':
            self.position[0] += self.front[0] * velocity
            self.position[1] += self.front[1] * velocity
            self.position[2] += self.front[2] * velocity
        elif direction == 'BACKWARD':
            self.position[0] -= self.front[0] * velocity
            self.position[1] -= self.front[1] * velocity
            self.position[2] -= self.front[2] * velocity
        elif direction == 'RIGHT':
            self.position[0] += self.right[0] * velocity
            self.position[1] += self.right[1] * velocity
            self.position[2] += self.right[2] * velocity
        elif direction == 'LEFT':
            self.position[0] -= self.right[0] * velocity
            self.position[1] -= self.right[1] * velocity
            self.position[2] -= self.right[2] * velocity

    def get_gpu_data(self, width, height, frame_count):
        sin_half_vfov = math.sin(self.vfov / 2.0)
        # Rotation matrix rows (or columns depending on shader expectation)
        # Here we pack basis vectors as rows for simplicity
        rot = [
            self.front, # Row 0
            self.right, # Row 1
            self.up     # Row 2
        ]
        # Pad to 4 floats per row if required by struct
        rot_packed = [
            self.front + [0.0],
            self.right + [0.0],
            self.up + [0.0]
        ]
        
        return {
            "position": self.position + [0.0],
            "rot": rot_packed,
            "sin_half_vfov": sin_half_vfov,
            "frame_count": frame_count,
            "width": float(width),
            "height": float(height),
            "prev_position": self.position + [0.0] # Update prev in actual loop
        }
```

## Subtopics

### [Cinematic Camera Metaphor](7134-cinematic-camera-metaphor.md)
The Cinematic Camera Metaphor is a navigation technique for non-immersive 3D environments where user input manipulates the position and orientation of a virtual camera to control the viewpoint. It enables users to "tumble, track, and dolly" the view, allowing for precise framing of the scene. It is distinguished from direct manipulation metaphors (like Scene-in-Hand) by treating the camera as an independent entity controlled via standard input devices, often requiring modifier keys or specific gestures to invoke different camera motions.

### setCamera
A programmatic interface for setting camera state, often used in scientific visualization toolkits (e.g., VTK.js) [S4, unofficial]. It handles synchronization between requested state and applied state, ensuring that changes are only applied if they differ from the last request [S4, unofficial]. It supports setting position, focal point, view up, and zoom (parallel scale) [S4, unofficial]. A key pitfall is handling XR mode, where headset pose should override programmatic control, and ensuring that unchanged camera steps do not reset the view unexpectedly [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per frame for input processing and matrix updates.
*   **Memory Complexity:** $O(1)$ for camera state storage.
*   **Performance:** Highly efficient; the primary cost is the CPU-to-GPU upload of the camera uniform buffer, which can be batched or updated only when state changes [S1, unofficial].
*   **Strengths:** Intuitive for exploration; supports free movement and rotation; standard in FPS games and medical imaging viewers.
*   **Limitations:** Prone to gimbal lock if not implemented with quaternions or careful clamping (though yaw/pitch avoids singularity at poles, it restricts full spherical rotation) [G]. Not suitable for orbiting around a specific point without additional logic (pan/trackball hybrid).
*   **When to use:** Immersive navigation, medical volume inspection, FPS-style viewers.
*   **When not to use:** When a fixed orbit around a center point is required (use Arcball) or when spherical sampling is needed (use Uniform Sphere Sampling).

## Background
First-person camera control is a standard technique in computer graphics, originating from first-person shooter games and virtual reality applications. In the context of volume rendering, it allows users to navigate through volumetric datasets (e.g., medical CT/MRI scans) with intuitive controls. The specific implementation details (yaw/pitch decomposition, GPU struct packing) are common in modern WebGL and WebGPU-based volume renderers [S1, unofficial][S2, unofficial][S3, unofficial].

## Sources
[S1] newDINO/voxel_ray_demo, https://github.com/newDINO/voxel_ray_demo, fetched 2026-10-24 (unofficial)
[S2] fraserlove/ossium, https://github.com/fraserlove/ossium, fetched 2026-10-27 (unofficial)
[S3] Justinj68/OM3D_PROJECT, https://github.com/Justinj68/OM3D_PROJECT, fetched 2026-10-27 (unofficial)
[S4] kevinkunkel98/reinforcement-learning-volume-rendering-transfer-function, https://github.com/kevinkunkel98/reinforcement-learning-volume-rendering-transfer-function, fetched 2026-10-27 (unofficial)
