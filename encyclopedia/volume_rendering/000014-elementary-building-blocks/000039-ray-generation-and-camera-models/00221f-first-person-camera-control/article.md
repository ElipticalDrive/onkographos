# First-Person Camera Control

Also called: Camera Control, Camera Controller, Camera State & GPU Data Packing, Input Handler, Mouse and Keyboard Input Controller

## Overview
First-Person Camera Control manages camera position, orientation (yaw and pitch), and projection parameters, translating discrete user input (mouse movement, keyboard presses) into continuous 3D transformations [S2, S3, unofficial]. It packs these state variables into a GPU-friendly structure to minimize bandwidth overhead during rendering passes [S1, unofficial].

## How it works
The method consists of three phases: input handling, state update, and GPU data packing [S1, S2, S3, unofficial].

### Inputs and Outputs
**Inputs:**
- Mouse movement deltas ($\Delta x, \Delta y$) and active button state [S2, unofficial].
- Keyboard key states (WASD or arrow keys for movement) [S3, unofficial].
- Time delta ($\Delta t$) for frame-rate independent movement [S3, unofficial].
- Projection parameters: vertical field of view ($\text{vfov}$), aspect ratio, near/far planes [S3, unofficial].

**Outputs:**
- A packed camera state structure sent to the GPU for ray generation [S1, unofficial].

### Data Structures
The GPU-friendly structure `CameraGpuData` contains the following fields [S1, unofficial]:
- `position`: `float[4]` (x, y, z, padding) [S1, unofficial].
- `rot`: `float[3][4]` (3 rows, 4 columns; represents the rotation basis) [S1, unofficial].
- `sin_half_vfov`: `float` [S1, unofficial].
- `frame_count`: `uint` [S1, unofficial].
- `width`: `float` [S1, unofficial].
- `height`: `float` [S1, unofficial].
- `prev_position`: `float[4]` (used for temporal accumulation or motion vectors) [S1, unofficial].

The internal CPU-side state includes [S3, unofficial]:
- `_position`: `vec3` [S3, unofficial].
- `_yaw`: `float` (horizontal rotation) [S3, unofficial].
- `_pitch`: `float` (vertical rotation) [S3, unofficial].
- `_front`: `vec3` (normalized forward vector) [S3, unofficial].
- `_right`: `vec3` (normalized right vector) [S3, unofficial].
- `_up`: `vec3` (normalized up vector) [S3, unofficial].
- `_worldUp`: `vec3` (default $(0, 1, 0)$) [S3, unofficial].
- `_speed`: `float` (movement speed) [S3, unofficial].
- `_sensitivity`: `float` (mouse sensitivity) [S3, unofficial].

### Parameters
Typical default values [S1, S2, S3, unofficial]:
- `vfov`: $\pi \times 0.5$ radians (90 degrees) [S1, unofficial].
- `moving_speed`: $1.0$ units/second [S1, unofficial].
- `mouse_speed`: $0.01$ radians/pixel [S1, unofficial].
- `ROTATION_FACTOR`: $0.0025$ [S2, unofficial].
- `PAN_FACTOR`: $1.0$ [S2, unofficial].
- `LIGHT_FACTOR`: $0.0025$ [S2, unofficial].
- `ZOOM_FACTOR`: $0.0025$ [S2, unofficial].

### Steps

1. **Initialize State**: Set `_position` to the starting eye coordinates. Set `_yaw` to $0.0$ and `_pitch` to $0.0$ [G].

2. **Process Input**:
   - **Mouse**: Read $\Delta x$ and $\Delta y$.
     - If left button is active and Control key is held: Update lighting parameters using $\Delta x \times \text{LIGHT\_FACTOR}$ and $\Delta y \times \text{LIGHT\_FACTOR}$ [S2, unofficial].
     - If left button is active and Control key is not held: Update rotation angles:
       $$ \text{yaw} \leftarrow \text{yaw} + \Delta x \times \text{ROTATION\_FACTOR} $$
       $$ \text{pitch} \leftarrow \text{pitch} + \Delta y \times \text{ROTATION\_FACTOR} $$
     - If right button is active: Update camera pan (position offset) using $\Delta x \times \text{PAN\_FACTOR}$ and $\Delta y \times \text{PAN\_FACTOR}$ [S2, unofficial].
   - **Keyboard**: Track active keys (W, A, S, D, Q, E) to determine movement direction [S2, S3, unofficial].

3. **Update Camera Vectors**: Recalculate the basis vectors based on current `_yaw` and `_pitch` [S3, unofficial]:
   $$ \text{vec\_z} = (\cos(\text{pitch})\sin(\text{yaw}), -\sin(\text{pitch}), \cos(\text{pitch})\cos(\text{yaw})) $$
   $$ \text{vec\_y} = \text{normalize}(\text{cross}(\text{vec\_z}, \text{world\_up})) $$
   $$ \text{vec\_x} = \text{normalize}(\text{cross}(\text{vec\_y}, \text{vec\_z})) $$
   Assign `_front = vec_z`, `_right = vec_x`, `_up = vec_y` [S1, S3, unofficial].

4. **Update Position**: If movement keys are active, modify `_position` along the basis vectors [S3, unofficial]:
   $$ \text{velocity} = \text{speed} \times \Delta t $$
   $$ \text{position} \leftarrow \text{position} + \text{velocity} \times (\text{front} \times \text{forward\_key} + \text{right} \times \text{right\_key} + \text{up} \times \text{up\_key}) $$
   Note: "Forward" key moves along `-front` if front points into the scene, or `front` depending on coordinate system convention [G].

5. **Pack GPU Data**: Construct `CameraGpuData` [S1, unofficial]:
   - `prev_position = position` [S1, unofficial].
   - `rot`: Pack `vec_x`, `vec_y`, `vec_z` into the three rows of the rotation matrix [S1, unofficial].
   - `sin_half_vfov`: $\sin(\text{vfov} / 2)$ [S1, unofficial].
   - `width`, `height`: Current viewport dimensions [S1, unofficial].
   - `frame_count`: Incremented per frame [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

class FirstPersonCamera:
    def __init__(self, position=(0.0, 0.0, 0.0), vfov=math.pi * 0.5):
        self.position = list(position)
        self.yaw = 0.0
        self.pitch = 0.0
        self.speed = 1.0
        self.vfov = vfov
        self.world_up = [0.0, 1.0, 0.0]
        self._update_vectors()

    def _update_vectors(self):
        # S1 formula for front vector (vec_z)
        vec_z = [
            math.cos(math.radians(self.pitch)) * math.sin(math.radians(self.yaw)),
            -math.sin(math.radians(self.pitch)),
            math.cos(math.radians(self.pitch)) * math.cos(math.radians(self.yaw))
        ]
        # S1 dependency order: vec_y from cross(vec_z, world_up), vec_x from cross(vec_y, vec_z)
        vec_y = _normalize(_cross(vec_z, self.world_up))
        vec_x = _normalize(_cross(vec_y, vec_z))
        
        self.front = vec_z
        self.right = vec_x
        self.up = vec_y

    def process_mouse(self, dx, dy, button=0, keys=None):
        if keys is None:
            keys = {}
        
        if button == 0: # Left click
            if keys.get('Control'):
                # Update lighting parameters using S2 factors
                pass
            else:
                # S2: ROTATION_FACTOR = 0.0025
                self.yaw += dx * 0.0025
                self.pitch += dy * 0.0025
                self._update_vectors()
        elif button == 2: # Right click
            # S2: PAN_FACTOR = 1.0
            self.position[0] += dx * 1.0
            self.position[1] += dy * 1.0

    def process_keyboard(self, dt, keys):
        velocity = self.speed * dt
        direction = [0.0, 0.0, 0.0]
        
        if keys.get('W'): # Forward
            direction[0] -= self.front[0]
            direction[1] -= self.front[1]
            direction[2] -= self.front[2]
        if keys.get('S'): # Backward
            direction[0] += self.front[0]
            direction[1] += self.front[1]
            direction[2] += self.front[2]
        if keys.get('A'): # Left
            direction[0] -= self.right[0]
            direction[1] -= self.right[1]
            direction[2] -= self.right[2]
        if keys.get('D'): # Right
            direction[0] += self.right[0]
            direction[1] += self.right[1]
            direction[2] += self.right[2]
        if keys.get('Q'): # Down
            direction[1] -= 1.0
        if keys.get('E'): # Up
            direction[1] += 1.0
            
        if any(d != 0 for d in direction):
            direction = _normalize(direction)
            self.position[0] += direction[0] * velocity
            self.position[1] += direction[1] * velocity
            self.position[2] += direction[2] * velocity

    def get_gpu_data(self, frame_count, width, height, prev_position):
        # S1: Pack basis vectors into rot matrix
        rot = [self.right, self.up, self.front]
        
        return {
            "position": self.position + [0.0],
            "rot": rot,
            "sin_half_vfov": math.sin(self.vfov / 2.0),
            "frame_count": frame_count,
            "width": float(width),
            "height": float(height),
            "prev_position": prev_position + [0.0]
        }

def _normalize(v):
    length = math.sqrt(sum(x*x for x in v))
    if length == 0:
        return [0.0, 0.0, 0.0]
    return [x / length for x in v]

def _cross(a, b):
    return [
        a[1]*b[2] - a[2]*b[1],
        a[2]*b[0] - a[0]*b[2],
        a[0]*b[1] - a[1]*b[0]
    ]
```

### Edge Cases and Pitfalls
- **Temporal Coherence**: `prev_position` must be updated at the end of each frame to avoid jitter in temporal accumulation passes [S1, unofficial].
- **Coordinate Conventions**: The sign of movement along `_front` depends on the scene's coordinate system (left-handed vs right-handed) [G].
- **Input Clamping**: Unclamped pitch values will cause the camera to invert; standard implementations clamp pitch to $[-89^\circ, 89^\circ]$ [G].

## Subtopics

### [Arcball Camera](1639-arcball-camera.md)
Rotates the camera around a fixed target point using spherical coordinates; suitable for inspecting objects from all angles.

### [Camera Trajectory and Path Generation](6410-camera-trajectory-and-path-generation.md)
Interpolates camera positions and orientations along a predefined path; suitable for fly-throughs and animations.

### [Camera Matrix and Coordinate Transformation](6231-camera-matrix-and-coordinate-transformation.md)
Focuses on the mathematical construction of view and projection matrices; foundational for all camera models.

### [Depth of Field and Lens Effects](6232-depth-of-field-and-lens-effects.md)
Adds optical realism by blurring regions based on distance; requires additional camera parameters like aperture and focal length.

### Cinematic Camera Metaphor
A navigation technique where mouse movements coupled with modifier keys manipulate the position of a virtual camera to control the user's viewpoint [S4, unofficial]. Variants include mapping a 6DoF input device directly to the eyeball position (absolute or relative mapping) or controlling a virtual vehicle's spatial and angular velocity [S4, unofficial]. Relative mapping can destroy the user's mental model of a fixed scene, causing confusion [S4, unofficial]. Implemented in systems like Unicam and StyleCam [S4, unofficial].

## Operating characteristics
- **Time Complexity**: $O(1)$ per frame for input processing and state update, involving a fixed number of vector operations and trigonometric functions [G].
- **Memory Complexity**: $O(1)$ for CPU-side state and $O(1)$ for GPU-side data (fixed-size struct) [S1, unofficial].
- **Strengths**: Intuitive control for navigation; low computational overhead; decouples viewing direction from position [S3, unofficial].
- **Limitations**: Requires additional logic for orbital inspection or automated path-based visualization [G].
- **When to use**: Interactive exploration of volumetric datasets where the user needs to "walk through" or look around a scene [S3, unofficial].
- **When not to use**: When a fixed viewpoint, orbital rotation, or scripted animation is required [G].

## Background
First-person camera control is a standard technique in 3D computer graphics, originating from first-person shooter games and virtual reality applications [S3, unofficial]. In the context of volume rendering, it allows users to navigate large volumetric datasets (e.g., medical images, scientific simulations) with intuitive mouse and keyboard inputs [S2, S3, unofficial]. The cinematic camera metaphor extends this control to non-immersive environments, supporting tumbling, tracking, and dollying of viewpoints [S4, unofficial].

## Sources
[S1] newDINO/voxel_ray_demo, https://github.com/newDINO/voxel_ray_demo, fetched 2026-10-24 (unofficial)
[S2] fraserlove/ossium, https://github.com/fraserlove/ossium, fetched 2026-10-27 (unofficial)
[S3] Justinj68/OM3D_PROJECT, https://github.com/Justinj68/OM3D_PROJECT, fetched 2026-10-27 (unofficial)
[S4] Interaction with volumetric displays, http://hdl.handle.net/1807/16733, fetched 2026-10-26 (unofficial)
