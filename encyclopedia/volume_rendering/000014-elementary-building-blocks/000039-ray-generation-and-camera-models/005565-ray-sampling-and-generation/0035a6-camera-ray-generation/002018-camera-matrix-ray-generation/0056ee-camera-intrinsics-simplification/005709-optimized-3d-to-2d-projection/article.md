# Optimized 3D to 2D Projection

Also called: 3D to 2D Projection, Screen Space Transform

## Overview
This method transforms 3D world-space vertices of geometric primitives (specifically cube faces) into 2D screen-space coordinates using a simplified perspective projection model [S1, unofficial]. It solves the problem of determining which faces are visible on the screen by combining coordinate transformation, perspective division, and visibility culling (backface, near-plane, and screen-boundary checks) [S1, unofficial].

## How it works
**Inputs:**
- `verts`: Array of shape `(num_faces, 4, 3)` containing 3D world coordinates for the 4 vertices of each face [S1, unofficial].
- `origin`: 3D vector representing the camera position in world space [S1, unofficial].
- `yaw`: Float, rotation angle around the Y-axis (radians) [S1, unofficial].
- `pitch`: Float, rotation angle around the Z-axis (radians) [S1, unofficial].
- `settings`: Object containing `NEAR_PLANE` (float), `FOV` (float), `WIDTH` (int), `HEIGHT` (int) [S1, unofficial].
- `constants`: Object containing `HALF_WIDTH`, `HALF_HEIGHT` [S1, unofficial].

**Outputs:**
- `proj_points`: Array of shape `(num_faces, 4, 2)` containing integer screen coordinates $(x, y)$ [S1, unofficial].
- `depth_per_face`: Array of shape `(num_faces,)` containing the average depth of each face [S1, unofficial].
- `visibility_mask`: Boolean array of shape `(num_faces,)` indicating visible faces [S1, unofficial].

**Data Structures:**
- Vertices are stored as contiguous float32 arrays [S1, unofficial].
- Screen coordinates are packed as int32 [S1, unofficial].
- Depth is stored as float32 [S1, unofficial].

**Parameters:**
- `NEAR`: `settings.NEAR_PLANE` [S1, unofficial].
- `FOV_VAL`: `settings.FOV` [S1, unofficial].
- `_AREA_EPSILON`: `1e-6` [S1, unofficial].

**Steps:**

1. **Translation**: Translate vertices relative to the camera origin [S1, unofficial].
   $$ v_{local} = \text{reshape}(verts, -1, 3) - origin $$
   Reshape `verts` to `(num_faces * 4, 3)` for vectorized subtraction [S1, unofficial].

2. **Yaw Rotation (XY Plane)**: Rotate the local X and Y components around the Y-axis [S1, unofficial].
   $$ rx_y = v_x \cos(\text{yaw}) - v_y \sin(\text{yaw}) $$
   $$ ry_{temp} = v_x \sin(\text{yaw}) + v_y \cos(\text{yaw}) $$
   Where $v_x, v_y$ are the X and Y components of $v_{local}$ [S1, unofficial].

3. **Pitch Rotation (YZ Plane)**: Rotate the temporary Y and Z components around the Z-axis [S1, unofficial].
   $$ ry_p = v_z \sin(\text{pitch}) + ry_{temp} \cos(\text{pitch}) $$
   $$ rz_p = v_z \cos(\text{pitch}) - ry_{temp} \sin(\text{pitch}) $$
   Where $v_z$ is the Z component of $v_{local}$ [S1, unofficial].
   Resulting coordinates are $(rx_y, ry_p, rz_p)$ [S1, unofficial].

4. **Perspective Projection**:
   Clamp depth to prevent division by zero or negative depth artifacts [S1, unofficial]:
   $$ depth_{safe} = \max(ry_p, NEAR) $$
   Calculate screen coordinates [S1, unofficial]:
   $$ screen_x = \left( HALF\_WIDTH + rx_y \cdot \frac{FOV\_VAL}{depth_{safe}} \right)_{\text{int32}} $$
   $$ screen_y = \left( HALF\_HEIGHT - rz_p \cdot \frac{FOV\_VAL}{depth_{safe}} \right)_{\text{int32}} $$
   The implementation uses `np.maximum` for clamping and explicit casting to `np.int32` [S1, unofficial].

5. **Pack Projections**:
   Reshape `screen_x` and `screen_y` back to `(num_faces, 4, 2)` [S1, unofficial]:
   $$ proj\_points = \text{stack}([screen_x, screen_y], \text{axis}=1).\text{reshape}(num\_faces, 4, 2) $$
   Calculate average depth per face [S1, unofficial]:
   $$ depth_{per\_face} = \text{mean}(ry_p.\text{reshape}(num\_faces, 4), \text{axis}=1) $$

6. **Visibility Mask Calculation**:
   Compute a boolean mask for each face based on four criteria [S1, unofficial]:
   - **Behind Near Plane**: Face is behind the camera if all vertices have depth $< NEAR$ [S1, unofficial].
     $$ is\_behind = \text{all}(depths < NEAR, \text{axis}=1) $$
   - **Off Screen X**: Face is off-screen if all vertices are outside horizontal bounds [S1, unofficial].
     $$ is\_off\_x = \text{all}(screen_x < 0, \text{axis}=1) \lor \text{all}(screen_x > WIDTH, \text{axis}=1) $$
   - **Off Screen Y**: Face is off-screen if all vertices are outside vertical bounds [S1, unofficial].
     $$ is\_off\_y = \text{all}(screen_y < 0, \text{axis}=1) \lor \text{all}(screen_y > HEIGHT, \text{axis}=1) $$
   - **Backface Culling**: Use the Shoelace formula on screen-space vertices to determine orientation [S1, unofficial].
     $$ dx_1 = sx_{f,1} - sx_{f,0}, \quad dy_1 = sy_{f,1} - sy_{f,0} $$
     $$ dx_2 = sx_{f,2} - sx_{f,0}, \quad dy_2 = sy_{f,2} - sy_{f,0} $$
     $$ area = (dx_1 \cdot dy_2) - (dy_1 \cdot dx_2) $$
     $$ is\_backface = area > -\_AREA\_EPSILON $$
     A face is considered a backface if the signed area is non-negative (or close to zero) [S1, unofficial].
   Final mask [S1, unofficial]:
   $$ visibility\_mask = \neg (is\_behind \lor is\_off\_x \lor is\_off\_y \lor is\_backface) $$

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def optimized_3d_to_2d_projection(
    verts: np.ndarray,  # Shape: (num_faces, 4, 3)
    origin: np.ndarray, # Shape: (3,)
    yaw: float,
    pitch: float,
    settings: dict,
    constants: dict
) -> tuple:
    """
    Transforms 3D face vertices to 2D screen space and calculates visibility.
    """
    num_faces = verts.shape[0]
    if num_faces == 0:
        return (
            np.empty((0, 4, 2), dtype=np.int32),
            np.empty((0,), dtype=np.float32),
            np.empty((0,), dtype=np.bool_)
        )

    NEAR = settings['NEAR_PLANE']
    FOV_VAL = settings['FOV']
    WIDTH = settings['WIDTH']
    HEIGHT = settings['HEIGHT']
    HALF_WIDTH = constants['HALF_WIDTH']
    HALF_HEIGHT = constants['HALF_HEIGHT']
    AREA_EPSILON = 1e-6

    # 1. Translation
    verts_flat = verts.reshape(-1, 3)
    v_local = verts_flat - origin

    # Extract components
    vx = v_local[:, 0]
    vy = v_local[:, 1]
    vz = v_local[:, 2]

    # 2. Yaw Rotation (XY Plane)
    cos_y = np.cos(yaw)
    sin_y = np.sin(yaw)
    rx_y = (vx * cos_y) - (vy * sin_y)
    ry_temp = (vx * sin_y) + (vy * cos_y)

    # 3. Pitch Rotation (YZ Plane)
    cos_p = np.cos(pitch)
    sin_p = np.sin(pitch)
    ry_p = (vz * sin_p) + (ry_temp * cos_p)
    rz_p = (vz * cos_p) - (ry_temp * sin_p)

    # 4. Perspective Projection
    depth_safe = np.maximum(ry_p, NEAR)
    screen_x = (HALF_WIDTH + (rx_y * FOV_VAL / depth_safe)).astype(np.int32)
    screen_y = (HALF_HEIGHT - (rz_p * FOV_VAL / depth_safe)).astype(np.int32)

    # 5. Pack Projections
    proj_points = np.stack([screen_x, screen_y], axis=1).reshape(num_faces, 4, 2)
    depth_per_face = ry_p.reshape(num_faces, 4).mean(axis=1)

    # 6. Visibility Mask
    sx_f = screen_x.reshape(num_faces, 4)
    sy_f = screen_y.reshape(num_faces, 4)
    depths = ry_p.reshape(num_faces, 4)

    is_behind = np.all(depths < NEAR, axis=1)
    is_off_x = np.all(sx_f < 0, axis=1) | np.all(sx_f > WIDTH, axis=1)
    is_off_y = np.all(sy_f < 0, axis=1) | np.all(sy_f > HEIGHT, axis=1)

    # Backface culling using Shoelace formula
    dx1 = sx_f[:, 1] - sx_f[:, 0]
    dy2 = sy_f[:, 2] - sy_f[:, 0]
    dy1 = sy_f[:, 1] - sy_f[:, 0]
    dx2 = sx_f[:, 2] - sx_f[:, 0]
    area = (dx1 * dy2) - (dy1 * dx2)
    is_backface = area > -AREA_EPSILON

    visibility_mask = ~(is_behind | is_off_x | is_off_y | is_backface)

    return proj_points, depth_per_face, visibility_mask
```

**Edge Cases and Pitfalls:**
- **Zero Faces**: If `num_faces` is 0, the function returns empty arrays immediately [S1, unofficial].
- **Backface Orientation**: The Shoelace formula assumes a specific vertex winding order (counter-clockwise or clockwise) [S1, unofficial]. The source defines backfaces as those with area $> -1e-6$, implying a specific winding convention [S1, unofficial].
- **Integer Casting**: Screen coordinates are explicitly cast to `int32` during projection [S1, unofficial].

## Operating characteristics
- **Time Complexity**: $O(N)$ where $N$ is the number of faces, using vectorized operations [G].
- **Memory Complexity**: $O(N)$ for intermediate arrays and outputs [G].
- **Performance**: Optimized for bulk processing of voxel faces via vectorized NumPy operations, avoiding Python loops [S1, unofficial].
- **Strengths**: Simple, fast, and explicit. No matrix library dependencies [S1, unofficial].
- **Limitations**: Limited to perspective projection with fixed FOV. Not suitable for complex camera models. Backface culling is approximate due to epsilon [S1, unofficial].

## Background
This method is derived from the `pyvorengi-sdk-demo` project, an unofficial SDK for a voxel-based game engine [S1, unofficial]. It implements a simplified screen-space transform for rendering voxel chunks [S1, unofficial]. The approach is tailored for real-time voxel rendering where performance is critical and complex camera models are unnecessary [S1, unofficial].

## Sources
[S1] herbal1st/pyvorengi-sdk-demo, https://github.com/herbal1st/pyvorengi-sdk-demo, fetched 2026-10-27 (unofficial)
