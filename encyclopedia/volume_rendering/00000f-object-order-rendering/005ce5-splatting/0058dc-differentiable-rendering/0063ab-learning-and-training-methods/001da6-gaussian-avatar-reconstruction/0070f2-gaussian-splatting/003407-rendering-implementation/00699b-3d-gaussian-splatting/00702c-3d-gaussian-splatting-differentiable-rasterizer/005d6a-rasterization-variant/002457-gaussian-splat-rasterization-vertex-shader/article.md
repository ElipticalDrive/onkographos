# Gaussian Splat Rasterization (Vertex Shader)

Also called: 2D Gaussian Splatting Vertex Shader, Fast Gaussian Rasterization Vertex Shader, GSplat Vert, Splat Vertex Shader, dsplat.vert

## Overview
This method implements the vertex shader stage of a differentiable 3D Gaussian splatting rasterizer, designed for high-performance rendering on GPU architectures that support geometry shaders or instanced rendering. It transforms 3D Gaussian parameters (mean and covariance) into screen-space 2D ellipses, computes the necessary basis vectors for quad generation, and performs early culling to optimize downstream processing. It differs from CUDA-based variants by offloading the heavy linear algebra and culling logic to the vertex/geometry shader pipeline, relying on texture buffers for data storage rather than linear memory layouts.

## How it works
### Inputs and Outputs
**Inputs:**
*   **Vertex Attributes:**
    *   `position` (`vec2`): Instance attribute defining the local quad basis coordinates (typically $\pm 1$ or similar range for the ellipse expansion).
    *   `index` (`uint`): Instance attribute identifying the specific Gaussian splat.
*   **Uniforms:**
    *   `P` (`mat4x4`): Projection matrix.
    *   `VM` (`mat4x4`): View-Model matrix.
    *   `focal` (`vec2`): Camera focal lengths $(f_x, f_y)$.
    *   `principal` (`vec2`): Camera principal point $(c_x, c_y)$.
    *   `basisViewport` (`vec2`): Viewport dimensions $(width, height)$.
    *   `discardAlpha` (`float`): Alpha threshold for culling. Default: $0.0001$ [S1, unofficial].
    *   `maxScreenSpaceSplatSize` (`float`): Maximum allowed screen-space size. Default: $2048.0$ [S1, unofficial].
    *   `sqrt8` (`float`): Constant $\sqrt{8}$. Used to scale eigenvalues to cover approximately 3 standard deviations [S1, unofficial].
*   **Texture Buffer (`u_texture`):**
    *   Type: `usampler2D`.
    *   Layout: Packed Gaussian data. Each splat index $i$ occupies two texels (columns).
        *   Center Texel: `uvec4` at `ivec2(((i & 0x3FF) << 1), i >> 10)`.
        *   Covariance/Color Texel: `uvec4` at `ivec2(((i & 0x3FF) << 1) | 1, i >> 10)`.
        *   This layout allows 1024 Gaussians per row [S2, unofficial].

**Outputs (Varyings to Geometry/Fragment Shader):**
*   `basisVector0` (`vec2`): First basis vector of the ellipse.
*   `basisVector1` (`vec2`): Second basis vector of the ellipse.
*   `gColor` (`vec4`): Color and alpha (clamped to 0 if culled).
*   `gDepth` (`float`): Depth of the splat center.

### Data Structures
The 3D covariance matrix $\mathbf{V}_{rk}$ is stored as two `vec3` attributes (`aCov0_3`, `aCov3_6`) or packed into the texture. It is a symmetric $3 \times 3$ matrix:
$$
\mathbf{V}_{rk} = \begin{bmatrix}
V_{00} & V_{01} & V_{02} \\
V_{01} & V_{11} & V_{12} \\
V_{02} & V_{12} & V_{22}
\end{bmatrix}
$$
Where:
*   $V_{00} = \text{aCov0\_3}.x$, $V_{01} = \text{aCov0\_3}.y$, $V_{02} = \text{aCov0\_3}.z$
*   $V_{11} = \text{aCov3\_6}.x$, $V_{12} = \text{aCov3\_6}.y$
*   $V_{22} = \text{aCov3\_6}.z$ [S1, unofficial]

### Steps
1.  **Alpha Culling:** If the input alpha `aColor.a` is less than `discardAlpha`, set output alpha `gColor.a` to $0.0$ and return immediately. This prevents the geometry shader from emitting quads for invisible splats [S1, unofficial].
2.  **Data Fetching:**
    *   Fetch the center position from the texture using the `index`. Unpack the `xyz` components from the `uvec4` center texel. Convert integer bits to float for position [S2, unofficial].
    *   Fetch the covariance and color data from the adjacent texel. Unpack half-precision components if necessary [S2, unofficial].
3.  **Position Transformation:**
    *   Compute view-space center: $\mathbf{c}_{view} = \mathbf{VM} \cdot [\mathbf{p}, 1]^T$.
    *   Compute clip-space center: $\mathbf{c}_{clip} = \mathbf{P} \cdot [\mathbf{p}, 1]^T$.
    *   Perform perspective division: $\mathbf{c}_{clip} = \mathbf{c}_{clip} / \mathbf{c}_{clip}.w$ [S1, unofficial].
    *   *Alternative (S2):* Transform center to camera space $\mathbf{c}_{cam} = \mathbf{view} \cdot [\mathbf{p}, 1]^T$. Project to clip space $\mathbf{pos}_{2d} = \mathbf{projection} \cdot \mathbf{c}_{cam}$.
4.  **Frustum Culling:**
    *   *Method 1 (S1):* Compute FOV limits.
        $$ \tan_{fov_x} = 0.5 \cdot \frac{width}{f_x}, \quad \tan_{fov_y} = 0.5 \cdot \frac{height}{f_y} $$
        $$ \text{lim}_{x+} = \frac{width - c_x}{f_x} + 0.3 \tan_{fov_x}, \quad \text{lim}_{x-} = \frac{c_x}{f_x} + 0.3 \tan_{fov_x} $$
        $$ \text{lim}_{y+} = \frac{height - c_y}{f_y} + 0.3 \tan_{fov_y}, \quad \text{lim}_{y-} = \frac{c_y}{f_y} + 0.3 \tan_{fov_y} $$
        Clip coordinates $x, y$ using these limits before Jacobian calculation [S1, unofficial].
    *   *Method 2 (S2):* Apply a 1.2x margin to the clip space bounds. If $\mathbf{pos}_{2d}.z < -\mathbf{pos}_{2d}.w$ or $\mathbf{pos}_{2d}.z > \mathbf{pos}_{2d}.w$ or $|\mathbf{pos}_{2d}.x| > 1.2 \mathbf{pos}_{2d}.w$ or $|\mathbf{pos}_{2d}.y| > 1.2 \mathbf{pos}_{2d}.w$, discard the splat by setting `gl_Position` to a clip-space point outside the view (e.g., $[0,0,2,1]$) [S2, unofficial].
5.  **Jacobian Computation:**
    *   Compute the Jacobian $\mathbf{J}$ of the projection transformation.
    *   Let $z = \mathbf{c}_{view}.z$ (or $\mathbf{c}_{cam}.z$).
    *   $r_z = 1/z$, $r_{z2} = r_z^2$.
    *   $$ \mathbf{J} = \begin{bmatrix}
    f_x r_z & 0 & -f_x x r_{z2} \\
    0 & f_y r_z & -f_y y r_{z2} \\
    0 & 0 & 0
    \end{bmatrix} $$
    *   Where $x, y$ are the view-space coordinates of the center [S1, S2, unofficial].
6.  **Covariance Transformation:**
    *   Compute the transformation matrix $\mathbf{T} = \mathbf{W} \mathbf{J}$, where $\mathbf{W} = \text{transpose}(\mathbf{VM})$ (or $\text{transpose}(\mathbf{view})$) [S1, S2, unofficial].
    *   Compute the 2D covariance matrix in clip/camera space:
        $$ \mathbf{C}_{2D} = \mathbf{T}^T \mathbf{V}_{rk} \mathbf{T} $$
    *   Extract the upper-left $2 \times 2$ submatrix.
    *   **Blur Term:** Add a blur term to the diagonal to account for downsampling and anti-aliasing.
        *   S1 adds $0.3$ to both diagonal elements: $\mathbf{C}_{2D}[0][0] += 0.3$, $\mathbf{C}_{2D}[1][1] += 0.3$.
        *   S2 adds $0.1$ to both diagonal elements: $\mathbf{C}_{2D}[0][0] += 0.1$, $\mathbf{C}_{2D}[1][1] += 0.1$ [S1, S2, unofficial].
7.  **Eigenvalue Decomposition:**
    *   Let $a = \mathbf{C}_{2D}[0][0]$, $b = \mathbf{C}_{2D}[0][1]$, $d = \mathbf{C}_{2D}[1][1]$.
    *   Compute determinant $D = ad - b^2$.
    *   Compute trace $T = a + d$.
    *   Compute eigenvalues:
        $$ \lambda_1 = \frac{T}{2} + \sqrt{\max(0.1, (\frac{T}{2})^2 - D)} $$
        $$ \lambda_2 = \frac{T}{2} - \sqrt{\max(0.1, (\frac{T}{2})^2 - D)} $$
    *   *Note:* S2 uses a slightly different formulation for stability:
        $$ \text{mid} = \frac{a+d}{2}, \quad \text{radius} = \sqrt{(\frac{a-d}{2})^2 + b^2} $$
        $$ \lambda_1 = \text{mid} + \text{radius}, \quad \lambda_2 = \text{mid} - \text{radius} $$
    *   Compute the major axis direction:
        $$ \mathbf{v} = \text{normalize}([b, \lambda_1 - a]) $$
    *   Compute basis vectors scaled by $\sqrt{8}$ (approx 3 sigma) and clamped to a maximum size (e.g., 1024.0):
        $$ \mathbf{basis}_0 = \sqrt{8} \cdot \min(\sqrt{2\lambda_1}, 1024.0) \cdot \mathbf{v} $$
        $$ \mathbf{basis}_1 = \sqrt{8} \cdot \min(\sqrt{2\lambda_2}, 1024.0) \cdot \mathbf{v}_{\perp} $$
        Where $\mathbf{v}_{\perp} = [v_y, -v_x]$ [S1, S2, unofficial].
8.  **Output Generation:**
    *   Pass the color (unpacked from texture if using S2 layout) and depth to the geometry shader.
    *   Output the computed basis vectors.
    *   Set `gl_Position` to the projected center position (S2) or pass through (S1, relying on geometry shader for final position) [S1, S2, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def vertex_shader(index, position, aPos, aCov0_3, aCov3_6, aColor, 
                  P, VM, focal, principal, viewport, discardAlpha):
    # 1. Alpha Culling
    if aColor.a < discardAlpha:
        return discard_splat()

    # 2. Reconstruct 3D Covariance Matrix Vrk
    Vrk = mat3(
        aCov0_3.x, aCov0_3.y, aCov0_3.z,
        aCov0_3.y, aCov3_6.x, aCov3_6.y,
        aCov0_3.z, aCov3_6.y, aCov3_6.z
    )

    # 3. Transform Center
    viewCenter = VM * vec4(aPos, 1.0)
    clipCenter = P * vec4(aPos, 1.0)
    clipCenter /= clipCenter.w

    # 4. Frustum Culling (Simplified S1/S2 hybrid)
    # Check if behind camera or outside clip space
    if viewCenter.z <= 0.0 or clipCenter.z < -clipCenter.w or clipCenter.z > clipCenter.w:
        return discard_splat()
    
    # Optional: Additional FOV margin culling (S1) or 1.2x margin (S2)
    # For brevity, we assume basic clip space culling is sufficient for the core method.

    # 5. Compute Jacobian J
    z = viewCenter.z
    rz = 1.0 / z
    rz2 = rz * rz
    x = viewCenter.x
    y = viewCenter.y
    
    fx, fy = focal
    J = mat3(
        fx * rz, 0.0, -fx * x * rz2,
        0.0, fy * rz, -fy * y * rz2,
        0.0, 0.0, 0.0
    )

    # 6. Transform Covariance to 2D
    W = transpose(mat3(VM))
    T = W * J
    cov2Dm = transpose(T) * Vrk * T
    
    # Extract 2x2 upper-left
    a = cov2Dm[0][0]
    b = cov2Dm[0][1]
    d = cov2Dm[1][1]

    # 7. Add Blur Term
    a += 0.3 # S1 blur
    d += 0.3 # S1 blur

    # 8. Eigenvalue Decomposition
    trace = a + d
    det = a * d - b * b
    term = sqrt(max(0.1, (trace * 0.5) ** 2 - det))
    
    lambda1 = trace * 0.5 + term
    lambda2 = trace * 0.5 - term

    # 9. Validate Eigenvalues
    if lambda1 <= 0.01 or lambda2 <= 0.01 or lambda1 < lambda2:
        return discard_splat()
    
    # Check for degenerate aspect ratio (optional but recommended)
    if lambda1 / lambda2 > 10000.0:
        return discard_splat()

    # 10. Compute Basis Vectors
    # Major axis direction
    if abs(lambda1 - a) < 1e-6:
        diag_vec = vec2(0.0, 1.0)
    else:
        diag_vec = normalize(vec2(b, lambda1 - a))
        
    minor_vec = vec2(diag_vec.y, -diag_vec.x)

    scale = sqrt(8.0) # 3 sigma
    max_size = 1024.0
    
    basis0 = scale * min(sqrt(2.0 * lambda1), max_size) * diag_vec
    basis1 = scale * min(sqrt(2.0 * lambda2), max_size) * minor_vec

    # 11. Output
    gColor = aColor
    gDepth = clipCenter.z
    return basis0, basis1, gColor, gDepth, clipCenter
```

### Edge Cases and Pitfalls
*   **Invalid Covariance:** If the determinant $D \le 0$ or diagonal elements are non-positive, the matrix is not positive definite. This results in invalid eigenvalues. The splat must be discarded to prevent artifacts and zero gradients [S1, unofficial].
*   **Negative Eigenvalues:** If $\lambda_2 < 0$, the covariance is invalid. Discard the splat [S2, unofficial].
*   **Degenerate Eigenvalues:** If $\lambda_1 \approx \lambda_2$ or the ratio $\lambda_1/\lambda_2$ is extremely large ($>10000$), the ellipse is ill-conditioned. This can lead to numerical instability in the geometry shader or fragment shader. Culling or clamping is required [S1, unofficial].
*   **Precision:** Half-precision unpacking in the texture fetch (S2) may introduce quantization errors in the covariance matrix. Ensure the reconstruction logic matches the packing format exactly.
*   **Blur Term Sensitivity:** The added blur term ($0.1$ or $0.3$) significantly affects the visual smoothness and aliasing. It must be tuned to the resolution and downsampling factors of the pipeline.

## Operating characteristics
*   **Time Complexity:** $O(N)$ for $N$ Gaussians. The vertex shader performs constant-time linear algebra per splat.
*   **Memory Complexity:** $O(N)$ for the texture buffer storing Gaussian data. The texture layout (2 texels per splat) requires $2 \times N \times 16$ bytes (if `uvec4`) or less if packed more tightly.
*   **Performance:** This method is highly efficient on GPUs with robust geometry shader support. It reduces CPU-GPU synchronization overhead compared to CPU-side sorting and culling. However, it may be less flexible than CUDA-based implementations for custom culling strategies or hybrid rendering pipelines.
*   **Strengths:** Simple integration into standard OpenGL/Vulkan pipelines; leverages hardware-accelerated vertex processing; early culling reduces geometry shader load.
*   **Limitations:** Geometry shader emission can be a bottleneck if culling is not aggressive enough; less control over memory layout compared to CUDA; debugging linear algebra in shaders is harder.

## Background
This method is part of the 3D Gaussian Splatting rendering pipeline, specifically the vertex shader variant. It was developed to enable real-time differentiable rendering of 3D Gaussians using standard graphics APIs. The approach adapts the mathematical foundation of 3D Gaussian Splatting [S1, S2, unofficial] to the vertex shader stage, computing the 2D projection and basis vectors on the GPU. It is often used in conjunction with a geometry shader that generates the actual quads for rasterization.

## Sources
[S1] dendenxu/fast-gaussian-rasterization, https://github.com/dendenxu/fast-gaussian-rasterization, fetched 2026-10-26 (unofficial)
[S2] liaozhangsheng/gsplat-opengl, https://github.com/liaozhangsheng/gsplat-opengl, fetched 2026-10-28 (unofficial)
