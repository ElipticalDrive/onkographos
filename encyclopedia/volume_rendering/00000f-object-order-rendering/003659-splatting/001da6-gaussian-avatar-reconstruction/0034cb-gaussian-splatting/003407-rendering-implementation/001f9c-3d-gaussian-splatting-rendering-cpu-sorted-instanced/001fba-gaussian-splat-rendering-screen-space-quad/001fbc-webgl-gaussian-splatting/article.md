# WebGL Gaussian Splatting

Also called: Gaussian Splat Rendering Pipeline, Gaussian Splatting Web Rendering (Three.js), Gaussian Splatting WebGL, Three.js Gaussian Splatting, Web-based 3D Gaussian Splatting Rendering, WebGL 1.0 Gaussian Splatting

## Overview
WebGL Gaussian Splatting is a real-time rendering technique for 3D Gaussian Splatting (3DGS) scenes in web browsers, typically implemented using WebGL 1.0 and libraries like Three.js [S1, S2, unofficial]. It solves the problem of displaying large-scale, photorealistic radiance field reconstructions on standard client hardware without requiring neural network inference or heavy server-side processing [S1, unofficial]. Unlike parent method [Gaussian Splat Rendering (Screen-Space Quad)](2075-gaussian-splat-rendering-screen-space-quad.md), which relies on a pre-sorted order buffer and view-space projection data, this method performs depth sorting in a JavaScript Web Worker and projects 3D Gaussians into screen-space quads using custom vertex and fragment shaders [S2, unofficial]. It sets itself apart from sibling [Gaussian Splat Fragment Shader](3258-gaussian-splat-fragment-shader.md) by providing the complete pipeline, including data parsing, instance matrix construction, and CPU-side sorting, rather than just the shading logic [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A binary `.splat` file containing Gaussian parameters (position, scale, quaternion, color, opacity) [S2, unofficial].
*   **Output:** A rendered image of the Gaussian splats composited via alpha blending [S1, S2, unofficial].

**Data Structures:**
The implementation uses `InstancedBufferGeometry` with the following attributes [S2, unofficial]:
1.  `position`: A `Float32Array` of size 18 (6 vertices * 3 components) defining a hardcoded quad with vertices at $(2,2,0), (-2,2,0), (-2,-2,0), (2,-2,0)$ and their duplicates for triangle indexing [S2, unofficial].
2.  `splatColor`: A `Float32Array` of size $4 \times N$, where $N$ is the vertex count. Stores RGBA values normalized to $[0,1]$ [S2, unofficial].
3.  `splatCenterCovariance`: A `Float32Array` of size $9 \times N$. Stores the upper-triangular 3x3 covariance matrix components. Layout per splat: `[center_x, center_y, center_z, M11, M12, M13, M22, M23, M33]` [S2, unofficial].

**Binary File Layout (.splat):**
Each vertex occupies 32 bytes [S2, unofficial]:
*   Offset 0-11: `Float32` center (3 floats) [S2, unofficial].
*   Offset 12-23: `Float32` scale (3 floats) [S2, unofficial].
*   Offset 24-27: `Uint8` color (R, G, B, A) [S2, unofficial].
*   Offset 28-31: `Uint8` quaternion (x, y, z, w) [S2, unofficial]. The quaternion is stored as integers 0-255 and must be normalized by subtracting 128 and dividing by 128.0, with the z-component negated [S2, unofficial].

**Numbered Steps:**

1.  **Data Parsing:**
    *   Read the binary file into a `Uint8Array` [S2, unofficial].
    *   Calculate $N = \lfloor \text{byteLength} / 32 \rfloor$ [S2, unofficial].
    *   For each vertex $i$:
        *   Extract center from `f_buffer[8*i + 0..2]` [S2, unofficial].
        *   Extract scale from `f_buffer[8*i + 3..5]` [S2, unofficial].
        *   Extract quaternion bytes from `u_buffer[32*i + 28..31]` [S2, unofficial]. Normalize: $q = (\text{byte} - 128) / 128.0$. Negate $q_z$ [S2, unofficial].
        *   Extract color bytes from `u_buffer[32*i + 24..27]` [S2, unofficial]. Normalize to $[0,1]$ [S2, unofficial].
        *   Construct a 4x4 instance matrix $M_i$:
            *   Create rotation matrix $R$ from quaternion $q$ [S2, unofficial].
            *   Create scale matrix $S$ from scale vector [S2, unofficial].
            *   $M_i = R \cdot S \cdot R^T$ (transposes and premultiplication form the transformation matrix) [S2, unofficial].
            *   Set position components of $M_i$ to center [S2, unofficial].
            *   Set color/alpha components of $M_i$ (elements 3, 7, 11, 15) to normalized RGBA [S2, unofficial].

2.  **Depth Sorting (Web Worker):**
    *   Pass the array of $N$ instance matrices and the camera view vector (from `camera.matrixWorld` elements 2, 6, 10) to a sorting function [S2, unofficial].
    *   Calculate depth for each splat $i$: $d_i = \text{view}_x \cdot M_i[12] + \text{view}_y \cdot M_i[13] + \text{view}_z \cdot M_i[14]$ [S2, unofficial].
    *   The notes describe a 16-bit single-pass counting sort, but the provided implementation uses JavaScript's native `.sort()` method, yielding $O(N \log N)$ complexity [S2, unofficial].
    *   Reorder matrices: $M_{sorted}[j] = M_{original}[depthIndex[j]]$ [S2, unofficial].
    *   Transfer sorted matrices back to the main thread via `postMessage` with transferable objects to avoid blocking [S2, unofficial].

3.  **Vertex Shader Execution (Per Instance):**
    *   **Projection:**
        *   Extract splat center $C$ and instance matrix $M$ [S2, unofficial].
        *   Compute camera-space position: $C_{cam} = V \cdot C$, where $V$ is the view matrix [S2, unofficial].
        *   Compute focal length: $f = \frac{h}{2 \tan(\frac{fov \cdot \pi}{360})}$, where $h$ is viewport height and $fov$ is vertical FOV in degrees [S2, unofficial].
        *   Compute Jacobian $J$ of the projection:
            $$
            J = \begin{bmatrix}
            \frac{f_x}{z} & 0 & -\frac{f_x x}{z^2} \\
            0 & \frac{f_y}{z} & -\frac{f_y y}{z^2}
            \end{bmatrix}
            $$
            where $(x,y,z) = C_{cam}$ and $f_x, f_y$ are focal lengths [S2, unofficial].
        *   Compute transformation matrix $T = W \cdot J$, where $W$ is the rotation part of the view matrix (transpose of view matrix rotation) [S2, unofficial].
        *   Compute 2D covariance: $\mathbf{cov}_{2D} = T^T \mathbf{M}_{instance} T$ [S2, unofficial].
        *   Regularize: Add 0.3 to $\mathbf{cov}_{2D}[0][0]$ and $\mathbf{cov}_{2D}[1][1]$ to prevent singularities [S2, unofficial].
    *   **Eigen Decomposition:**
        *   $d_1 = \mathbf{cov}_{2D}[0][0]$, $d_2 = \mathbf{cov}_{2D}[1][1]$, $o = \mathbf{cov}_{2D}[0][1]$ [S2, unofficial].
        *   $mid = 0.5(d_1 + d_2)$ [S2, unofficial].
        *   $radius = \sqrt{(\frac{d_1 - d_2}{2})^2 + o^2}$ [S2, unofficial].
        *   $\lambda_1 = mid + radius$ [S2, unofficial].
        *   $\lambda_2 = \max(mid - radius, 0.1)$ [S2, unofficial].
    *   **Basis Vectors:**
        *   $d = \text{normalize}(o, \lambda_1 - d_1)$ [S2, unofficial].
        *   $v_1 = \min(\sqrt{2\lambda_1}, 1024.0) \cdot d$ [S2, unofficial].
        *   $v_2 = \min(\sqrt{2\lambda_2}, 1024.0) \cdot d^\perp$ (where $d^\perp = (d_y, -d_x)$) [S2, unofficial].
    *   **Positioning and Culling:**
        *   Compute screen center: $v_{Center} = \frac{pos_{2d}}{w}$, where $pos_{2d}$ is the projected center [S2, unofficial].
        *   Compute final position for quad vertex $(x,y)$:
            $$
            p_{screen} = v_{Center} + \frac{x \cdot v_2 + y \cdot v_1}{viewport} \cdot 2.0
            $$
            Set $gl\_Position = (p_{screen}, 0.0, 1.0)$ [S2, unofficial].
        *   **Frustum Culling:**
            *   $bounds = 1.2 \cdot w$ [S2, unofficial].
            *   If $pos_{2d}.z < -pos_{2d}.w$ or $|pos_{2d}.x| > bounds$ or $|pos_{2d}.y| > bounds$, set $gl\_Position = (0,0,2,1)$ and return [S2, unofficial].

4.  **Fragment Shader Execution:**
    *   Compute squared distance from pixel to splat center: $A = -\mathbf{v}_{Position} \cdot \mathbf{v}_{Position}$ [S2, unofficial].
    *   If $A < -4.0$ (i.e., distance squared $> 4.0$), discard fragment [S2, unofficial].
    *   Compute alpha: $B = e^A \cdot v_{Color}.a$ [S2, unofficial].
    *   Output color: $gl\_FragColor = (B \cdot v_{Color}.rgb, B)$ [S2, unofficial].

5.  **Blending:**
    *   Use custom blending: `blendSrc = OneMinusDstAlphaFactor`, `blendDst = OneFactor`, `blendSrcAlpha = OneMinusDstAlphaFactor`, `blendDstAlpha = OneFactor` [S2, unofficial].
    *   Disable depth test and depth write [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def vertex_shader(instance_matrix, view_matrix, projection_matrix, viewport, fov, quad_vertex):
    # 1. Extract center and compute camera space
    center = instance_matrix.position
    center_cam = view_matrix * vec4(center, 1.0)
    x, y, z = center_cam.xyz

    # 2. Compute Jacobian
    fx = viewport.y / (2.0 * math.tan(fov * math.pi / 360.0))
    fy = fx  # Assuming square pixels or similar scaling
    J = mat3(
        fx / z, 0, -(fx * x) / (z * z),
        0, fy / z, -(fy * y) / (z * z),
        0, 0, 0
    )

    # 3. Compute 2D Covariance
    # Extract 3x3 rotation matrix from view_matrix
    W = mat3(view_matrix[0][0], view_matrix[0][1], view_matrix[0][2],
             view_matrix[1][0], view_matrix[1][1], view_matrix[1][2],
             view_matrix[2][0], view_matrix[2][1], view_matrix[2][2])
    T = transpose(W) * J
    cov_2d = transpose(T) * instance_matrix.covariance_3x3 * T

    # 4. Regularize
    cov_2d[0][0] += 0.3
    cov_2d[1][1] += 0.3

    # 5. Eigen Decomposition
    d1 = cov_2d[0][0]
    d2 = cov_2d[1][1]
    off = cov_2d[0][1]
    mid = 0.5 * (d1 + d2)
    radius = math.sqrt(((d1 - d2) / 2.0)**2 + off**2)
    lambda1 = mid + radius
    lambda2 = max(mid - radius, 0.1)

    # 6. Basis Vectors
    diag_vec = normalize(vec2(off, lambda1 - d1))
    v1 = min(math.sqrt(2.0 * lambda1), 1024.0) * diag_vec
    v2 = min(math.sqrt(2.0 * lambda2), 1024.0) * vec2(diag_vec.y, -diag_vec.x)

    # 7. Project Center
    pos_2d = projection_matrix * vec4(center, 1.0)
    v_center = pos_2d.xy / pos_2d.w

    # 8. Final Position
    projected_cov = v_center + (quad_vertex.x * v2 + quad_vertex.y * v1) / viewport * 2.0
    gl_Position = vec4(projected_cov, 0.0, 1.0)

    # 9. Culling
    bounds = 1.2 * pos_2d.w
    if pos_2d.z < -pos_2d.w or abs(pos_2d.x) > bounds or abs(pos_2d.y) > bounds:
        gl_Position = vec4(0, 0, 2, 1)
        return

def fragment_shader(v_position, v_color):
    A = -dot(v_position, v_position)
    if A < -4.0:
        return  # Discard fragment
    B = math.exp(A) * v_color.a
    return vec4(B * v_color.rgb, B)
```

## Operating characteristics
*   **Time Complexity:** Sorting is $O(N \log N)$ using JavaScript's native `.sort()` method, performed in a Web Worker to avoid blocking the main thread [S2, unofficial]. Rendering is $O(N \cdot P)$ where $P$ is the number of pixels covered by quads.
*   **Memory Complexity:** $O(N)$ for storing splat data and instance matrices [S2, unofficial].
*   **Performance:** Real-time on ordinary graphics hardware, significantly faster than NeRFs for inference [S1, unofficial]. Performance depends on vertex count and viewport resolution.
*   **Strengths:** No external dependencies beyond WebGL 1.0 and Three.js; works in standard web browsers; supports progressive loading [S1, S2, unofficial].
*   **Limitations:** Does not support view-dependent shading (spherical harmonics) in this basic implementation to reduce file size [S1, unofficial]. Precision issues may arise with very large scenes or extreme aspect ratios if not handled by the culling bounds [S2, unofficial].
*   **When to use:** Web-based visualization of 3DGS scenes where server-side rendering is not feasible.
*   **When not to use:** When view-dependent effects are required or when higher precision is needed than WebGL 1.0 provides [S1, unofficial].

## Background
The technique was popularized by the "3D Gaussian Splatting for Real-Time Radiance Field Rendering" paper, which introduced the method for generating photorealistic 3D scenes from images [S1, unofficial]. The specific WebGL implementation by antimatter15 demonstrated that 3DGS could be rendered efficiently on ordinary graphics hardware by extending point cloud rendering techniques, avoiding the neural network inference required by NeRFs [S1, unofficial]. It was implemented in JavaScript with WebGL 1.0 to ensure broad compatibility [S1, unofficial].

## Sources
[S1] antimatter15/splat, https://github.com/antimatter15/splat, fetched 2026-10-27 (unofficial)
[S2] akbartus/Gaussian-Splatting-WebViewers, https://github.com/akbartus/Gaussian-Splatting-WebViewers, fetched 2026-10-27 (unofficial)
