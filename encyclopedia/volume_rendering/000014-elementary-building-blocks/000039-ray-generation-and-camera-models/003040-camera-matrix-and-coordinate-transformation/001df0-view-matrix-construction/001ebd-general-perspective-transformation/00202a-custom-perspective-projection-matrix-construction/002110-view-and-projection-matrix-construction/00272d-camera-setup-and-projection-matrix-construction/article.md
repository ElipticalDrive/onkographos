# Camera Setup and Projection Matrix Construction

Also called: CamSetup, Camera View and Projection Matrix generation, Camera View and Projection Setup

## Overview
This method initializes camera parameters for volume rendering or 3D Gaussian Splatting training, handling downscaling, view matrix construction, and projection matrix calculation [S2, S3, unofficial]. It sets apart from simpler camera models by explicitly managing the transformation from camera-local coordinates to world coordinates via a cached view matrix and computing a perspective projection matrix that accounts for aspect ratio and field of view, often integrating with spherical harmonic degree calculations for rendering [S1, S2, unofficial].

## How it works
**Inputs:**
- Camera position (world space) [S3, unofficial].
- Camera center (look-at target, typically origin) [S1, S3, unofficial].
- Up vector [S1, S3, unofficial].
- Vertical Field of View (fovy) in radians or degrees [S1, S2, unofficial].
- Image width and height [S1, S2, unofficial].
- Intrinsics: focal lengths $f_x, f_y$ and principal points $c_x, c_y$ [S2, unofficial].
- Camera-to-world transformation matrix (4x4 row-major) [S2, unofficial].
- Downscale factor $sf$ (optional, for multi-resolution) [S2, unofficial].
- Spherical Harmonic degree interval and features size (optional, for SH degree calculation) [S2, unofficial].

**Outputs:**
- View matrix $V$ (4x4) [S1, S2, S3, unofficial].
- Projection matrix $P$ (4x4) [S1, S2, unofficial].
- Combined Projection-View matrix $PV = P \times V$ [S2, unofficial].
- Camera position vector [S1, S2, unofficial].
- Cached FOV values [S2, unofficial].

**Data Structures:**
- `Camera` struct containing:
  - `width`, `height`: int [S2, unofficial].
  - `fx`, `fy`, `cx`, `cy`: float [S2, unofficial].
  - `camToWorld`: float[16] (4x4 row-major) [S2, unofficial].
  - `cachedViewMat`, `cachedProjViewMat`: GPU tensor handles [S2, unofficial].
  - `cachedCamPos`: float[3] [S2, unofficial].
  - `cachedFovX`, `cachedFovY`: float [S2, unofficial].
- `ori_*` members for storing initial parameters [S3, unofficial].

**Steps:**

1.  **Initialize Default State:**
    Set default camera position to $(0, 0, 1)$, center to $(0, 0, 0)$, up vector to $(0, 1, 0)$, fovy to 45 degrees, and aspect ratio to 1.0 [S1, unofficial].

2.  **Handle Downscaling (if applicable):**
    If a downscale factor $sf$ is provided:
    -   Calculate new width $W' = \lfloor W / sf \rfloor$ and height $H' = \lfloor H / sf \rfloor$ [S2, unofficial].
    -   Adjust intrinsics: $f_x' = f_x / sf$, $f_y' = f_y / sf$, $c_x' = c_x / sf$, $c_y' = c_y / sf$ [S2, unofficial].
    -   Calculate horizontal and vertical fields of view using the adjusted intrinsics and dimensions:
        $$ f_{ovX} = 2.0 \cdot \arctan\left(\frac{W'}{2.0 \cdot f_x'}\right) $$
        $$ f_{ovY} = 2.0 \cdot \arctan\left(\frac{H'}{2.0 \cdot f_y'}\right) $$
        [S2, unofficial].
    -   *Note:* If intrinsics are not provided, derive horizontal FOV from vertical FOV and aspect ratio:
        $$ f_{ovX} = 2.0 \cdot \arctan\left(\tan\left(\frac{f_{ovY}}{2}\right) \cdot \frac{W}{H}\right) $$
        [S1, unofficial].

3.  **Construct View Matrix:**
    -   Extract rotation $R$ and translation $T$ from the `camToWorld` matrix (4x4 row-major). Let $M_{camToWorld}$ be the matrix.
        $$ R = \begin{bmatrix} M_{00} & M_{01} & M_{02} \\ M_{10} & M_{11} & M_{12} \\ M_{20} & M_{21} & M_{22} \end{bmatrix}, \quad T = \begin{bmatrix} M_{03} \\ M_{13} \\ M_{23} \end{bmatrix} $$
        [S2, unofficial].
    -   Compute inverse rotation $R^{-1} = R^T$ (transpose) and inverse translation $T^{-1} = -R^T T$ [S2, unofficial].
    -   Construct the view matrix $V$ (4x4):
        $$ V = \begin{bmatrix} R^{-1} & T^{-1} \\ 0 & 1 \end{bmatrix} $$
        [S2, unofficial].
    -   *Alternative:* If using a standard look-at function with position $C$, center $L$, and up vector $U$:
        $$ V = \text{lookAt}(C, L, U) $$
        [S1, S3, unofficial].

4.  **Construct Projection Matrix:**
    -   Define near plane $n = 0.001$ and far plane $f = 1000.0$ (or $10000.0$ in some implementations) [S1, S2, unofficial].
    -   Calculate right and top bounds for the near plane:
        $$ r_p = n \cdot \tan\left(\frac{f_{ovX}}{2}\right) $$
        $$ t_p = n \cdot \tan\left(\frac{f_{ovY}}{2}\right) $$
        [S2, unofficial].
    -   Construct the perspective projection matrix $P$ (column-major or row-major depending on convention; here described by elements):
        $$ P = \begin{bmatrix} \frac{n}{r_p} & 0 & 0 & 0 \\ 0 & \frac{n}{t_p} & 0 & 0 \\ 0 & 0 & \frac{f+n}{f-n} & -1 \\ 0 & 0 & \frac{-2fn}{f-n} & 0 \end{bmatrix} $$
        [S2, unofficial].
        *Note:* The element $P_{10}$ (row 2, col 2 in 0-indexed) is $\frac{f+n}{f-n}$ and $P_{14}$ (row 3, col 2) is $\frac{-2fn}{f-n}$ [S2, unofficial]. Some sources use $10000.0$ for far plane [S1, unofficial].

5.  **Compute Combined Matrix and Cache:**
    -   Calculate $PV = P \times V$ [S2, unofficial].
    -   Cache $V$, $PV$, camera position, and FOV values [S2, unofficial].
    -   Invalidate cache if FOV changes or view matrix is not cached [S2, unofficial].

6.  **Calculate Spherical Harmonic Degree (Optional):**
    -   Determine `degreesToUse` based on step and SH degree interval:
        $$ \text{degreesToUse} = \min\left(\frac{\text{step}}{\text{shDegreeInterval}}, \text{shDegree}\right) $$
        [S2, unofficial].
    -   Determine degree from `featuresRest` size [S2, unofficial].

7.  **Calculate Tile Bounds (Optional):**
    -   Compute tile bounds based on width, height, and block sizes $BLOCK\_X, BLOCK\_Y$ [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def setup_camera_and_projection(camera, downscale_factor=1.0):
    """
    camera: object with attributes:
        width, height: int
        fx, fy, cx, cy: float
        camToWorld: list of 16 floats (row-major 4x4)
        position: list of 3 floats (optional, if not in camToWorld)
        center: list of 3 floats (optional, default [0,0,0])
        up: list of 3 floats (optional, default [0,1,0])
        fovy: float (radians, optional, if intrinsics not used)
    """
    # 1. Handle Downscaling
    sf = downscale_factor
    W = camera.width
    H = camera.height
    
    if sf > 1.0:
        W_new = int(W / sf)
        H_new = int(H / sf)
        fx_new = camera.fx / sf
        fy_new = camera.fy / sf
    else:
        W_new = W
        H_new = H
        fx_new = camera.fx
        fy_new = camera.fy

    # 2. Calculate FOV
    # Try to derive from intrinsics if available
    if camera.fx > 0 and camera.fy > 0:
        fov_x = 2.0 * math.atan(W_new / (2.0 * fx_new))
        fov_y = 2.0 * math.atan(H_new / (2.0 * fy_new))
    else:
        # Use provided fovy and aspect ratio
        aspect = W_new / H_new
        fov_y = camera.fovy
        fov_x = 2.0 * math.atan(math.tan(fov_y / 2.0) * aspect)

    # 3. Construct View Matrix
    # Extract R and T from camToWorld (row-major)
    M = camera.camToWorld
    R = [
        [M[0], M[1], M[2]],
        [M[4], M[5], M[6]],
        [M[8], M[9], M[10]]
    ]
    T = [M[3], M[7], M[11]]

    # Inverse Rotation (Transpose)
    R_inv = [
        [R[0][0], R[1][0], R[2][0]],
        [R[0][1], R[1][1], R[2][1]],
        [R[0][2], R[1][2], R[2][2]]
    ]

    # Inverse Translation: -R^T * T
    T_inv = [
        - (R_inv[0][0]*T[0] + R_inv[0][1]*T[1] + R_inv[0][2]*T[2]),
        - (R_inv[1][0]*T[0] + R_inv[1][1]*T[1] + R_inv[1][2]*T[2]),
        - (R_inv[2][0]*T[0] + R_inv[2][1]*T[1] + R_inv[2][2]*T[2])
    ]

    # Construct View Matrix V (4x4 row-major)
    V = [
        R_inv[0][0], R_inv[0][1], R_inv[0][2], T_inv[0],
        R_inv[1][0], R_inv[1][1], R_inv[1][2], T_inv[1],
        R_inv[2][0], R_inv[2][1], R_inv[2][2], T_inv[2],
        0.0, 0.0, 0.0, 1.0
    ]

    # 4. Construct Projection Matrix
    n = 0.001
    f = 1000.0 # Or 10000.0 depending on source [S1, unofficial]
    
    r_p = n * math.tan(fov_x / 2.0)
    t_p = n * math.tan(fov_y / 2.0)
    
    # Projection Matrix P (row-major)
    # P[0] = n/r_p, P[5] = n/t_p, P[10] = (f+n)/(f-n), P[14] = -2*f*n/(f-n)
    P = [
        n/r_p, 0.0, 0.0, 0.0,
        0.0, n/t_p, 0.0, 0.0,
        0.0, 0.0, (f+n)/(f-n), -1.0,
        0.0, 0.0, -2.0*f*n/(f-n), 0.0
    ]

    # 5. Compute Combined Matrix PV = P * V
    # Matrix multiplication for 4x4 row-major
    PV = [[0.0]*16 for _ in range(4)]
    for i in range(4):
        for j in range(4):
            sum_val = 0.0
            for k in range(4):
                sum_val += P[i*4 + k] * V[k*4 + j]
            PV[i*4 + j] = sum_val

    # 6. Cache Results
    cached_view_mat = V
    cached_proj_view_mat = PV
    cached_cam_pos = [M[3], M[7], M[11]] # World position of camera
    cached_fov_x = fov_x
    cached_fov_y = fov_y

    return {
        'view_mat': V,
        'proj_view_mat': PV,
        'cam_pos': cached_cam_pos,
        'fov_x': fov_x,
        'fov_y': fov_y
    }
```

**Edge Cases and Pitfalls:**
-   **Precision:** Use `float` or `double` consistently. Near plane $0.001$ is small; ensure depth buffer precision is sufficient [S1, S2, unofficial].
-   **Matrix Layout:** Sources specify row-major for `camToWorld` and view/projection matrices [S2, unofficial]. Ensure multiplication order is correct ($P \times V$) [S2, unofficial].
-   **FOV Derivation:** If intrinsics are missing, FOV must be derived from aspect ratio and vertical FOV [S1, unofficial]. If both are missing, defaults are used [S1, unofficial].
-   **Downscaling:** Intrinsics must be scaled down by the same factor as resolution [S2, unofficial].

## Operating characteristics
-   **Time Complexity:** $O(1)$ for matrix construction. $O(N)$ if caching invalidation requires recomputation for $N$ cameras.
-   **Memory Complexity:** $O(1)$ per camera for matrices and cached values.
-   **Performance:** Fast, as it involves basic linear algebra operations. Caching prevents redundant computation [S2, unofficial].
-   **Strengths:** Handles downscaling, integrates with SH degree calculation, caches results for efficiency [S2, unofficial].
-   **Limitations:** Assumes perspective projection. Does not handle distortion correction in the core method (though distortion parameters are stored) [S2, unofficial].
-   **When to Use:** When setting up camera for volume rendering or Gaussian Splatting, especially with multi-resolution training [S1, S2, unofficial].
-   **When Not to Use:** For orthographic projection or non-perspective cameras [S2, unofficial].

## Background
This method is commonly used in 3D Gaussian Splatting implementations, such as those by DBauer15 [S1, unofficial] and rayanht [S2, unofficial], and in general graphics pipelines [S3, unofficial]. It evolved from standard computer graphics camera models, adapting them for differentiable rendering and training scenarios [S1, S2, unofficial].

## Sources
[S1] DBauer15/gscache, https://github.com/DBauer15/gscache, fetched 2026-10-26 (unofficial)
[S2] Show HN: Msplat – 3D Gaussian Splatting training in ~90s on M4 Max, native Metal, https://github.com/rayanht/msplat, fetched 2026-10-26 (unofficial)
[S3] Conradyen/Slice-Base-Volume-Rendering, https://github.com/Conradyen/Slice-Base-Volume-Rendering, fetched 2026-10-27 (unofficial)
