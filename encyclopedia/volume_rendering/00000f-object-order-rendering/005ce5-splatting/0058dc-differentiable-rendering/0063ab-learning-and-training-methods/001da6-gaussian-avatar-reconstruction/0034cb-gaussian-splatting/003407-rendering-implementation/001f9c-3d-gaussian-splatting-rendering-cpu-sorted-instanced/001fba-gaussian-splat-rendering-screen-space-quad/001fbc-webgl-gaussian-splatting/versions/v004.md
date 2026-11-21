# WebGL Gaussian Splatting

Also called: Gaussian Splat Rendering Pipeline, Gaussian Splatting Web Rendering (Three.js), Gaussian Splatting WebGL, Three.js Gaussian Splatting, Web-based 3D Gaussian Splatting Rendering, WebGL 1.0 Gaussian Splatting

## Overview
WebGL Gaussian Splatting is a real-time rendering technique for 3D Gaussian Splatting (3DGS) implemented in web browsers using WebGL 1.0, with variants utilizing the Three.js library or running without external dependencies [S1, S2, unofficial]. It solves the problem of displaying large-scale, photorealistic 3D scenes reconstructed via Gaussian Splatting on client-side hardware without requiring neural network inference or server-side ray tracing [S1, unofficial]. It sets itself apart from its siblings by utilizing a CPU-side depth sorting algorithm (often offloaded to a Web Worker) to resolve the non-commutative nature of alpha blending, whereas the parent [Gaussian Splat Rendering (Screen-Space Quad)](2075-gaussian-splat-rendering-screen-space-quad.md) describes the general projection and quad-drawing mechanism without specifying the sorting implementation [S2, unofficial]. Unlike [Animated 3D Gaussian Splatting (Splatv rendering)](7758-animated-3d-gaussian-splatting-splatv-rendering.md), this method typically omits spherical harmonics to reduce file size and memory footprint for web delivery [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A binary `.splat` file containing $N$ Gaussian primitives. Each primitive consists of a 3D center, 3D scale, a quaternion stored as 4 bytes (normalized by subtracting 128 and dividing by 128.0, with the z-component negated), and color/alpha data [S2, unofficial].
*   **Output:** A 2D screen image rendered via WebGL 1.0 `drawElements` or `drawArrays` with instanced rendering, using custom vertex and fragment shaders [S1, S2, unofficial].

**Data Structures:**
The implementation typically uses `InstancedBufferGeometry` in Three.js [S2, unofficial]. The layout per vertex in the raw `.splat` file is 32 bytes [S2, unofficial]:
1.  `center`: 3 `Float32` (12 bytes, offset 0).
2.  `scale`: 3 `Float32` (12 bytes, offset 12).
3.  `color/alpha`: 4 `Uint8` (R, G, B, A, offset 24).
4.  `quaternion`: 4 `Uint8` (x, y, z, w, offset 28) [S2, unofficial].

In the GPU buffers, these are unpacked into:
*   `splatColor`: `Float32Array` of size $4N$ (RGBA, normalized to [0,1]) [S2, unofficial].
*   `splatCenterCovariance`: `Float32Array` of size $9N$. The covariance matrix $\mathbf{V}_{rk}$ is stored as 3 `vec3`s: `[0]` is the center, `[1]` is $(M_{11}, M_{12}, M_{13})$, `[2]` is $(M_{22}, M_{23}, M_{33})$ [S2, unofficial].
*   `position`: `Float32Array` of size $18$ (6 components for 4 vertices, including duplicates for triangle rendering) defining a quad with corners $(2,2,0), (-2,2,0), (-2,-2,0), (2,-2,0)$ [S2, unofficial].

**Numbered Steps:**

1.  **Data Parsing and Matrix Construction:**
    Parse the `.splat` file. For each vertex $i$:
    *   Extract `center` from `f_buffer[8*i+0..2]`, `scale` from `f_buffer[8*i+3+0..2]`, `quaternion` from `u_buffer[32*i+28..31]` (normalized by subtracting 128 and dividing by 128.0, with z-component negated), and `color`/`alpha` from `u_buffer[32*i+24..27]` (divided by 255) [S2, unofficial].
    *   Construct a $4 \times 4$ instance matrix $\mathbf{M}_i$ [S2, unofficial]. Compute rotation matrix $\mathbf{R}$ from $q$ [S2, unofficial]. Scale by `scale` vector. The matrix is constructed as $\mathbf{M}_i = \mathbf{R}^T \cdot \text{Scale} \cdot \mathbf{R} \cdot \text{Translate}(\text{center})$ [S2, unofficial].
    *   Encode color/alpha into matrix elements 3, 7, 11, 15 [S2, unofficial].

2.  **Depth Sorting (CPU):**
    *   Calculate depth for each splat using the view vector $\mathbf{v}_{view}$ (camera forward direction from `camera.matrixWorld` elements 2, 6, 10) [S2, unofficial]:
        $$ \text{depth}_i = \mathbf{v}_{view}[0] \cdot \mathbf{M}_i[12] - \mathbf{v}_{view}[1] \cdot \mathbf{M}_i[13] - \mathbf{v}_{view}[2] \cdot \mathbf{M}_i[14] $$
    *   Perform a 16-bit single-pass counting sort to reorder indices based on `depth` [S2, unofficial]. Compute `depthInv = (65535)/(maxDepth-minDepth)`, `bin = ((depth-minDepth)*depthInv)|0`, count bins, compute starts, and place indices in `depthIndex` array [S2, unofficial].
    *   Reorder the instance matrices and attributes into `sortedMatrices` and `sortedAttributes` [S2, unofficial].
    *   This step is often offloaded to a Web Worker to prevent blocking the main thread [S2, unofficial].

3.  **Vertex Shader Execution:**
    For each instance (splat) and each vertex of the quad:
    *   **Frustum Culling:** Compute clip-space position $\mathbf{pos}_{2d} = \text{realProjectionMatrix} \cdot \text{viewMatrix} \cdot \text{vec4}(\text{splatCenter}, 1)$ [S2, unofficial]. Let $w = \mathbf{pos}_{2d}.w$. If $\mathbf{pos}_{2d}.z < -w$ or $|\mathbf{pos}_{2d}.x| > 1.2w$ or $|\mathbf{pos}_{2d}.y| > 1.2w$, discard the primitive by setting `gl_Position` to $(0,0,2,1)$ [S2, unofficial].
    *   **Covariance Projection:**
        *   Compute focal length: $focal = \frac{\text{size}.y}{2.0 \cdot \tan(\frac{\text{camera.fov} \cdot \pi}{360.0})}$ [S2, unofficial].
        *   Compute Jacobian $\mathbf{J}$ of the projection at the splat center $\mathbf{c}$ in camera space $\mathbf{c}_{cam}$:
            $$ \mathbf{J} = \begin{bmatrix} \frac{f_x}{c_z} & 0 & -\frac{f_x c_x}{c_z^2} \\ 0 & \frac{f_y}{c_z} & -\frac{f_y c_y}{c_z^2} \end{bmatrix} $$
        *   Compute transformation $\mathbf{T} = \mathbf{W} \mathbf{J}$, where $\mathbf{W}$ is the rotation part of the view matrix (transpose of the upper-left 3x3 of `viewMatrix`) [S2, unofficial].
        *   Compute 2D covariance $\mathbf{cov}_{2D} = \mathbf{T}^T \mathbf{V}_{rk} \mathbf{T}$ [S2, unofficial].
        *   Regularize: $\mathbf{cov}_{00} \leftarrow \mathbf{cov}_{00} + 0.3$, $\mathbf{cov}_{11} \leftarrow \mathbf{cov}_{11} + 0.3$ [S2, unofficial].
    *   **Eigenvalue Decomposition:**
        *   $diagonal1 = \mathbf{cov}_{00}$, $diagonal2 = \mathbf{cov}_{11}$, $offDiagonal = \mathbf{cov}_{01}$ [S2, unofficial].
        *   $mid = 0.5(diagonal1 + diagonal2)$ [S2, unofficial].
        *   $radius = \sqrt{(\frac{diagonal1 - diagonal2}{2})^2 + offDiagonal^2}$ [S2, unofficial].
        *   $\lambda_1 = mid + radius$ [S2, unofficial].
        *   $\lambda_2 = \max(mid - radius, 0.1)$ [S2, unofficial].
    *   **Basis Vectors:**
        *   $\mathbf{d} = \text{normalize}(offDiagonal, \lambda_1 - diagonal1)$ [S2, unofficial].
        *   $\mathbf{v}_1 = \min(\sqrt{2\lambda_1}, 1024.0) \cdot \mathbf{d}$ [S2, unofficial].
        *   $\mathbf{v}_2 = \min(\sqrt{2\lambda_2}, 1024.0) \cdot \text{vec2}(\mathbf{d}.y, -\mathbf{d}.x)$ [S2, unofficial].
    *   **Position Output:**
        *   $\mathbf{v}_{Center} = \frac{\mathbf{pos}_{2d}}{w}$ [S2, unofficial].
        *   $\mathbf{p}_{screen} = \mathbf{v}_{Center} + \frac{x \cdot \mathbf{v}_2 + y \cdot \mathbf{v}_1}{\text{viewport}} \cdot 2.0$ [S2, unofficial].
        *   `gl_Position` = $\text{vec4}(\mathbf{p}_{screen}, 0.0, 1.0)$ [S2, unofficial].

4.  **Fragment Shader Execution:**
    *   Compute distance from fragment center to splat center in local quad space: $A = -\mathbf{v}_{Position} \cdot \mathbf{v}_{Position}$ [S2, unofficial].
    *   If $A < -4.0$, discard fragment [S2, unofficial].
    *   Compute alpha: $B = e^A \cdot v_{Color}.a$ [S2, unofficial].
    *   Output color: $\text{gl\_FragColor} = \text{vec4}(B \cdot v_{Color}.rgb, B)$ [S2, unofficial].

5.  **Blending:**
    *   Render with `depthTest=false`, `depthWrite=false` [S2, unofficial].
    *   Blending function: `blendSrc=OneMinusDstAlphaFactor`, `blendDst=OneFactor`, `blendSrcAlpha=OneMinusDstAlphaFactor`, `blendDstAlpha=OneFactor` [S2, unofficial].

**Parameters:**
*   **WebGL Version:** 1.0 [S1, unofficial].
*   **Culling Bound:** $1.2 \cdot w$ [S2, unofficial].
*   **Covariance Regularization:** $0.3$ added to diagonal elements [S2, unofficial].
*   **Minimum Eigenvalue:** $0.1$ for $\lambda_2$ [S2, unofficial].
*   **Maximum Axis Length:** $1024.0$ for $\mathbf{v}_1, \mathbf{v}_2$ [S2, unofficial].
*   **Gaussian Cutoff:** Discard if squared distance $> 4.0$ [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Sorting:** Incorrect sorting order leads to visual artifacts due to non-commutative alpha blending. The sorting must be updated every frame as the camera moves [S2, unofficial].

## Subtopics

### [Isotropic Gaussian Point-Sprite Rendering](6949-isotropic-gaussian-point-sprite-rendering.md)
A simpler variant using isotropic point sprites with CPU-side back-to-front sorting to correct blending order, lacking anisotropic covariance and spherical harmonics.

## Operating characteristics
*   **Time Complexity:** Sorting is $O(N)$ with counting sort, but dominated by the constant factor of memory access and worker communication [S2, unofficial]. Rendering is $O(N)$ for draw calls [S2, unofficial].
*   **Memory Complexity:** $O(N)$ for storing splat data and sorted indices [S2, unofficial].
*   **Performance:** Real-time on modern desktop GPUs via WebGL 1.0 [S2, unofficial]. Performance scales with the number of splats $N$ and viewport resolution [S2, unofficial].
*   **Strengths:** No external dependencies beyond Three.js; works on standard web browsers; efficient on ordinary graphics hardware compared to NeRFs [S1, unofficial].
*   **Limitations:** No view-dependent shading (spherical harmonics omitted for file size); limited to WebGL 1.0 features; sorting on CPU can block main thread if not offloaded [S1, S2, unofficial].
*   **When to use:** Web-based visualization of static or slowly moving 3DGS scenes [S1, S2, unofficial].
*   **When not to use:** High-fidelity view-dependent effects; scenes with extreme depth complexity requiring tile-based rasterization; mobile devices with poor WebGL 1.0 support [S1, S2, unofficial].

## Background
3D Gaussian Splatting was introduced by Kerbl et al. in 2023 as a real-time radiance field rendering technique [S1, unofficial]. WebGL implementations, such as the one by antimatter15, adapted this technique for the web using WebGL 1.0 and Three.js, focusing on efficient point-cloud-like rendering without neural network inference [S1, unofficial]. The method was developed to allow interactive viewing of 3DGS models in browsers without heavy dependencies [S1, unofficial].

## Sources
[S1] antimatter15/splat, https://github.com/antimatter15/splat, fetched 2026-10-27 (unofficial)
[S2] akbartus/Gaussian-Splatting-WebViewers, https://github.com/akbartus/Gaussian-Splatting-WebViewers, fetched 2026-10-27 (unofficial)
