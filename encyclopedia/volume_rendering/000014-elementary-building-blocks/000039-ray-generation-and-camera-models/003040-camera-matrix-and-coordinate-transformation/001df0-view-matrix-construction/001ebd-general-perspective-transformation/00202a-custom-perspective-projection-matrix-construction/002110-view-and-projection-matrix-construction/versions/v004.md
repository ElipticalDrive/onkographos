# View and Projection Matrix Construction

Also called: Camera Projection Setup, Camera Uniform and Matrix Setup, View-Projection Matrix Setup, View/Projection Matrix Construction

## Overview
This method constructs the view and projection matrices required to transform 3D scene coordinates into clip space for rendering, specifically tailored for Gaussian Splatting and similar volume rendering techniques. It solves the problem of establishing the correct camera coordinate system and perspective distortion for ray generation. It differs from standard fixed-function pipeline setups by explicitly managing the computation of tangent-based FOV parameters for shader uniforms and handling coordinate system conversions (e.g., OpenGL to WGPU) via explicit matrix multiplication or axis flipping [S1, S2, S3, unofficial].

## How it works
The method computes the view matrix from camera position, target, and up vector, and the projection matrix from field of view (FOV), aspect ratio, and clipping planes. It often outputs a combined view-projection matrix and its inverse for use in shaders.

**Inputs:**
- Camera position (eye) $E$
- Camera target $T$
- Camera up vector $U$
- Viewport width $W$ and height $H$
- Vertical Field of View angle $\theta_{fov}$ (in degrees or radians)
- Near plane distance $n$
- Far plane distance $f$
- Coordinate system conversion requirement (e.g., OpenGL to WGPU) [S2, unofficial]

**Outputs:**
- View Matrix $V$
- Projection Matrix $P$
- View-Projection Matrix $VP = P \times V$
- Inverse View-Projection Matrix $(VP)^{-1}$
- Camera Uniform data structure [S2, unofficial]

**Data Structures:**
The `CameraUniform` struct is passed to the GPU. Its layout is defined as follows [S2, unofficial]:
- `view_proj`: 4x4 floating-point matrix (16 floats)
- `inv_view_proj`: 4x4 floating-point matrix (16 floats)
- `position`: 3-component floating-point vector (3 floats)
- `_pad`: 1 floating-point padding value (1 float)
Total: 36 floats. Depending on alignment requirements (e.g., `#[repr(C)]` in Rust), the total size may be padded to 36 or 40 bytes.

**Steps:**

1.  **Compute Aspect Ratio:**
    Calculate the aspect ratio $a$ from the viewport dimensions [S3, unofficial]:
    $$ a = \frac{W}{H} $$

2.  **Compute Tangent FOV:**
    Calculate the tangent of half the vertical FOV. If the input FOV is $\theta_{fov}$ (in degrees), convert to radians first [S1, S3, unofficial]:
    $$ \theta_{rad} = \text{radians}(\theta_{fov}) $$
    $$ t_{fov_y} = \tan\left(\frac{\theta_{rad}}{2}\right) $$
    Calculate the horizontal tangent $t_{fov_x}$ using the aspect ratio [S1, unofficial]:
    $$ t_{fov_x} = t_{fov_y} \times a $$
    *Note: Some implementations compute focal lengths $f_x, f_y$ directly for shader use [S3, unofficial]:*
    $$ f_y = \frac{H}{2 \times t_{fov_y}} $$
    $$ f_x = f_y $$

3.  **Construct View Matrix:**
    Compute the view matrix $V$ using the LookAt function, which constructs a rotation-translation matrix from eye, target, and up vectors [S3, unofficial]:
    $$ V = \text{LookAt}(E, T, U) $$
    Alternatively, if using inverse transformation logic [S1, unofficial]:
    $$ V = \text{Inverse}(\text{Translation}(T) \times \text{Rotation}(U, \text{axis})) $$
    *Standard LookAt implementation [G]:*
    Let $z_{axis} = \text{normalize}(E - T)$.
    Let $x_{axis} = \text{normalize}(\text{cross}(U, z_{axis}))$.
    Let $y_{axis} = \text{cross}(z_{axis}, x_{axis})$.
    $$ V = \begin{bmatrix} x_{axis}.x & x_{axis}.y & x_{axis}.z & -\text{dot}(x_{axis}, E) \\ y_{axis}.x & y_{axis}.y & y_{axis}.z & -\text{dot}(y_{axis}, E) \\ z_{axis}.x & z_{axis}.y & z_{axis}.z & -\text{dot}(z_{axis}, E) \\ 0 & 0 & 0 & 1 \end{bmatrix} $$

4.  **Construct Projection Matrix:**
    Compute the perspective projection matrix $P$. Using the standard `perspective` function with vertical FOV [S3, unofficial]:
    $$ P = \text{Perspective}(\theta_{rad}, a, n, f) $$
    *Standard Perspective implementation [G]:*
    $$ P = \begin{bmatrix} \frac{1}{t_{fov_x}} & 0 & 0 & 0 \\ 0 & \frac{1}{t_{fov_y}} & 0 & 0 \\ 0 & 0 & \frac{-(f+n)}{f-n} & \frac{-2fn}{f-n} \\ 0 & 0 & -1 & 0 \end{bmatrix} $$
    *Note: Some sources use `atan(t_fovy) * 2` as the FOV argument if `t_fovy` was precomputed [S1, unofficial].*

5.  **Apply Coordinate System Conversion (Optional):**
    If converting between coordinate systems (e.g., OpenGL to WGPU), apply a conversion matrix $C$ [S2, unofficial].
    Method A: Multiply the projection matrix by the conversion matrix.
    $$ P' = C \times P $$
    Where $C$ for OpenGL to WGPU is [S2, unofficial]:
    $$ C = \begin{bmatrix} 1 & 0 & 0 & 0 \\ 0 & 1 & 0 & 0 \\ 0 & 0 & 0.5 & 0.5 \\ 0 & 0 & 0 & 1 \end{bmatrix} $$
    Method B: Flip Y and Z axes of the view and projection matrices by multiplying columns 1 and 2 by -1.0 [S1, unofficial].

6.  **Compute Combined Matrices:**
    Calculate the view-projection matrix [S3, unofficial]:
    $$ VP = P \times V $$
    Calculate the inverse view-projection matrix [S2, unofficial]:
    $$ VP^{-1} = \text{Inverse}(VP) $$

7.  **Populate Camera Uniform:**
    Fill the `CameraUniform` struct with $VP$, $VP^{-1}$, and the eye position $E$ [S2, unofficial].

**Parameters and Typical Values:**
- Default FOV: 45.0 degrees [S3, unofficial] or 60 degrees [S2, unofficial].
- Default Near Plane: 0.1 [S2, S3, unofficial].
- Default Far Plane: 100.0 [S3, unofficial] or 2000.0 [S2, unofficial].
- Default Camera Position: $(0, 5, 5)$ [S3, unofficial] or $(16, 90, 90)$ [S2, unofficial].
- Default Target: $(0, 0, 0)$ [S3, unofficial] or $(0, 25, 0)$ [S2, unofficial].
- Default Up Vector: $(0, -1, 0)$ [S3, unofficial] or $(0, 1, 0)$ [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Division by Zero:** If $W$ or $H$ is 0, the aspect ratio is undefined. Guard against this.
- **Near/Far Plane Clipping:** If $n \ge f$, the projection matrix is invalid.
- **Coordinate System Mismatch:** Failing to apply the OpenGL-to-WGPU conversion or axis flipping results in inverted or distorted depth buffers [S1, S2, unofficial].
- **Precision:** Using `float` for matrix elements is standard, but ensure consistent precision across CPU and GPU calculations.

## Subtopics

### [Camera Setup and Projection Matrix Construction](3965-camera-setup-and-projection-matrix-construction.md)
Prepares camera parameters for rendering and training, handling downscaling, view matrix construction, and projection matrix calculation.

## Operating characteristics
- **Time Complexity:** $O(1)$ for matrix construction. Matrix inversion is $O(1)$ for 4x4 matrices (constant number of operations).
- **Memory Complexity:** $O(1)$ for the uniform buffer (36-40 floats).
- **Performance:** The computation is extremely fast and typically performed on the CPU once per frame or when camera parameters change. It does not bottleneck rendering.
- **Strengths:** Explicit control over FOV and aspect ratio; flexible coordinate system handling.
- **Limitations:** Requires manual management of uniform buffers and matrix updates.
- **When to use:** When implementing custom renderers, Gaussian Splatting, or when fine-grained control over camera matrices is needed beyond fixed-function pipeline defaults.

## Background
This method is fundamental to computer graphics, originating from the standard perspective projection and look-at camera models described in early graphics literature. The specific variations cited here are adapted for modern real-time Gaussian Splatting implementations using Vulkan [S1, unofficial], WGPU [S2, unofficial], and OpenGL [S3, unofficial]. The explicit computation of tangent FOV and focal lengths is particularly relevant for splat-based rendering where shader uniforms require these values for correct projection.

## Sources
[S1] Show HN: 3DGS.cpp – performant, cross platform Gaussian Splatting with Vulkan, https://github.com/shg8/3DGS.cpp, fetched 2026-10-26 (unofficial)
[S2] scarlyle0/voxel-compare, https://github.com/scarlyle0/voxel-compare, fetched 2026-10-27 (unofficial)
[S3] liaozhangsheng/gsplat-opengl, https://github.com/liaozhangsheng/gsplat-opengl, fetched 2026-10-28 (unofficial)
