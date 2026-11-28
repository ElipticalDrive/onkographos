# Custom Perspective Projection Matrix Construction

Also called: 3D Projection Matrix Construction, Camera projection matrix setup, Canvas projection setup, Custom Perspective Projection Matrix, View matrix assembly, View matrix setup

## Overview
This method constructs the perspective projection matrix and view matrix for 3D rendering, combining them with the model matrix to form the final transformation matrix passed to the shader [S3, unofficial]. It solves the problem of defining the camera's field of view, aspect ratio, and depth clipping planes for specific rendering pipelines [S2, unofficial][S3, unofficial]. It differs from generic graphics pipelines by explicitly managing intrinsic scaling (focal lengths) for multi-resolution training and constructing a specific projection matrix tailored for the near/far plane constraints of the rendering engine [S1, unofficial][S2, unofficial]. It builds on [General Perspective Transformation](1845-general-perspective-transformation.md) [G].

## How it works
**Inputs:**
- Camera parameters: Field of view (FOV) in radians, aspect ratio ($W/H$), near plane distance ($n$), far plane distance ($f$) [S3, unofficial].
- Optional: Focal lengths ($f_x, f_y$) if constructing from intrinsic parameters [S2, unofficial].
- Screen dimensions: Width ($W$) and Height ($H$) in pixels [S2, unofficial].
- Camera orientation: Yaw and pitch angles [S1, unofficial].
- Shift vectors: `before` and `after` vectors for coordinate system adjustments [S1, unofficial].

**Outputs:**
- Projection matrix $P$ (4x4) [S2, unofficial].
- View matrix $V$ (4x4) [S3, unofficial].
- Combined model-view-projection matrix $MVP = P \cdot V \cdot Model$ [S3, unofficial].

**Data Structures:**
- `Matrix4`: A 4x4 matrix [G].
- `Vector3`: 3D coordinates for shifts and rotations [G].

**Steps:**

1. **Initialize Matrices:**
   - Set the Model matrix $Model$ to the identity matrix $I$ [S3, unofficial].
   - Initialize the Projection matrix $P$ with zeros [S2, unofficial].

2. **Calculate Projection Matrix Elements:**
   - If focal lengths $f_x, f_y$ are provided, use them directly [S2, unofficial].
   - Otherwise, derive them from FOV and aspect ratio:
     $$f_x = \frac{W}{2 \tan(\text{FOV}_y / 2)}$$
     $$f_y = \frac{H}{2 \tan(\text{FOV}_y / 2)}$$
     [G]
   - Set the non-zero elements of $P$ as follows [S2, unofficial]:
     $$P_{0,0} = \frac{2 f_x}{W}$$
     $$P_{1,1} = \frac{2 f_y}{H}$$
     $$P_{2,2} = -\frac{f + n}{f - n}$$
     $$P_{2,3} = -1$$
     $$P_{3,2} = -\frac{2 f n}{f - n}$$
     All other elements are 0.
   - This formulation maps the view space $z$ to clip space $z'$ such that $z' \in [-1, 1]$ [G].
   - The negative signs on $P_{2,2}$ and $P_{2,3}$ assume a right-handed coordinate system where the camera looks down the negative Z-axis [G].

3. **Apply Coordinate Shifts (Optional):**
   - If a `before` shift vector is non-zero, apply a translation to move the origin before projection [S1, unofficial].
   - If an `after` shift vector is non-zero, apply an inverse translation after projection to map back to screen coordinates [S1, unofficial].

4. **Construct View Matrix:**
   - Get the view matrix $V$ from the camera object [S3, unofficial].
   - Alternatively, construct $V$ manually using yaw and pitch rotations [S1, unofficial]:
     - If $|\text{yaw}| > 10^{-20}$, apply rotation around the Z-axis [S1, unofficial].
     - If $|\text{pitch}| > 10^{-20}$, apply rotation around the X-axis [S1, unofficial].
     - Rotations are skipped if angles are near zero ($10^{-20}$) to prevent numerical instability and unnecessary matrix multiplications [S1, unofficial].

5. **Apply Model Transformations (Optional):**
   - Apply uniform scaling (e.g., factor 0.7) to the Model matrix if required by the scene setup [S1, unofficial].

6. **Combine Matrices:**
   - Compute the final transformation matrix:
     $$MVP = P \cdot V \cdot Model$$
     [S3, unofficial]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def construct_projection_matrix(width, height, fov_radians, near, far, fx=None, fy=None):
    if fx is None or fy is None:
        fy = (height / 2.0) / math.tan(fov_radians / 2.0)
        fx = fy
    P = [[0.0] * 4 for _ in range(4)]
    P[0][0] = (2.0 * fx) / width
    P[1][1] = (2.0 * fy) / height
    P[2][2] = -(far + near) / (far - near)
    P[2][3] = -1.0
    P[3][2] = (-2.0 * far * near) / (far - near)
    return P

def construct_view_matrix(yaw, pitch, camera_position):
    V = [[1.0 if i == j else 0.0 for j in range(4)] for i in range(4)]
    if abs(yaw) > 1e-20:
        c, s = math.cos(yaw), math.sin(yaw)
        R = [[c, -s, 0, 0], [s, c, 0, 0], [0, 0, 1, 0], [0, 0, 0, 1]]
        V = multiply_matrices(R, V)
    if abs(pitch) > 1e-20:
        c, s = math.cos(pitch), math.sin(pitch)
        R = [[1, 0, 0, 0], [0, c, -s, 0], [0, s, c, 0], [0, 0, 0, 1]]
        V = multiply_matrices(R, V)
    T = [[1, 0, 0, -camera_position[0]], [0, 1, 0, -camera_position[1]], [0, 0, 1, -camera_position[2]], [0, 0, 0, 1]]
    V = multiply_matrices(T, V)
    return V

def multiply_matrices(A, B):
    C = [[0.0] * 4 for _ in range(4)]
    for i in range(4):
        for j in range(4):
            for k in range(4):
                C[i][j] += A[i][k] * B[k][j]
    return C

def combine_matrices(projection, view, model):
    return multiply_matrices(projection, multiply_matrices(view, model))
```

**Parameters:**
- `DEFAULT_CAMERA_SPECS`: Typical default values include $f_x = 1159.5880733038064$, $f_y = 1164.6601287484507$, $near = 0.1$, $far = 500$ [S2, unofficial].
- Aspect Ratio: Typically $SCR\_WIDTH / SCR\_HEIGHT$, e.g., $1280/720$ [S3, unofficial].

**Edge Cases and Pitfalls:**
- **Numerical Instability:** Rotations should be skipped if yaw or pitch are near zero ($< 10^{-20}$) to avoid numerical noise [S1, unofficial].
- **Far Plane Clipping:** Standard finite far planes can cause precision loss at large distances [G].
- **Depth Buffer Convention:** The formula provided maps $z$ to $[-1, 1]$ [G]. If the renderer expects $[0, 1]$, the matrix elements $P_{2,2}$ and $P_{3,2}$ must be adjusted [G].

## Subtopics

### [View and Projection Matrix Construction](2432-view-and-projection-matrix-construction.md)
This method constructs the view and projection matrices required for volume rendering, specifically tailored for Gaussian Splatting pipelines where splat shaders may require explicit focal length calculations derived from the field of view (FOV). It differs from generic graphics pipelines by explicitly managing intrinsic scaling for multi-resolution training and constructing a specific projection matrix tailored for the near/far plane constraints of the rendering engine. It builds on [Custom Perspective Projection Matrix Construction](2175-custom-perspective-projection-matrix-construction.md).

### [Reverse Z Perspective Projection with Infinite Far Plane](7544-reverse-z-perspective-projection-with-infinite-far-plane.md)
This method constructs a perspective projection matrix that maps view space coordinates to clip space while utilizing a reversed depth buffer (Reverse Z) and an effectively infinite far plane. It solves the problem of depth precision loss at large distances inherent in standard finite-far-plane projections, thereby avoiding far-plane clipping artifacts and maintaining high precision near the camera. It differs from standard perspective projections by inverting the depth range and removing the far-plane dependency from the matrix structure. It builds on [Custom Perspective Projection Matrix Construction](2175-custom-perspective-projection-matrix-construction.md) [G].

### [MiniGS Full Projection Matrix](13954-minigs-full-projection-matrix.md)
The MiniGS Full Projection Matrix constructs a combined view-projection matrix matching the MiniGS reference kernel convention using explicit frustum plane calculations and a specific z-near/z-far mapping.

### Object Space Projection
Object Space Projection is a technique where a second-order tensor $T$ is projected onto object space (surface space) using a symmetric projection matrix $P$ [S6, unofficial]. It is used to transform tensor fields from world space to object space for visualization, such as in Line Integral Convolution (LIC) methods [S6, unofficial]. The projection is defined as $T' = P \cdot T \cdot P^T$, where $P$ is constructed from the surface normal $\mathbf{n} = (n_x, n_y, n_z)$ as:
$$P = \begin{pmatrix} 1 - n_x^2 & -n_y n_x & -n_z n_x \\ -n_x n_y & 1 - n_y^2 & -n_z n_y \\ -n_x n_z & -n_y n_z & 1 - n_z^2 \end{pmatrix}$$
[S6, unofficial]. This method is typically applied during the rendering pass where depth and surface normal information are available [S6, unofficial].

## Operating characteristics
- **Time Complexity:** Matrix construction is $O(1)$ as it involves a fixed number of arithmetic operations [G]. Combining matrices is $O(1)$ for 4x4 matrices [G].
- **Memory Complexity:** $O(1)$ for storing the matrices [G].
- **Performance:** The method is computationally inexpensive [G]. The bottleneck is usually the rendering of the splats or volume slices, not the matrix construction [G].
- **Strengths:** Programmatic control over focal lengths allows adjustment for multi-resolution training [S2, unofficial].
- **Limitations:** Standard perspective projection suffers from depth precision issues at large distances [G]. The method does not inherently support infinite far planes without modification [G].
- **When to Use:** Use when explicit control over camera intrinsics is required, such as in Gaussian Splatting, or when integrating with OpenGL/DirectX pipelines that expect standard projection matrices [G].
- **When Not to Use:** Use [Reverse Z Perspective Projection with Infinite Far Plane](7544-reverse-z-perspective-projection-with-infinite-far-plane.md) if depth precision at large distances is critical [G]. Use [MiniGS Full Projection Matrix](13954-minigs-full-projection-matrix.md) if targeting the MiniGS kernel specifically [G].

## Background
The specific formulation used here, with explicit focal lengths and the particular arrangement of matrix elements, is commonly found in modern real-time rendering engines and Gaussian Splatting implementations [S2, unofficial][S3, unofficial]. The method has been adapted for volume rendering and 3D Gaussian Splatting to ensure that the rendered splats accurately reflect the camera's intrinsic parameters, which is essential for high-fidelity novel view synthesis [S1, unofficial][S2, unofficial].

## Sources
[S1] hikaricai/vdrm, https://github.com/hikaricai/vdrm, fetched 2026-10-26 (unofficial)
[S2] akbartus/Gaussian-Splatting-WebViewers, https://github.com/akbartus/Gaussian-Splatting-WebViewers, fetched 2026-10-27 (unofficial)
[S3] Kaefaell/VoxelDemo, https://github.com/Kaefaell/VoxelDemo, fetched 2026-10-27 (unofficial)
[S4] Image-Space Tensor Field Visualization Using a LIC-like Method, https://doi.org/10.1007/978-3-642-21608-4_11, fetched 2026-10-27
