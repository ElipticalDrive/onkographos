# Custom Perspective Projection Matrix Construction

Also called: 3D Projection Matrix Construction, Camera projection matrix setup, Canvas projection setup, Custom Perspective Projection Matrix, Perspective projection, View matrix assembly, View matrix setup

## Overview
This method constructs the projection and view matrices required to transform 3D scene coordinates into clip space for rendering [S3, unofficial]. It differs from standard symmetric frustum constructors by allowing explicit control over non-square focal lengths ($f_x, f_y$) and supporting viewport shift vectors to align the camera with a visibility bounding box [S1, S2, unofficial].

## How it works
**Inputs:**
- Focal lengths $f_x, f_y$ [S2, unofficial].
- Viewport dimensions: width $W$, height $H$ [S2, unofficial].
- Near plane distance $n$ [S2, S3, unofficial].
- Far plane distance $f$ [S2, S3, unofficial].
- Shift vectors `before` and `after` for coordinate alignment [S1, unofficial].
- Camera orientation: yaw, pitch, and zoom/aspect ratio [S1, S3, unofficial].

**Outputs:**
- Projection matrix $P$ [S2, S3, unofficial].
- View matrix $V$ [S3, unofficial].
- Model matrix $M$ [S3, unofficial].
- Combined transformation matrix $M_{final} = P \cdot V \cdot M$ [S3, unofficial].

**Data Structures:**
- $4 \times 4$ matrices for projection, view, and model [S3, unofficial].

**Steps:**
1. **Initialize Model Matrix:** Set $M$ to the $4 \times 4$ identity matrix [S3, unofficial]. Apply uniform scaling with factor $0.7$ if required by the scene setup [S1, unofficial].
2. **Construct View Matrix:** Initialize $V$ based on camera orientation [S3, unofficial]. Apply yaw rotation around the Z-axis if $|\text{yaw}| > 10^{-20}$ [S1, unofficial]. Apply pitch rotation around the X-axis if $|\text{pitch}| > 10^{-20}$ [S1, unofficial]. This threshold prevents numerical instability and skips unnecessary multiplications for near-zero angles [S1, unofficial].
3. **Calculate Projection Matrix $P$:** Compute the aspect ratio $a = W / H$ [S3, unofficial]. Set the matrix elements explicitly as [S2, unofficial]:
   $$P[0][0] = \frac{2 f_x}{W}, \quad P[1][1] = \frac{2 f_y}{H}, \quad P[2][2] = -\frac{f + n}{f - n}, \quad P[2][3] = -1, \quad P[3][2] = -\frac{2 f n}{f - n}$$
   All other elements are zero [S2, unofficial]. Alternatively, standard libraries may be used with `glm::perspective(glm::radians(camera.Zoom), a, n, f)` [S3, unofficial], but the explicit construction is necessary when $f_x \neq f_y$ [S2, unofficial].
4. **Apply Viewport Shifts:** If the `before` shift vector is non-zero, apply a translation to shift coordinates to the origin before projection [S1, unofficial]. If the `after` shift vector is non-zero, apply an inverse translation to shift back to screen coordinates after projection [S1, unofficial].
5. **Combine and Update:** Compute $M_{final} = P \cdot V \cdot M$ [S3, unofficial]. Pass $M_{final}$ to shader uniforms. On viewport resize, retrieve new dimensions, temporarily set the renderer size to $1 \times 1$, update the camera aspect ratio and projection matrix, restore the actual renderer dimensions, and update the real projection matrix and splat mesh uniforms [S2, unofficial].

**Parameters:**
- Default camera specs: $f_x = 1159.5880733038064$, $f_y = 1164.6601287484507$, $n = 0.1$, $f = 500.0$ [S2, unofficial].
- Default resolution: $1280 \times 720$ [S3, unofficial].
- Rotation threshold: $10^{-20}$ [S1, unofficial].

**Pitfalls:**
- Numerical instability occurs if yaw or pitch are extremely small but non-zero; skip rotation if $|\theta| < 10^{-20}$ [S1, unofficial].
- The resize procedure requires temporarily setting the renderer size to $1 \times 1$ to correctly update the aspect ratio and projection matrix before restoring the actual dimensions [S2, unofficial].

## Subtopics

## Operating characteristics

## Background
This method is documented in implementations for Gaussian Splatting web viewers and voxel demos [S2, S3, unofficial]. It provides explicit matrix assembly steps for custom camera models, combining projection, view, and model transformations for shader input [S3, unofficial].

## Sources
[S1] hikaricai/vdrm, https://github.com/hikaricai/vdrm, fetched 2026-10-26 (unofficial)
[S2] akbartus/Gaussian-Splatting-WebViewers, https://github.com/akbartus/Gaussian-Splatting-WebViewers, fetched 2026-10-27 (unofficial)
[S3] Kaefaell/VoxelDemo, https://github.com/Kaefaell/VoxelDemo, fetched 2026-10-27 (unofficial)
