# Animated 3D Gaussian Splatting (Splatv rendering)
Also called: Splatv vertex shader, VideoRenderProgram

## Overview
Animated 3D Gaussian Splatting is a GPU-accelerated rendering method for time-varying 3D Gaussian splats that computes time-dependent position, rotation, and scale deltas from packed texture data to enable smooth animation without per-frame CPU updates [S1, unofficial]. It differs from static 3D Gaussian Splatting by evaluating polynomial motion deltas and applying temporal opacity decay in the vertex and fragment shaders, building on the screen-space quad rendering of [Gaussian Splat Rendering (Screen-Space Quad)](2075-gaussian-splat-rendering-screen-space-quad.md).

## How it works
**Inputs and Outputs:**
*   **Inputs:** A packed 2D texture (`u_texture`) containing static and motion data for $N$ splats; camera matrices (View $V$, Projection $P$); camera intrinsics (`focal` vector $(f_x, f_y)$); current animation timestamp (`time`); viewport dimensions (`viewport` $(W, H)$); and the splat index `i`.
*   **Outputs:** Screen-space quad vertices for each splat; final color and alpha per fragment.

**Data Structures:**
Splat data is packed into a 2D texture where each splat index `i` occupies 4 consecutive texels [S1, unofficial].
*   **Row Index:** `row = uint(i) >> 10`
*   **Column Index:** `col = ((uint(i) & 0x3ffu) << 2) | k`, where $k \in \{0, 1, 2, 3\}$ is the texel offset within the splat's block.
*   **Texel Layout:**
    *   `.x`: `motion0.x`
    *   `.y`: `motion0.y`
    *   `.z`: `motion0.z`
    *   `.w`: `motion1.w`
*   **Static Data:** `static0` and `static1` are fetched at offsets relative to the base index (specifically `| 0` and `| 1` respectively in the source context, implying adjacent blocks or specific packing offsets not fully detailed beyond the motion block) [S1, unofficial].

**Numbered Steps:**

1.  **Fetch Motion Data:** Unpack `motion0` and `motion1` from the texture using the row/column coordinates derived from the splat index `i` [S1, unofficial].
2.  **Compute Time Delta:** Calculate $dt = t - t_{start}$, where $t$ is the uniform `time` and $t_{start}$ is unpacked from `motion1.w` as the x-component of a half-float pair [S1, unofficial].
3.  **Compute Motion Deltas:** Calculate position delta $\Delta p$ and rotation delta $\Delta q$ as polynomials in $dt$ [S1, unofficial]:
    $$ \Delta p = c_{p1} dt + c_{p2} dt^2 + c_{p3} dt^3 $$
    $$ \Delta q = c_{q1} dt $$
    Coefficients $c_{p1..3}$ and $c_{q1}$ are unpacked from `motion0` and `motion1` as half-float pairs [S1, unofficial].
4.  **Transform Position:** Add $\Delta p$ to the static world position $p_{static}$ (converted from fixed-point or half-float representation) and transform to camera space:
    $$ p_{cam} = V \cdot (p_{static} + \Delta p, 1)^T $$
    Then project to clip space:
    $$ p_{clip} = P \cdot p_{cam} $$
5.  **Clip:** Discard the splat if it is outside the extended view frustum:
    $$ |p_{clip}.x| > 1.2 p_{clip}.w, \quad |p_{clip}.y| > 1.2 p_{clip}.w, \quad \text{or} \quad p_{clip}.z < -1.2 p_{clip}.w $$
    [S1, unofficial].
6.  **Reconstruct Rotation:** Compute the normalized quaternion $q$ by adding the rotation delta to the static quaternion components and normalizing:
    $$ q = \text{normalize}(q_{static} + \Delta q) $$
    [S1, unofficial].
7.  **Compute World Covariance:** Construct the scale matrix $S = \text{diag}(s_x, s_y, s_z)$ and rotation matrix $R$ from $q$. Compute the transformation matrix $M = S R$ and the world covariance matrix:
    $$ \Sigma_{world} = 4 M^T M $$
    [S1, unofficial].
8.  **Compute 2D Covariance:** Calculate the 2D covariance matrix $\Sigma_{2D}$ using the Jacobian $J$ of the projection and the view rotation $V_{rot}$ (upper-left 3x3 of $V$) [S1, unofficial]:
    $$ T = V_{rot}^T J $$
    $$ \Sigma_{2D} = T^T \Sigma_{world} T $$
    The Jacobian $J$ is defined as:
    $$ J = \begin{bmatrix} f_x/z & 0 & -f_x x/z^2 \\ 0 & -f_y/z & f_y y/z^2 \\ 0 & 0 & 0 \end{bmatrix} $$
    where $(x,y,z)$ are the camera-space coordinates from $p_{cam}$ [S1, unofficial].
9.  **Eigenvalue Decomposition:** Compute eigenvalues $\lambda_1, \lambda_2$ and eigenvector $v$ of $\Sigma_{2D}$ analytically [S1, unofficial]:
    $$ \mu = \frac{\Sigma_{2D}[0][0] + \Sigma_{2D}[1][1]}{2} $$
    $$ r = \sqrt{\left(\frac{\Sigma_{2D}[0][0] - \Sigma_{2D}[1][1]}{2}\right)^2 + \Sigma_{2D}[0][1]^2} $$
    $$ \lambda_1 = \mu + r, \quad \lambda_2 = \mu - r $$
    Discard the splat if $\lambda_2 < 0$ [S1, unofficial].
10. **Compute Ellipse Axes:** Determine the screen-space axes $a$ and $b$ [S1, unofficial]:
    $$ v = \text{normalize}([\Sigma_{2D}[0][1], \lambda_1 - \Sigma_{2D}[0][0]]) $$
    $$ a = \min(\sqrt{2\lambda_1}, 1024) v $$
    $$ b = \min(\sqrt{2\lambda_2}, 1024) [v_y, -v_x] $$
    The maximum axis length is clamped to 1024 pixels to prevent overdraw [S1, unofficial].
11. **Compute Screen Position:** For each quad vertex with local position $(u,v)$, compute the final screen position:
    $$ p_{screen} = \frac{p_{clip}}{p_{clip}.w} + u \frac{a}{W} + v \frac{b}{H} $$
    where $(W,H)$ is the viewport size [S1, unofficial].
12. **Fragment Evaluation:** In the fragment shader, evaluate the Gaussian kernel based on the local position $p_{local}$ (relative to the quad center) [S1, unofficial]:
    $$ A = -\|p_{local}\|^2 $$
    Discard if $A < -4$ [S1, unofficial].
13. **Compute Final Color:** Calculate the time-decayed opacity $\alpha_{time}$ and final color [S1, unofficial]:
    $$ \alpha_{time} = \exp\left(-\left(\frac{dt}{\tau}\right)^2\right) $$
    where $\tau$ is unpacked from `motion1.w` as the y-component of a half-float pair.
    $$ C = \exp(A) \alpha_{time} C_{base} $$
    Apply depth fog factor $\min(\frac{z}{w}+1, 1)$ if required by the specific implementation context [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Vertex Shader Core
def render_splat_vertex(splat_index, view_matrix, projection_matrix, focal, time, viewport):
    # 1. Fetch Data
    row = splat_index >> 10
    col = ((splat_index & 0x3ff) << 2)
    
    # Unpack motion data (assuming half-float unpacking helper exists)
    motion0 = unpack_half4(texture_2d(u_texture, row, col))
    motion1 = unpack_half4(texture_2d(u_texture, row, col + 1)) # Assuming adjacent or specific layout
    
    # Static data fetch (offsets implied by source)
    static0 = unpack_half4(texture_2d(u_texture, row, col)) # Placeholder for static0 fetch logic
    static1 = unpack_half4(texture_2d(u_texture, row, col + 1)) # Placeholder for static1 fetch logic
    
    # 2. Time Delta
    t_start = motion1.w.x # x-component of half-float pair in motion1.w
    tau = motion1.w.y     # y-component of half-float pair in motion1.w
    dt = time - t_start
    
    # 3. Motion Deltas
    # Coefficients unpacked from motion0/motion1
    cp1, cp2, cp3 = unpack_motion_coeffs(motion0)
    cq1 = unpack_rotation_coeff(motion1)
    
    delta_p = cp1 * dt + cp2 * (dt ** 2) + cp3 * (dt ** 3)
    delta_q = cq1 * dt
    
    # 4. Transform Position
    p_static = vec4(unpack_static_pos(static0), 1.0)
    p_cam = view_matrix * (p_static + vec4(delta_p, 0.0))
    p_clip = projection_matrix * p_cam
    
    # 5. Clip
    if abs(p_clip.x) > 1.2 * p_clip.w or abs(p_clip.y) > 1.2 * p_clip.w or p_clip.z < -1.2 * p_clip.w:
        return discard
    
    # 6. Reconstruct Rotation
    q_static = unpack_quaternion(static1)
    q = normalize(q_static + vec4(delta_q, 0.0)) # Assuming delta_q is axis-angle or similar vector
    R = quaternion_to_matrix(q)
    
    # 7. World Covariance
    S = diag(unpack_scale(static0)) # Assuming scale in static0 or similar
    M = S @ R
    Sigma_world = 4.0 * (M.transpose() @ M)
    
    # 8. 2D Covariance
    fx, fy = focal
    x, y, z = p_cam.xyz
    J = mat3(
        fx / z, 0.0, -(fx * x) / (z ** 2),
        0.0, -fy / z, (fy * y) / (z ** 2),
        0.0, 0.0, 0.0
    )
    V_rot = view_matrix[0:3, 0:3]
    T = V_rot.transpose() @ J
    Sigma_2D = T.transpose() @ Sigma_world @ T
    
    # 9. Eigenvalues
    mu = (Sigma_2D[0][0] + Sigma_2D[1][1]) / 2.0
    r = sqrt(((Sigma_2D[0][0] - Sigma_2D[1][1]) / 2.0) ** 2 + Sigma_2D[0][1] ** 2)
    lambda_1 = mu + r
    lambda_2 = mu - r
    
    if lambda_2 < 0:
        return discard
        
    # 10. Axes
    v = normalize(vec2(Sigma_2D[0][1], lambda_1 - Sigma_2D[0][0]))
    a = min(sqrt(2 * lambda_1), 1024.0) * v
    b = min(sqrt(2 * lambda_2), 1024.0) * vec2(v.y, -v.x)
    
    # 11. Screen Position (for each quad vertex u,v)
    W, H = viewport
    p_screen = p_clip / p_clip.w + u * (a / W) + v * (b / H)
    
    return p_screen, v_color, v_position_local

# Fragment Shader Core
def render_splat_fragment(v_position_local, v_color, time, t_start, tau):
    A = -dot(v_position_local, v_position_local)
    if A < -4.0:
        return discard
    
    dt = time - t_start
    alpha_time = exp(-((dt / tau) ** 2))
    
    B = exp(A) * v_color.a * alpha_time
    frag_color = vec4(B * v_color.rgb, B)
    
    return frag_color
```

**Parameters:**
*   `focal`: Camera focal length vector $(f_x, f_y)$. Typical value: derived from camera intrinsics [S1, unofficial].
*   `time`: Current animation timestamp in seconds. Used to compute $dt$ for motion interpolation [S1, unofficial].
*   `viewport`: Canvas dimensions $(W, H)$ in pixels. Used to scale ellipse axes to screen space [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Negative Eigenvalue:** $\lambda_2 < 0$ indicates a degenerate or flipped covariance matrix, often caused by extreme perspective or invalid transforms. The shader discards the splat to prevent rendering artifacts [S1, unofficial].
*   **Axis Clamping:** Axis lengths are clamped to a maximum of 1024 pixels to prevent overdraw and performance degradation from excessively large splats [S1, unofficial].
*   **Precision:** Coefficients are stored as half-floats, which may limit the precision of high-frequency motion or very small $dt$ values [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ for $N$ splats, performed entirely on the GPU in the vertex and fragment shaders. No CPU sorting or per-frame update overhead for motion [S1, unofficial].
*   **Memory Complexity:** $O(N)$ for the packed texture storing static and motion data. The motion data adds overhead to the static splat representation [S1, unofficial].
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Enables smooth, real-time animation of Gaussian splats without CPU-side interpolation or sorting updates per frame [S1, unofficial].
*   **Limitations:** Relies on pre-computed polynomial coefficients, limiting the complexity of motion to cubic polynomials for position and linear for rotation [S1, unofficial]. Precision is limited by half-float storage [S1, unofficial].
*   **When to Use:** When rendering animated 3D Gaussian Avatars or dynamic scenes where motion is smooth and can be approximated by low-order polynomials [S1, unofficial].
*   **When Not to Use:** For highly discontinuous or complex non-polynomial motion, or when higher precision is required than half-floats allow [S1, unofficial].

## Background
Animated 3D Gaussian Splatting (Splatv rendering) is an extension of 3D Gaussian Splatting designed for time-varying scenes. It was introduced in the context of GPU-accelerated web-based Gaussian Splatting implementations, specifically within the `gsplat.js` project [S1, unofficial]. The method allows for the rendering of animated Gaussian splats by embedding motion parameters directly into the texture data and evaluating them in the shader pipeline [S1, unofficial].

## Sources
[S1] huggingface/gsplat.js, https://github.com/huggingface/gsplat.js, fetched 2026-10-27 (unofficial)
