# EWA Splatting

Also called: Elliptical Weighted Average Splatting

## Overview
EWA Splatting is an object-order rendering technique that approximates the volumetric integration of 3D Gaussian primitives by projecting them into 2D elliptical disks (splats) and rasterizing them as screen-space filters [S1]. It solves the computational cost of ray-marching by assuming Gaussians do not overlap and are sorted by view-space depth, trading continuous extinction accumulation for efficient, hole-free reconstruction [S1][S2, unofficial]. It differs from standard point rendering by using anisotropic, band-limited elliptical footprints derived from the Gaussian covariance, whereas simplified variants use screen-aligned spherical impostors with isotropic falloff [S2, S3, unofficial].

## How it works
**Inputs:** A set of $N$ 3D Gaussians, each defined by weight $w_i$, mean position $\boldsymbol{\mu}_i$ (in camera space), and $3 \times 3$ covariance matrix $\boldsymbol{\Sigma}_i$ [G]. Camera projection matrix $\mathbf{P}$ and view matrix $\mathbf{W}$ [G].
**Outputs:** A 2D image with accumulated color and depth [G].

**Data Structures:**
*   **Surfel vertex buffer layout** (stride 64 bytes) [S2, unofficial]:
    *   Offset 0: `center` (vec3, 3 floats) [S2, unofficial]
    *   Offset 12: `tangent_u` (vec3, 3 floats) [S2, unofficial]
    *   Offset 24: `tangent_v` (vec3, 3 floats) [S2, unofficial]
    *   Offset 36: `clipping_plane` (vec3, 3 floats) [S2, unofficial]
    *   Offset 48: `color_rgba` (vec4, 4 unsigned bytes) [S2, unofficial]
*   **Uniforms:**
    *   `UniformBufferParameter`: `color` (vec3, offset 0), `shininess` (float, offset 12), `radius_scale` (float, offset 16), `ewa_radius` (float, offset 20), `epsilon` (float, offset 24). Total 8 floats [S2, unofficial].
    *   `UniformBufferFrustum`: 6 frustum planes (vec4) [S2, unofficial].
    *   `UniformBufferRaycast`: `projection_matrix_inv` (mat4), `viewport` (vec4) [S2, unofficial].

**Steps:**

1.  **Frustum Culling:** Extract 6 frustum planes from the projection matrix. A splat is visible if its bounding sphere intersects the view frustum [S2, unofficial].
    ```python
    def extract_frustum_planes(P):
        frustum_planes = []
        for i in range(6):
            sign = -1.0 if i % 2 == 0 else 1.0
            row_idx = i // 2
            plane = [P[3][j] + sign * P[row_idx][j] for j in range(4)]
            norm = (plane[0]**2 + plane[1]**2 + plane[2]**2)**0.5
            plane = [x / norm for x in plane]
            frustum_planes.append(plane)
        return frustum_planes
    ```

2.  **2D Gaussian Projection (EWA):** Project the 3D Gaussian to 2D screen space to determine the splat's shape and position [S1].
    *   Compute the Jacobian $\mathbf{J}$ of the perspective projection, approximated by first-order Taylor expansion at the Gaussian mean $\boldsymbol{\mu}$ [S1]:
        $$ \mathbf{J} = \begin{pmatrix} 1/\mu_2 & 0 & -\mu_0/\mu_2^2 \\ 0 & 1/\mu_2 & -\mu_1/\mu_2^2 \\ \mu_0/l & \mu_1/l & \mu_2/l \end{pmatrix} $$
        where $l = \|\boldsymbol{\mu}\|_2$ and $\boldsymbol{\mu} = (\mu_0, \mu_1, \mu_2)$ is the mean in camera space [S1].
    *   Compute the projected 2D covariance $\boldsymbol{\Sigma}'$ [S1]:
        $$ \boldsymbol{\Sigma}' = \mathbf{J} \mathbf{W} \boldsymbol{\Sigma} \mathbf{W}^T \mathbf{J}^T $$
        where $\mathbf{W}$ transforms to camera space [S1].
    *   The projected 2D Gaussian is $\mathcal{G}_2^n(\mathbf{p}, w_i, \boldsymbol{\mu}'_i, \boldsymbol{\Sigma}'_i)$ [S1]. The weight $w$ is preserved across projection [S1].

3.  **Sorting:** Sort all visible Gaussians by their view-space depth (z-coordinate of $\boldsymbol{\mu}_i$) from front to back [S1].

4.  **Rasterization (Soft Z-Buffer Passes):** Standard EWA applies a bandlimiting prefilter to avoid frequencies higher than the Nyquist frequency [S2, unofficial]. A common GPU implementation uses a three-pass soft z-buffer approach with spherical impostors [S3, unofficial].
    *   **Bandlimiting Prefilter:** Generate a 1D filter kernel texture using $\sigma = 0.316228$ to approximate the prefilter [S2, unofficial].
        ```python
        def generate_filter_kernel():
            sigma = 0.316228
            kernel = []
            for i in range(256):
                x = i / 255.0
                w = (x * x) / (2.0 * sigma)
                kernel.append(math.exp(-w))
            return kernel
        ```
    *   **Pass 1 (Visibility/Depth):** Bind FBO, clear color and depth buffers [S2, unofficial]. Render splats as point sprites [S3, unofficial]. Set `gl_PointSize` based on the projected radius $r$ [S3, unofficial]:
        $$ s = h \cdot \cot\left(\frac{\theta_{fov}}{2}\right) \cdot \frac{r}{w_{clip}} $$
        where $h$ is viewport height, $\theta_{fov}$ is FOV, and $w_{clip}$ is clip-space w [S3, unofficial]. Compute analytic front-surface depth of the sphere impostor for each fragment [S3, unofficial]:
        $$ z_{surface} = z_{center} - \sqrt{r^2 - x^2 - y^2} $$
        where $x, y$ are fragment coordinates relative to splat center [S3, unofficial]. Push depth back by $\varepsilon$ before writing to define a thin depth slab [S3, unofficial]. Discard fragments outside the unit disk [S3, unofficial]. Write only depth; mask off color [S3, unofficial].
    *   **Pass 2 (Attribute Blending):** Disable depth writes and depth testing [S3, unofficial]. Set blending to pure additive (`GL_ONE`, `GL_ONE`) [S3, unofficial]. Rasterize splats again [S3, unofficial]. For each fragment, compare its depth against the depth texture from Pass 1. Discard fragments falling behind the $\varepsilon$ slab [S3, unofficial]. Accumulate attributes with weight $w = w_{radial} \cdot w_{depth}$ [S3, unofficial]:
        *   Radial falloff: $w_{radial} = 1 - \frac{d}{r}$, where $d$ is distance from splat center [S3, unofficial].
        *   Depth falloff: $w_{depth} = \frac{z_{surface} - z_{frag}}{\varepsilon}$, where $z_{frag}$ is current fragment depth [S3, unofficial].
    *   **Pass 3 (Normalization/Shading):** Render a full-screen quad [S3, unofficial]. Compute weighted average color and normal [S3, unofficial]:
        $$ \mathbf{c} = \frac{\sum_i w_i \mathbf{c}_i}{\sum_i w_i}, \quad \mathbf{n} = \frac{\sum_i w_i \mathbf{n}_i}{\lVert \sum_i w_i \mathbf{n}_i \rVert} $$
        Apply Phong lighting [S3, unofficial].

**Parameters:**
*   `m_epsilon`: Epsilon value for numerical stability, default $1.0 \times 10^{-3}$ [S2, unofficial]. Defines the thickness of the thin depth slab around the visible surface [S3, unofficial].
*   `m_radius_scale`: Scale factor for splat radius, default $1.0$ [S2, unofficial].
*   `m_ewa_radius`: Radius for EWA filter approximation, default $1.0$ [S2, unofficial].
*   `sigma`: Gaussian filter kernel parameter, $0.316228$ [S2, unofficial].
*   `m_shininess`: Shininess value for Phong splatting, default $8.0$ [S2, unofficial].
*   `m_smooth`: Boolean flag that enables depth and normal texture attachment on the FBO, and enables blending/smoothing in the finalization pass [S2, unofficial].
*   `m_multisample`: Boolean flag that enables multisampling [S2, unofficial].

**Pitfalls:**
*   Botsch et al.'s approximation of screen-space extents can underestimate splat size, causing chopping artifacts; a safety factor $>1$ is required [S2, unofficial].
*   Zwicker et al.'s centralized conics approach is numerically unstable when projection approaches a line [S2, unofficial].
*   Weyrich et al.'s method yields NDC coordinates, preventing frustum culling, and fails if the projected ellipse becomes a parabola or hyperbola [S2, unofficial].
*   EWA filter requires soft z-buffer to be active; otherwise, it is disabled [S2, unofficial].
*   Depth mask is explicitly disabled during splatting when soft z-buffer is enabled to allow alpha blending to accumulate depth correctly [S2, unofficial].

## Subtopics

### [Deferred Surface Splatting](1072-deferred-surface-splatting.md)
Converts point clouds to continuous shaded surfaces via G-buffers and deferred shading, differing from standard splatting by deferring lighting calculations to a final pass [1072-deferred-surface-splatting.md]. Choose it when surface normals and deferred shading are required [1072-deferred-surface-splatting.md].

### [Opacity-Thin-Side (OTS)](1155-opacity-thin-side-ots.md)
A parameterization scheme for EWA-based splatting that scales the learned weight $\theta$ so the projected amplitude equals $\theta$ at the thinnest view, differing from standard EWA by ensuring robust gradient-descent optimization [1155-opacity-thin-side-ots.md]. Choose it when training 3D Gaussians requires stable opacity gradients [1155-opacity-thin-side-ots.md].

### [GPU-Accelerated Volume Splatting With Elliptical RBFs](1186-gpu-accelerated-volume-splatting-with-elliptical-rbfs.md)
Extends EWA to irregular grids using elliptical Radial Basis Functions and sheet buffers, differing from standard EWA by supporting volume rendering on non-uniform structures [1186-gpu-accelerated-volume-splatting-with-elliptical-rbfs.md]. Choose it for GPU-accelerated volume rendering on irregular point distributions [1186-gpu-accelerated-volume-splatting-with-elliptical-rbfs.md].

### [Ellipsoidal Kernel Splatting with Warped Slicing](1290-ellipsoidal-kernel-splatting-with-warped-slicing.md)
Represents points as 3D ellipsoids and warps slicing planes to unit sphere coordinates for efficient GPU rasterization, differing from standard EWA by avoiding expensive square roots and conditional statements during ellipsoid slicing [1290-ellipsoidal-kernel-splatting-with-warped-slicing.md]. Choose it when rendering dense 3D ellipsoidal kernels with hard-wired GPU texture mapping [1290-ellipsoidal-kernel-splatting-with-warped-slicing.md].

## Operating characteristics
*   **Time Complexity:** Not given in the sources [S2, unofficial].
*   **Memory Complexity:** Not given in the sources [S2, unofficial].
*   **Strengths:** Hole-free reconstruction, efficient rasterization, avoids expensive volume integration [S2, unofficial].
*   **Limitations:** Assumes no overlap; continuous extinction is ignored [S1]. Numerical instability when projection approaches a line, or failure if projected ellipse becomes a parabola or hyperbola [S2, unofficial].
*   **When to use:** Real-time rendering of point clouds or Gaussian splats where efficient rasterization and hole-free reconstruction are prioritized over physical accuracy of attenuation [S1][S2, unofficial].

## Background
EWA Splatting originated from the need to render point-sampled surfaces without aliasing or holes [S2, unofficial]. It was formalized by Zwicker et al. (2001) as "Surface Splatting," using Elliptical Weighted Average (EWA) filters to project object-space reconstruction filters to screen space [S2, unofficial]. It builds on [Gaussian Splatting](946-gaussian-splatting.md) by providing the rasterization backend for 3D Gaussian primitives, approximating the volumetric integral via 2D projections [S1].

## Sources
[S1] Does 3D Gaussian Splatting Need Accurate Volumetric Rendering?, https://doi.org/10.1111/cgf.70032, fetched 2026-10-26
[S2] sebastianlipponer/surface_splatting, https://github.com/sebastianlipponer/surface_splatting, fetched 2026-10-28 (unofficial)
[S3] arminkz/PointCloudRendering, https://github.com/arminkz/PointCloudRendering, fetched 2026-10-28 (unofficial)
