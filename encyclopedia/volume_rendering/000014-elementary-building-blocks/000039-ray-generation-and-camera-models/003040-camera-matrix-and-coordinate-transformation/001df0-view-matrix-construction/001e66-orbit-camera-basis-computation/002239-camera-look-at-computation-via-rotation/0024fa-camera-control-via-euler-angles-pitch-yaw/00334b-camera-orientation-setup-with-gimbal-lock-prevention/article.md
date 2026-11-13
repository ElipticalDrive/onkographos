# Camera Orientation Setup with Gimbal Lock Prevention

Also called: Camera Matrix Setup, Spherical Camera Orientation with Up-Vector Flip, setCam, setCamera

## Overview
This method computes a camera's position, look-at target, and up-vector from spherical coordinates, explicitly handling the singularity where the camera points downwards or backwards to prevent gimbal lock and inverted views [S2, unofficial]. It constructs a 3x3 rotation matrix representing the camera's orientation using up, right, and forward vectors [S1, unofficial].

## How it works
### Inputs and Outputs
- **Inputs**: `cam_vz` (distance/zoom), `cam_dx` (horizontal angle), `cam_dy` (vertical angle), `cameraRoll` (optional roll angle), and `cameraTarget` (look-at point) [S1, unofficial][S2, unofficial].
- **Outputs**: `cameraPosition`, `cameraUp`, `cameraRight`, `cameraForward`, and `viewMatrix` [S1, unofficial][S2, unofficial].

### Data Structures
- **Vector**: 3-component float vector `(x, y, z)` [S2, unofficial].
- **Matrix**: 3x3 column-major matrix composed of basis vectors `[S1, unofficial]`.

### Steps
1. Compute camera position using spherical coordinates [S2, unofficial].
2. Set look-at target to `(0, 0, 0)` [S2, unofficial].
3. Compute forward vector: `cameraForward = normalize(cameraTarget - cameraPosition)` [S1, unofficial].
4. Calculate vertical component: `val = cam_vz * cos(cam_dy)` [S2, unofficial].
5. Determine initial up-vector sign: if `val >= 0`, set `up_vektor = 1.0`; else `up_vektor = -1.0` [S2, unofficial].
6. Flip sign for backward orientation: if `cam_vz < 0`, set `up_vektor = -up_vektor` [S2, unofficial].
7. Set up-vector to `(0, up_vektor, 0)` [S2, unofficial].
8. Compute right vector: `cameraRight = cross(cameraUp, cameraForward)` [S1, unofficial].
9. Construct 3x3 matrix from columns `[cameraUp, cameraRight, cameraForward]` [S1, unofficial].

### Formulas
- Position: $\mathbf{p} = \begin{pmatrix} -v \sin \theta_x \cos \theta_y \\ v \sin \theta_y \\ -v \cos \theta_x \cos \theta_y \end{pmatrix}$ where $v = cam\_vz$, $\theta_x = cam\_dx$, $\theta_y = cam\_dy$ [S2, unofficial].
- Up-vector: $\mathbf{u} = \begin{pmatrix} 0 \\ \pm 1 \\ 0 \end{pmatrix}$ [S2, unofficial].
- The up-vector flips sign if $v \cos \theta_y < 0$ or $v < 0$ [S2, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def setup_camera_orientation(cam_vz, cam_dx, cam_dy, camera_roll=0.0, target=(0.0, 0.0, 0.0)):
    # Step 1: Compute camera position
    x = -cam_vz * math.sin(cam_dx) * math.cos(cam_dy)
    y = cam_vz * math.sin(cam_dy)
    z = -cam_vz * math.cos(cam_dx) * math.cos(cam_dy)
    position = (x, y, z)

    # Step 2: Target is fixed at origin
    # target = (0.0, 0.0, 0.0)

    # Step 3: Compute forward vector
    dx = target[0] - position[0]
    dy = target[1] - position[1]
    dz = target[2] - position[2]
    length = math.sqrt(dx*dx + dy*dy + dz*dz)
    if length == 0:
        forward = (0.0, 0.0, -1.0) # Guard against zero length [G]
    else:
        forward = (dx/length, dy/length, dz/length)

    # Step 4-6: Determine up-vector sign to prevent inversion
    val = cam_vz * math.cos(cam_dy)
    if val >= 0:
        up_vektor = 1.0
    else:
        up_vektor = -1.0
    if cam_vz < 0:
        up_vektor *= -1.0

    # Step 7: Set up-vector
    up = (0.0, up_vektor, 0.0)

    # Step 8: Compute right vector
    right_x = up[1]*forward[2] - up[2]*forward[1]
    right_y = up[2]*forward[0] - up[0]*forward[2]
    right_z = up[0]*forward[1] - up[1]*forward[0]
    right = (right_x, right_y, right_z)

    return position, up, right, forward
```

### Parameters
- `cam_vz`: Camera distance/zoom [S2, unofficial].
- `cam_dx`: Horizontal rotation angle [S2, unofficial].
- `cam_dy`: Vertical rotation angle [S2, unofficial].
- `cameraRoll`: Roll angle [S1, unofficial].
- Clipping planes: near `0.1f`, far `30.0f` [S2, unofficial].
- Typical values for angles and distance: Not given in the sources.

### Edge Cases and Pitfalls
- **Gimbal Lock/Inversion**: Occurs when the camera points straight down or backwards [S2, unofficial].
- **Zero Length Forward Vector**: If the camera position equals the target, normalization fails [S1, unofficial].
- **Precision**: Floating point comparisons for `val >= 0` are sensitive to precision near zero [G].

## Operating characteristics
- **Time Complexity**: $O(1)$ per frame [G].
- **Memory Complexity**: $O(1)$ [G].
- **Strengths**: Explicitly handles the singularity at the poles for spherical cameras [S2, unofficial].
- **Limitations**: The flip logic is specific to the coordinate system and angle conventions used in [S2, unofficial] [S2, unofficial].
- **When to Use**: Spherical camera controls where the user rotates around a central point [S2, unofficial].
- **When Not to Use**: Not given in the sources.

## Background
This method originates from implementations of spherical camera controls in volume rendering and 3D visualization tools [S2, unofficial]. The specific up-vector flip logic is documented in the `parallel-volume-rendering` repository [S2, unofficial]. The basis construction using roll is found in the `DivineVoxelEngine` [S1, unofficial].

## Sources
[S1] Divine-Star-Software/DivineVoxelEngine, https://github.com/Divine-Star-Software/DivineVoxelEngine, fetched 2026-10-26 (unofficial)
[S2] besnik/parallel-volume-rendering, https://github.com/besnik/parallel-volume-rendering, fetched 2026-10-27 (unofficial)
