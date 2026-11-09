# View and Projection Matrix Construction

Also called: Camera Projection Setup, Camera Uniform and Matrix Setup, View-Projection Matrix Setup, View/Projection Matrix Construction

## Overview
This method computes the view and projection matrices for camera rendering, specifically handling focal length calculation from field of view (FOV) for use in splat shaders [S3, unofficial]. It differs from fixed-function pipeline setups by explicitly managing tangent-based FOV parameters, coordinate system conversions (e.g., OpenGL to WGPU), and uniform buffer population [S1, S2, S3, unofficial].

## How it works
**Inputs:** viewport width $W$, height $H$, FOV $\theta_{fov}$ (degrees), near plane $n$, far plane $f$, eye position $E$, target $T$, up vector $U$.
**Outputs:** view matrix $V$, projection matrix $P$, view-projection matrix $VP$, inverse view-projection $VP^{-1}$, `CameraUniform` struct.

**Data Structures:**
The `CameraUniform` struct is passed to the GPU [S2, unofficial]. Its layout is defined as `#[repr(C)]` and contains:
- `view_proj`: 4x4 f32 matrix (16 floats)
- `inv_view_proj`: 4x4 f32 matrix (16 floats)
- `position`: 3-component f32 vector (3 floats)
- `_pad`: 1 f32 padding value (1 float)
Total: 34 floats, padded to 36 or 40 bytes depending on alignment [S2, unofficial].

**Steps:**
1. **Compute Aspect Ratio:** $a = W / H$ [S3, unofficial].
2. **Compute FOV Tangents:** Convert FOV to radians $\theta_{rad} = \text{radians}(\theta_{fov})$. Compute horizontal tangent $t_{fov_x} = \tan(\theta_{rad} / 2)$ [S1, unofficial]. Compute vertical tangent $t_{fov_y} = t_{fov_x} \times (H / W)$ [S1, unofficial]. Alternatively, compute focal lengths for shader uniforms: $f_y = H / (2 \times \tan(\theta_{rad} / 2))$, $f_x = f_y$ [S3, unofficial].
3. **Construct View Matrix:** $V = \text{lookAt}(E, T, U)$ [S3, unofficial]. Some implementations compute it as $V = \text{inverse}(\text{translation} \times \text{rotation})$ [S1, unofficial].
4. **Construct Projection Matrix:** $P = \text{perspective}(\theta_{rad}, a, n, f)$ [S3, unofficial]. An alternative formulation uses the precomputed vertical tangent: $P = \text{perspective}(\arctan(t_{fov_y}) \times 2, W / H, n, f)$ [S1, unofficial].
5. **Apply Coordinate System Conversion:** To convert from OpenGL to WGPU, multiply the projection matrix by $C = \begin{bmatrix} 1 & 0 & 0 & 0 \\ 0 & 1 & 0 & 0 \\ 0 & 0 & 0.5 & 0.5 \\ 0 & 0 & 0 & 1 \end{bmatrix}$ [S2, unofficial]. Alternatively, flip the Y and Z axes of the view and projection matrices by multiplying columns 1 and 2 by $-1.0$ [S1, unofficial].
6. **Compute Combined Matrices:** $VP = P \times V$ [S3, unofficial]. Compute $VP^{-1} = \text{inverse}(VP)$ [S2, unofficial].
7. **Populate Camera Uniform:** Set `view_proj` to $VP$, `inv_view_proj` to $VP^{-1}$, and `position` to $E$ [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def compute_view_projection(width, height, fov_deg, near, far, eye, target, up):
    # 1. Aspect ratio
    aspect = width / height

    # 2. FOV tangents
    fov_rad = math.radians(fov_deg)
    tan_fov_x = math.tan(fov_rad / 2.0)
    tan_fov_y = tan_fov_x * (height / width)

    # 3. View matrix
    view = look_at(eye, target, up)

    # 4. Projection matrix
    proj = perspective(fov_rad, aspect, near, far)

    # 5. Coordinate conversion (OpenGL to WGPU)
    ogl_to_wgpu = np.array([[1, 0, 0, 0],
                            [0, 1, 0, 0],
                            [0, 0, 0.5, 0.5],
                            [0, 0, 0, 1]], dtype=np.float32)
    proj = ogl_to_wgpu @ proj

    # 6. Combined matrices
    view_proj = proj @ view
    inv_view_proj = np.linalg.inv(view_proj)

    # 7. Camera uniform
    camera_uniform = {
        'view_proj': view_proj,
        'inv_view_proj': inv_view_proj,
        'position': eye,
        '_pad': 0.0
    }
    return view, proj, view_proj, inv_view_proj, camera_uniform

def look_at(eye, target, up):
    z = normalize(eye - target)
    x = normalize(np.cross(up, z))
    y = np.cross(z, x)
    return np.array([[x[0], y[0], z[0], 0],
                     [x[1], y[1], z[1], 0],
                     [x[2], y[2], z[2], 0],
                     [-np.dot(x, eye), -np.dot(y, eye), -np.dot(z, eye), 1]], dtype=np.float32).T

def perspective(fov_rad, aspect, near, far):
    f = 1.0 / math.tan(fov_rad / 2.0)
    return np.array([[f / aspect, 0, 0, 0],
                     [0, f, 0, 0],
                     [0, 0, -(far + near) / (far - near), -1],
                     [0, 0, -(2 * far * near) / (far - near), 0]], dtype=np.float32)
```

**Parameters and Typical Values:**
- Default FOV: 45.0° [S3, unofficial] or 60° [S2, unofficial].
- Default Near Plane: 0.1 [S2, S3, unofficial].
- Default Far Plane: 100.0 [S3, unofficial] or 2000.0 [S2, unofficial].
- Default Camera Position: $(0, 5, 5)$ [S3, unofficial] or $(16, 90, 90)$ [S2, unofficial].
- Default Target: $(0, 0, 0)$ [S3, unofficial] or $(0, 25, 0)$ [S2, unofficial].
- Default Up Vector: $(0, -1, 0)$ [S3, unofficial] or $(0, 1, 0)$ [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Division by Zero:** Guard against $W=0$ or $H=0$ before computing aspect ratio [G].
- **Near/Far Plane Clipping:** The projection matrix is invalid if $n \ge f$ [G].
- **Precision:** Use consistent `float` precision across CPU and GPU calculations to avoid artifacts [G].

## Subtopics

### [Camera Setup and Projection Matrix Construction](3965-camera-setup-and-projection-matrix-construction.md)
Prepares camera parameters for rendering and training, handling downscaling, view matrix construction, and projection matrix calculation.

## Operating characteristics
- **Time Complexity:** $O(1)$ for matrix construction and inversion [G].
- **Memory Complexity:** $O(1)$ for the uniform buffer (36–40 bytes) [G].
- **Performance:** Computation is extremely fast and typically performed on the CPU once per frame or when camera parameters change [G].
- **Strengths:** Explicit control over FOV and aspect ratio; flexible coordinate system handling [S1, S2, S3, unofficial].
- **Limitations:** Requires manual management of uniform buffers and matrix updates [G].
- **When to use:** When implementing custom renderers, Gaussian Splatting, or when fine-grained control over camera matrices is needed beyond fixed-function pipeline defaults [S1, S2, S3, unofficial].

## Background
This method is fundamental to computer graphics, originating from standard perspective projection and look-at camera models. The specific variations cited here are adapted for modern real-time Gaussian Splatting implementations using Vulkan [S1, unofficial], WGPU [S2, unofficial], and OpenGL [S3, unofficial]. The explicit computation of tangent FOV and focal lengths is particularly relevant for splat-based rendering where shader uniforms require these values for correct projection [S3, unofficial].

## Sources
[S1] Show HN: 3DGS.cpp – performant, cross platform Gaussian Splatting with Vulkan, https://github.com/shg8/3DGS.cpp, fetched 2026-10-26 (unofficial)
[S2] scarlyle0/voxel-compare, https://github.com/scarlyle0/voxel-compare, fetched 2026-10-27 (unofficial)
[S3] liaozhangsheng/gsplat-opengl, https://github.com/liaozhangsheng/gsplat-opengl, fetched 2026-10-28 (unofficial)
