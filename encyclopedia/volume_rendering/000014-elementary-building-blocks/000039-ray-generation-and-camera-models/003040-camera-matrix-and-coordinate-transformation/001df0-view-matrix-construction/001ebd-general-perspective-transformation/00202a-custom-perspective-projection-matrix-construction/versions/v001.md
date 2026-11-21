# Custom Perspective Projection Matrix Construction

Also called: 3D Projection Matrix Construction, Camera projection matrix setup, Canvas projection setup, Custom Perspective Projection Matrix, Perspective projection, View matrix assembly, View matrix setup

## Overview
This method constructs the specific projection and view matrices required to transform 3D scene coordinates into clip space for rendering, specifically tailored for Gaussian Splatting pipelines [S3, unofficial]. It differs from standard textbook perspective matrices by allowing explicit control over focal lengths ($f_x, f_y$) and handling viewport shifts to align the camera center with the visibility bounding box (Vbb) [S2, S3, unofficial]. It builds on [General Perspective Transformation](1845-general-perspective-transformation.md) by providing the concrete matrix assembly steps for a custom camera model.

## How it works
**Inputs:**
- Camera focal lengths $f_x, f_y$ (in pixels).
- Viewport dimensions: width $W$, height $H$.
- Near plane distance $n$ (default 0.1).
- Far plane distance $f$ (default 500.0 or 1000.0).
- Optional shift vectors `before` and `after` for coordinate system alignment [S1, unofficial].
- Camera orientation parameters: yaw, pitch [S1, unofficial].

**Outputs:**
- A $4 \times 4$ projection matrix $P$.
- A $4 \times 4$ view matrix $V$.
- A combined model-view-projection matrix $MVP = P \cdot V \cdot M$ [S3, unofficial].

**Data Structures:**
- `ProjectionMatrix`: A $4 \times 4$ column-major or row-major matrix (implementation dependent, but typically column-major in GLM).
- `CameraSpecs`: Struct containing $f_x, f_y, n, f$.

**Steps:**

1. **Initialize Matrices:**
   Set the model matrix $M$ to the identity matrix $I_4$ [S3, unofficial].
   Set the view matrix $V$ based on camera orientation. If using a standard camera class, call `GetViewMatrix()` [S3, unofficial]. If constructing manually:
   - Apply yaw rotation $R_z(\text{yaw})$ around the Z-axis if $|\text{yaw}| > 10^{-20}$ [S1, unofficial].
   - Apply pitch rotation $R_x(\text{pitch})$ around the X-axis if $|\text{pitch}| > 10^{-20}$ [S1, unofficial].
   - Combine rotations to form the view rotation component.

2. **Calculate Projection Matrix $P$:**
   Compute the aspect ratio $a = W / H$ [S3, unofficial].
   Compute the field of view angle $\theta$ if using an angle-based constructor, or use focal lengths directly.
   
   The explicit elements of the projection matrix $P$ are defined as [S2, unofficial]:
   $$
   P = \begin{bmatrix}
   \frac{2 f_x}{W} & 0 & 0 & 0 \\
   0 & \frac{2 f_y}{H} & 0 & 0 \\
   0 & 0 & -\frac{f + n}{f - n} & -1 \\
   0 & 0 & -\frac{2 f n}{f - n} & 0
   \end{bmatrix}
   $$
   Note: The source [S2, unofficial] specifies $P[2][3] = -1$ and $P[3][2] = -\frac{2 f n}{f - n}$, with all other elements zero except the diagonal terms $P[0][0]$ and $P[1][1]$. This corresponds to a right-handed coordinate system where the camera looks down the negative Z-axis, mapping depth to $[-1, 1]$ in clip space (with a sign flip in the W-component handling).

   Alternatively, if using an API like GLM, one might call `glm::perspective(glm::radians(camera.Zoom), a, n, f)` [S3, unofficial], but the explicit formula above is required for custom focal lengths $f_x, f_y$ that differ from the standard symmetric frustum assumption.

3. **Apply Viewport Shifts (Optional):**
   If a shift vector `before` is provided (non-zero), apply a translation to shift the coordinate system to the origin before projection [S1, unofficial].
   If an `after` vector is provided (non-zero), apply an inverse translation to shift back to screen coordinates after projection [S1, unofficial].

4. **Combine Matrices:**
   Compute the final transformation matrix:
   $$
   M_{final} = P \cdot V \cdot M
   $$
   where $M$ is the model matrix (often identity for global scene coordinates) [S3, unofficial].

5. **Update Uniforms:**
   Pass $M_{final}$ to the shader uniforms. On viewport resize, update $W, H$, recalculate $P$, and update the splat mesh uniforms [S2, unofficial].

**Parameters:**
- Default camera specs: $f_x = 1159.5880733038064$, $f_y = 1164.6601287484507$, $n = 0.1$, $f = 500.0$ [S2, unofficial].
- Default resolution: $1280 \times 720$ [S3, unofficial].
- Rotation threshold: $10^{-20}$ to skip near-zero rotations [S1, unofficial].

**Pitfalls:**
- Numerical instability occurs if yaw or pitch are extremely small but non-zero; skip rotation if $|\theta| < 10^{-20}$ [S1, unofficial].
- Ensure consistent coordinate system handedness (left vs. right) between the projection matrix construction and the view matrix. The formula in [S2, unofficial] assumes a specific handedness (likely right-handed with negative Z look direction).
- When resizing, temporarily setting renderer size to $1 \times 1$ to update aspect ratio can cause visual glitches if not handled carefully [S2, unofficial].

## Subtopics

## Operating characteristics
- **Time Complexity:** $O(1)$ for matrix construction. $O(N)$ for updating uniforms if $N$ is the number of splats/objects, but the matrix math itself is constant time.
- **Memory Complexity:** $O(1)$ for the matrices themselves ($4 \times 4$ floats).
- **Strengths:** Allows precise control over focal length and aspect ratio, essential for Gaussian Splatting where $f_x \neq f_y$ is common. Handles viewport shifts for bounding box alignment.
- **Limitations:** Requires manual handling of coordinate system conventions. Not a general-purpose "black box" like `glm::perspective` for standard symmetric frustums.
- **When to use:** Gaussian Splatting rendering, custom camera models with non-square pixels or specific focal lengths, or when aligning the camera center with a specific Vbb.

## Background
This method is derived from implementations in Gaussian Splatting web viewers and voxel demos [S2, S3, unofficial]. It adapts standard perspective projection formulas to support the specific camera parameters (focal lengths) used in 3D Gaussian Splatting, which often differ from the standard OpenGL/DirectX default perspective matrix assumptions. The explicit matrix construction ensures compatibility with shaders expecting specific clip-space transformations.

## Sources
[S1] hikaricai/vdrm, https://github.com/hikaricai/vdrm, fetched 2026-10-26 (unofficial)
[S2] akbartus/Gaussian-Splatting-WebViewers, https://github.com/akbartus/Gaussian-Splatting-WebViewers, fetched 2026-10-27 (unofficial)
[S3] Kaefaell/VoxelDemo, https://github.com/Kaefaell/VoxelDemo, fetched 2026-10-27 (unofficial)
