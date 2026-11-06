# Phong Splatting

Also called: Per-pixel lighting splatting, Phong Splat Generation, Splatting with linear normal fields

## Overview
Phong Splatting is a surface splatting technique that assigns a linearly varying normal field to each elliptical splat, enabling per-pixel Phong shading without surface connectivity [S1]. It solves the piecewise constant shading artifacts of standard point-based rendering by providing smooth, Phong-like shading quality on modern GPUs [S1]. Unlike its sibling [EWA Splatting](1069-ewa-splatting.md), which uses a simplified isotropic kernel, Phong Splatting uses elliptical splats with projectively correct per-pixel parameterization, trading the simplicity of isotropic filtering for superior visual quality and exact depth values [S1].

## How it works

### Inputs and Outputs
*   **Input:** A dense set of surface sample points $P = \{p_i\}$ with associated normal vectors $n_i$ [S1].
*   **Output:** A set of elliptical surface splats $S_j$, each defined by a center $c_j$, scaled orthogonal tangent vectors $u_j, v_j$, and linear normal field parameters [S1].

### Data Structures
Each splat $S_j$ requires **17 floating-point values** [S1]:
1.  **Center:** $c_j$ (3 floats: $x, y, z$ in eye space) [S1].
2.  **Tangents:** $u_j, v_j$ (6 floats: 3 for each vector, scaled by ellipse radii) [S1].
3.  **Normal Field Parameters:**
    *   $\bar{n}$ (2 floats: $\bar{x}, \bar{y}$ representing the center normal on the offset tangent plane) [S1].
    *   $\alpha, \beta$ (2 floats: linear coefficients for the x and y components of the normal field) [S1].
    *   *Note:* The third component of the normal is implicitly 1 (homogeneous representation) [S1].
4.  **Clipping Data:** Optional scalars $(a, b, c)$ defining clip lines in tangent space (3 floats per line) [S1].
5.  **Precomputed Depth Factor:** $n \cdot (c^T n)^{-1}$ (1 float, computed in vertex shader) [S1].
*Comparison:* Standard elliptical splatting requires only 12 floats per splat [S1]. However, Phong Splatting requires an order of magnitude fewer splats for comparable quality [S1].

### Steps

#### 1. Splat Generation (Preprocessing)
Given input samples $P$ and error tolerance $\epsilon$:
1.  **Selection:** Generate an initial set of elliptical splats $S_j$ covering $P$ using a curvature-aware selection and global relaxation algorithm (e.g., [WK04]) [S1].
2.  **Association:** For each splat $S_j$, identify the subset $P_j \subset P$ of samples contained within the elliptical cylinder defined by $S_j$'s tangents and offset by $\pm \epsilon$ along the normal [S1].
3.  **Frame Construction:** Align $u_j$ and $v_j$ with principal curvature directions. If the geometry fitting scheme does not provide this, estimate them as eigenvectors of the covariance matrix of sample normals in $P_j$ [S1].
4.  **Normal Fitting:**
    *   Represent each sample normal $n_i \in P_j$ in the local frame $(u_j, v_j, u_j \times v_j)$ [S1].
    *   Set the third coordinate (normal direction) to 1. The normal is thus represented as a point $(x_i, y_i)$ on an offset tangent plane [S1].
    *   Solve for the linear field parameters $\bar{n}=(\bar{x}, \bar{y})$, $\alpha$, and $\beta$ using least squares. The systems are uncoupled:
        $$
        \begin{pmatrix} |P_j| & \sum u_i \\ \sum u_i & \sum u_i^2 \end{pmatrix} \begin{pmatrix} \bar{x} \\ \alpha \end{pmatrix} = \begin{pmatrix} \sum x_i \\ \sum x_i u_i \end{pmatrix}
        $$
        $$
        \begin{pmatrix} |P_j| & \sum v_i \\ \sum v_i & \sum v_i^2 \end{pmatrix} \begin{pmatrix} \bar{y} \\ \beta \end{pmatrix} = \begin{pmatrix} \sum y_i \\ \sum y_i v_i \end{pmatrix}
        $$
    *   If $|P_j| < 2$ (underdetermined), compute the least-norm solution using the pseudo-inverse [S1].

#### 2. Rendering (Per-Splat)
Render each splat $S_j$ as a single point primitive (vertex at $c_j$) [S1]. Pass $u_j, v_j, \bar{n}, \alpha, \beta$ as texture coordinates to the fragment shader [S1].

**Vertex Shader:**
1.  Transform $c_j$ to window coordinates [S1].
2.  Transform $u_j, v_j, n_j$ to eye space [S1].
3.  Compute the rasterization square size $size$ using the larger ellipse radius $r = \max(\|u_j\|, \|v_j\|)$:
    $$
    size = 2r \cdot \frac{n}{z_{eye}} \cdot \frac{h_{vp}}{t-b}
    $$
    where $n, t, b$ are frustum near, top, bottom parameters, $h_{vp}$ is viewport height, and $z_{eye}$ is the eye-space depth of $c_j$ [S1].
4.  Precompute the ray intersection factor: $k = n_j \cdot (c_j^T n_j)^{-1}$ [S1].
5.  Pass $u_j, v_j, \bar{n}, \alpha, \beta$ as texture coordinates to the fragment shader [S1].

**Fragment Shader:**
For each pixel in the $size \times size$ square:
1.  **Ray Casting:** Compute the point $q_n$ on the near plane corresponding to window pixel $(x, y)$:
    $$
    q_n = \begin{pmatrix} x \frac{w_h}{w_{vp}} - \frac{w_h}{2} \\ y \frac{h_n}{h_{vp}} - \frac{h_n}{2} \\ -n \end{pmatrix}
    $$
    where $w_h, h_n$ are near plane width/height, and $w_{vp}, h_{vp}$ are viewport dimensions [S1].
2.  **Intersection:** Compute the point $q$ on the splat's supporting plane:
    $$
    q = q_n \cdot k
    $$
    where $k = \frac{c_j^T n_j}{q_n^T n_j}$ [S1].
3.  **Local Parameters:** Compute $(u, v)$ by projecting $q - c_j$ onto the tangents:
    $$
    u = u_j^T (q - c_j), \quad v = v_j^T (q - c_j)
    $$
4.  **Inside Test:** If $u^2 + v^2 > 1$, discard the fragment [S1].
5.  **Clipping (Optional):** If the splat has clip lines $(a, b, c)$, discard if $au + bv + c < 0$ [S1].
6.  **Normal Interpolation:** Compute the interpolated normal vector components on the offset plane:
    $$
    x_{interp} = \bar{x} + \alpha u, \quad y_{interp} = \bar{y} + \beta v
    $$
    The full normal in eye space is reconstructed from $(x_{interp}, y_{interp}, 1)$ in the local frame $(u_j, v_j, n_j)$ [S1].
7.  **Lighting:** Normalize the interpolated normal and use it to sample a precomputed cube map containing lighting terms (diffuse/specular intensities) [S1].
8.  **Depth:** Compute the exact eye-space depth $q_z$ of fragment $q$ for correct blending [S1].

#### 3. Blending and Filtering
To achieve high-quality antialiasing and correct transparency:
1.  **Pass 1 (Visibility Splatting):** Render all splats with alpha blending. Assign a radially decreasing weight $w(u,v)$ to each pixel (e.g., Gaussian falloff). Accumulate weighted color and weight sum [S1].
2.  **Pass 2 (Normalization):** Render a full-screen quad textured with the accumulated color and weight sum. Divide the color by the weight sum per pixel [S1].
*Optimization:* Render non-clipped splats first, then splats with one clip line, then two clip lines, to minimize shader state switches [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_splat(center, tangents, normal_params, clip_lines=None):
    # Vertex shader
    c_win = transform_to_window(center)
    u_eye, v_eye, n_eye = transform_to_eye(tangents[0], tangents[1], normal_params[0])
    r = max(norm(u_eye), norm(v_eye))
    size = 2 * r * (n / z_eye) * (h_vp / (t - b))
    k = dot(n_eye, inverse(dot(center, n_eye)))
    pass_to_fragment(u_eye, v_eye, normal_params, clip_lines, k)

    # Fragment shader
    for pixel in rasterize_square(size):
        x, y = pixel.window_coords
        q_n = vec3(x * (w_h / w_vp) - w_h/2, y * (h_n / h_vp) - h_n/2, -n)
        q = q_n * k
        u = dot(u_eye, q - center)
        v = dot(v_eye, q - center)
        if u*u + v*v > 1: discard
        if clip_lines:
            for a, b, c in clip_lines:
                if a*u + b*v + c < 0: discard
        x_interp = normal_params[1] + normal_params[2] * u
        y_interp = normal_params[3] + normal_params[4] * v
        normal_local = vec3(x_interp, y_interp, 1.0)
        normal_eye = reconstruct_normal(normal_local, u_eye, v_eye, n_eye)
        color = sample_cube_map(normalize(normal_eye))
        depth = q.z
        return color, depth
```

### Parameters
*   **Error Tolerance $\epsilon$:** Controls the coverage of splats during generation [S1].
*   **Frustum Parameters:** $n, t, b, h_{vp}$ are standard camera settings [S1].

### Edge Cases and Pitfalls
*   **Underdetermined Systems:** If a splat covers fewer than 2 samples, the normal fitting linear system is underdetermined. Use the pseudo-inverse to find the least-norm solution [S1].
*   **Affine Approximation:** Do not use an affine approximation for projection. It disables the exact per-pixel parameterization required for correct clipping and depth, and while it allows EWA filtering, the object-space filter alone is sufficient [S1].
*   **Backface Culling:** Optionally perform a conservative backface test in the vertex shader for closed models using the transformed normal [S1].

## Operating characteristics

### Time and Memory Complexity
*   **Memory:** 17 floats per splat (vs. 12 for standard elliptical splatting) [S1]. However, because Phong Splatting achieves comparable visual quality with ~10x fewer splats than flat shading, total memory usage is lower [S1].
*   **Speed:** Achieves up to 4 million Phong-shaded, filtered, and blended splats per second on modern GPUs [S1].

### Measured Performance
*   **Quality vs. Count:** A torso model with 3,000 splats rendered with Phong Splatting achieves visual quality comparable to Phong-shaded triangle meshes. Flat shading requires 170,000 splats, and flat shading with Gaussian blending requires 33,000 splats for comparable quality [S1].

### Strengths and Limitations
*   **Strengths:** Projectively correct per-pixel parameterization; exact depth values prevent blending artifacts; superior shading quality compared to flat or Gouraud-like splatting; efficient GPU implementation using cube maps for lighting [S1].
*   **Limitations:** Cannot use EWA filtering (combined object/image space Gaussian) because it relies on exact projective mapping rather than affine approximation [S1]. Requires preprocessing to generate splats and fit normal fields [S1].

### When to Use
*   Use when high visual quality (smooth shading) is required for point-based surfaces without connectivity [S1].
*   Use when rendering sharp features or clipped splats, as the method supports exact tangent-space clipping [S1].

### When Not to Use
*   Use [EWA Splatting](1069-ewa-splatting.md) if isotropic filtering and simpler kernel implementation are preferred over per-pixel shading quality [S1].
*   Use [Deferred Surface Splatting](1072-deferred-surface-splatting.md) if deferred shading pipelines are already in place and per-pixel lighting overhead is a concern [S1].

## Background
Phong Splatting was introduced in [S1]. It builds on the concept of varying normal fields proposed by [KV01], but differs by using a dense input point cloud and a linear normal field fit over the splat area, rather than local Taylor expansions at the center. It addresses the limitations of earlier point-based rendering systems that used constant normals, resulting in flat shading artifacts [S1]. The method leverages the programmable pipeline of modern GPUs, specifically vertex and fragment shaders, to achieve real-time performance [S1].

## Sources
[S1] Phong Splatting, https://doi.org/10.2312/spbg/spbg04/025-032, fetched 2026-10-26
