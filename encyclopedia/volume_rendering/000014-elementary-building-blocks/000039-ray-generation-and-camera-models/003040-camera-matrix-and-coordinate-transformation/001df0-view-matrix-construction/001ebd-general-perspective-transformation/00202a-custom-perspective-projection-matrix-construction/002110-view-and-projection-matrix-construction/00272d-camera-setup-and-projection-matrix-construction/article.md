# Camera Setup and Projection Matrix Construction

Also called: CamSetup, Camera View and Projection Setup

## Overview
This method initializes camera parameters for volume rendering or training, handling downscaling, view matrix construction, and projection matrix calculation [S1, S2, unofficial]. It differs from generic graphics pipelines by explicitly managing intrinsic scaling for multi-resolution training and constructing a specific projection matrix tailored for the near/far plane constraints of the rendering engine [S1, S2, unofficial].

## How it works
**Inputs:**
- Camera intrinsic parameters: focal lengths $f_x, f_y$, principal points $c_x, c_y$, original image width $W$, height $H$ [S1, unofficial].
- Camera extrinsic parameters: 4x4 row-major matrix $C_{world}$ (camera-to-world) [S1, unofficial].
- Rendering parameters: near plane $n$, far plane $f$, downscale factor $s_f$ (step) [S1, unofficial].
- Optional: Camera position, front, up vectors, zoom [S2, unofficial].

**Outputs:**
- View matrix $V$ (4x4) [S1, S2, unofficial].
- Projection matrix $P$ (4x4) [S1, S2, unofficial].
- Combined Proj-View matrix $PV = P \times V$ [S1, unofficial].
- Cached camera position and FOV values [S1, unofficial].

**Data Structures:**
- `Camera` struct containing:
  - `width`, `height`: int [S1, unofficial].
  - `fx`, `fy`, `cx`, `cy`: float [S1, unofficial].
  - `camToWorld[16]`: float array, 4x4 row-major [S1, unofficial].
  - `cachedViewMat`, `cachedProjViewMat`: cached matrices [S1, unofficial].
  - `cachedCamPos[3]`: float array [S1, unofficial].
  - `cachedFovX`, `cachedFovY`: float [S1, unofficial].
  - `ori_*` members for initial parameters [S2, unofficial].

**Steps:**

1. **Calculate Downscale and New Intrinsics:**
   Let $s_f$ be the downscale factor (step).
   Calculate new dimensions:
   $$ W' = \lfloor W / s_f \rfloor $$
   $$ H' = \lfloor H / s_f \rfloor $$
   Calculate new intrinsics by dividing original intrinsics by $s_f$:
   $$ f_x' = f_x / s_f $$
   $$ f_y' = f_y / s_f $$
   $$ c_x' = c_x / s_f $$
   $$ c_y' = c_y / s_f $$
   [S1, unofficial]

2. **Calculate Field of View (FOV):**
   Compute horizontal and vertical FOV in radians:
   $$ \text{fov}_X = 2.0 \cdot \arctan\left(\frac{W'}{2.0 \cdot f_x'}\right) $$
   $$ \text{fov}_Y = 2.0 \cdot \arctan\left(\frac{H'}{2.0 \cdot f_y'}\right) $$
   [S1, unofficial]

3. **Construct View Matrix ($V$):**
   If the view matrix is not cached or FOV has changed:
   Extract rotation $R$ (3x3) and translation $T$ (3x1) from $C_{world}$.
   $R$ consists of elements $C_{world}[i \cdot 4 + j]$ for $i,j \in \{0,1,2\}$.
   $T$ consists of elements $C_{world}[i \cdot 4 + 3]$ for $i \in \{0,1,2\}$.
   Compute inverse rotation $R^{-1} = R^T$ (transpose).
   Compute inverse translation $T^{-1} = -R^{-1} T$.
   Construct $V$ (4x4):
   $$ V = \begin{bmatrix} R^{-1} & T^{-1} \\ 0 & 1 \end{bmatrix} $$
   [S1, unofficial]
   Alternatively, using standard look-at logic with camera position $pos$, target $(0,0,0)$, and up vector $up$:
   $$ V = \text{lookAt}(pos, (0,0,0), up) $$
   [S2, unofficial]

4. **Construct Projection Matrix ($P$):**
   Define near plane $n = 0.001$ and far plane $f = 1000.0$ [S1, unofficial].
   Calculate intermediate values:
   $$ r_p = n \cdot \tan\left(\frac{\text{fov}_X}{2}\right) $$
   $$ t_p = n \cdot \tan\left(\frac{\text{fov}_Y}{2}\right) $$
   [S1, unofficial]
   Construct $P$ (4x4, row-major indices $0..15$):
   $$ P[0] = \frac{n}{r_p} $$
   $$ P[5] = \frac{n}{t_p} $$
   $$ P[10] = \frac{f + n}{f - n} $$
   $$ P[14] = \frac{-2 \cdot f \cdot n}{f - n} $$
   All other elements are typically 0 or 1 (identity for remaining standard perspective components, though specific implementation details for $P[11]$ and $P[15]$ are not explicitly defined in the source notes beyond the standard perspective form implied by the indices [S1, unofficial]). Note: The source uses $0.001f$ as $n$ directly in the formula for $r_p$ and $t_p$, and $1000.0f$ as $f$ [S1, unofficial].

5. **Compute Combined Matrix and Cache:**
   $$ PV = P \times V $$
   Cache $V$, $PV$, camera position, and FOV values [S1, unofficial].

6. **Additional Setup (Optional):**
   Determine spherical harmonic degree based on step and feature size [S1, unofficial].
   Calculate tile bounds based on $W', H'$ and block sizes [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def setup_camera(camera, step, sh_degree_interval, sh_degree, features_rest_size, block_x, block_y):
    # 1. Downscale
    sf = step
    new_width = camera.width // sf
    new_height = camera.height // sf
    
    fx = camera.fx / sf
    fy = camera.fy / sf
    cx = camera.cx / sf
    cy = camera.cy / sf
    
    # 2. FOV
    fov_x = 2.0 * atan(new_width / (2.0 * fx))
    fov_y = 2.0 * atan(new_height / (2.0 * fy))
    
    # 3. View Matrix
    if not camera.cached_view_mat or fov_changed:
        # Extract R and T from camera.camToWorld (4x4 row-major)
        R = [
            [camera.camToWorld[i*4 + j] for j in range(3)] for i in range(3)
        ]
        T = [camera.camToWorld[i*4 + 3] for i in range(3)]
        
        # Inverse Rotation (Transpose)
        R_inv = [[R[j][i] for j in range(3)] for i in range(3)]
        
        # Inverse Translation: -R_inv * T
        T_inv = [
            -sum(R_inv[i][k] * T[k] for k in range(3)) for i in range(3)
        ]
        
        # Construct View Matrix (4x4)
        V = [
            R_inv[0] + [T_inv[0]],
            R_inv[1] + [T_inv[1]],
            R_inv[2] + [T_inv[2]],
            [0.0, 0.0, 0.0, 1.0]
        ]
        camera.cached_view_mat = V
        
        # 4. Projection Matrix
        n = 0.001
        f = 1000.0
        r_p = n * tan(0.5 * fov_x)
        t_p = n * tan(0.5 * fov_y)
        
        P = [
            [n/r_p, 0, 0, 0],
            [0, n/t_p, 0, 0],
            [0, 0, (f+n)/(f-n), 1],
            [0, 0, (-2*f*n)/(f-n), 0]
        ]
        
        # 5. Combined Matrix
        camera.cached_proj_view_mat = multiply_matrices(P, V)
        
        # Cache FOV and Position
        camera.cached_fov_x = fov_x
        camera.cached_fov_y = fov_y
        camera.cached_cam_pos = T # Or derived from V
        
    # 6. Additional Setup
    degrees_to_use = min(step / sh_degree_interval, sh_degree)
    degree = determine_degree_from_features_rest(features_rest_size)
    tile_bounds = calculate_tile_bounds(new_width, new_height, block_x, block_y)
    
    return V, P, camera.cached_proj_view_mat
```

**Edge Cases and Pitfalls:**
- **Precision:** Use float32 for intrinsics and matrices as specified [S1, unofficial].
- **Boundaries:** Ensure $s_f \ge 1$ to avoid division by zero or invalid dimensions.
- **Degenerate Input:** If $f_x$ or $f_y$ is 0, FOV calculation will fail.
- **Caching:** View matrix must be recomputed if FOV changes [S1, unofficial].

## Subtopics

### View Matrix Construction
Focuses on the transformation from camera space to world space using rotation and translation [S1, S2, unofficial].

### Projection Matrix Construction
Focuses on mapping 3D points to 2D screen coordinates using FOV and near/far planes [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ for matrix construction, assuming fixed-size matrices.
- **Memory Complexity:** $O(1)$ for storing matrices and cached values.
- **Performance:** Efficient for real-time rendering and training due to caching and simple arithmetic [S1, S2, unofficial].
- **Strengths:** Handles multi-resolution training via downscaling; explicit control over near/far planes [S1, unofficial].
- **Limitations:** Specific to the Metal/OpenGL context described; may not generalize to all projection types without modification [S1, S2, unofficial].

## Background
Originated in the context of 3D Gaussian Splatting training and slice-based volume rendering [S1, S2, unofficial]. The method is implemented in projects like Msplat and Slice-Base-Volume-Rendering [S1, S2, unofficial].

## Sources
[S1] Show HN: Msplat – 3D Gaussian Splatting training in ~90s on M4 Max, native Metal, https://github.com/rayanht/msplat, fetched 2026-10-26 (unofficial)
[S2] Conradyen/Slice-Base-Volume-Rendering, https://github.com/Conradyen/Slice-Base-Volume-Rendering, fetched 2026-10-27 (unofficial)
