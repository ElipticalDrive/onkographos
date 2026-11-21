# 3D Gaussian Splatting Rendering (CPU-Sorted, Instanced)

Also called: 3DGS Weekend Renderer, Gaussian Splatting Instanced Draw, RViz Splat Cloud, SplatCloud Instanced Rendering

## Overview
This method renders trained 3D Gaussian Splatting (3DGS) scenes by projecting 3D Gaussian distributions into 2D screen-space quads, sorting them by camera-space depth on the CPU, and rendering them using hardware-instanced drawing with standard alpha blending [S1, S2, S3, unofficial]. It solves the problem of visualizing large-scale, photorealistic 3D scenes on client-side hardware (mobile, web, or desktop) without requiring neural network inference or server-side ray tracing [S1, S3, unofficial]. It sets itself apart from its siblings by performing the critical depth sorting on the CPU, which allows the GPU to focus solely on rasterization and alpha-blending, avoiding the complexity of GPU-based sorting or tile-based cooperative execution [S2, S3, unofficial]. Unlike [Tiled 2D Gaussian Splatting Rasterization](2061-tiled-2d-gaussian-splatting-rasterization.md), this method relies on explicit per-splat sorting rather than tile-based accumulation [S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of $N$ 3D Gaussians defined by mean position $\boldsymbol{\mu}$, covariance $\boldsymbol{\Sigma}_{3D}$ (derived from rotation $q$ and scale $s$), opacity $\alpha$, and spherical harmonic (SH) coefficients for color.
*   **Output:** A 2D image with color and alpha channels.

**Data Structures:**
1.  **Splat Data:** Stored in two square RGBA16F textures (or Texture Buffer Objects) to bypass uniform array limits [S5, S6, unofficial].
    *   `splatPositionScale`: Layout `xyz` = center position (model space), `w` = billboard half-extent (scale) [S5, S6, unofficial].
    *   `splatColorOpacity`: Layout `rgb` = SH0 color (linear), `a` = straight opacity [S5, S6, unofficial].
    *   Sampling must use `NEAREST` filtering and `CLAMP_TO_EDGE` wrapping to prevent blending adjacent splat attributes [S5, S6, unofficial].
2.  **Sort Indices:** An array of $N$ integers representing the sorted order of splats. For very large $N$ exceeding the 16-bit index buffer limit (65,535), the cloud is split into batches [S5, S6, unofficial].
3.  **Geometry:** A shared unit quad with corners at $(-1,-1), (1,-1), (-1,1), (1,1)$ and 6 indices forming two triangles [S1, S5, S6, unofficial].

**Steps:**

1.  **Parameter Activation:** Transform raw training parameters into geometric primitives [Parent].
    *   Normalize rotation: $q_{norm} = q / \|q\|$.
    *   Exponentiate scale: $s_{eff} = \exp(s)$.
    *   Sigmoid opacity: $\alpha_{eff} = \sigma(\alpha)$.
    *   Construct 3D covariance: $\boldsymbol{\Sigma}_{3D} = \mathbf{R} \mathbf{S} \mathbf{S}^T \mathbf{R}^T$, where $\mathbf{R}$ is derived from $q_{norm}$ and $\mathbf{S} = \text{diag}(s_{eff})$ [Parent].

2.  **Camera Space Transformation:**
    *   Transform mean to camera space: $\boldsymbol{\mu}' = \mathbf{V} \boldsymbol{\mu}$ [Parent].
    *   Transform covariance: $\boldsymbol{\Sigma}' = \mathbf{V}_{1:3, 1:3} \boldsymbol{\Sigma}_{3D} \mathbf{V}_{1:3, 1:3}^T$ [Parent].

3.  **Screen-Space Projection:**
    *   Compute Jacobian $\mathbf{J}$ at $\boldsymbol{\mu}'$:
        $$ \mathbf{J} = \begin{pmatrix} \frac{f_x}{\mu'_z} & 0 & -\frac{f_x \mu'_x}{\mu'_z^2} \\ 0 & \frac{f_y}{\mu'_z} & -\frac{f_y \mu'_y}{\mu'_z^2} \end{pmatrix} $$
        [Parent].
    *   Compute 2D covariance: $\boldsymbol{\Sigma}_{2D} = \mathbf{J} \boldsymbol{\Sigma}' \mathbf{J}^T$ [Parent].
    *   Compute 2D mean: $\boldsymbol{\mu}_{2D} = \begin{pmatrix} \frac{f_x \mu'_x}{\mu'_z} + c_x \\ \frac{f_y \mu'_y}{\mu'_z} + c_y \end{pmatrix}$ [Parent].

4.  **CPU Sorting:**
    *   Compute camera-space depth $z$ for each splat centroid.
    *   Sort splats by increasing camera-space $z$ (far-to-near order, as camera looks down negative Z) [S1, S3, unofficial].
    *   **Optimization:** Re-sorting is triggered only when camera movement exceeds a threshold (e.g., 1% of bounding radius or dot product of view direction change > 0.01) to avoid redundant computation [S5, S6, unofficial].
    *   **Batching:** If $N > 65,535$, split the sorted indices into batches of size $\le 65,535$ [S5, S6, unofficial].

5.  **Vertex Shader Execution (Per Instance):**
    *   Fetch splat attributes from textures using `gl_InstanceID` [S5, S6, unofficial].
    *   Compute clip space position: `clip = projection * view * vec4(splatCentroid, 1.0)` [S1, unofficial].
    *   Compute camera space centroid: `centerCamera = vec3(view * vec4(splatCentroid, 1.0))` [S1, unofficial].
    *   Project 3D covariance to 2D: `covariance2D = projectCovarianceToScreen(centerCamera, buildCovariance3D())` [S1, unofficial].
    *   **Eigen-decomposition:** Compute eigenvalues $\lambda_1, \lambda_2$ and eigenvectors $e_1, e_2$ of $\boldsymbol{\Sigma}_{2D}$ [S1, unofficial].
        *   Add jitter to diagonal elements: $a = \Sigma_{2D}[0][0] + 0.3$, $d = \Sigma_{2D}[1][1] + 0.3$ [S1, unofficial].
        *   Clamp eigenvalues: $\lambda_{1,2} = \max(\dots, 0.01)$ [S1, unofficial].
    *   Compute 3-sigma principal half-axes in pixels: $b_1 = 3.0 \sqrt{\lambda_1} e_1$, $b_2 = 3.0 \sqrt{\lambda_2} e_2$ [S1, unofficial].
    *   Compute vertex offset in NDC: `deltaNdc = (quadCorner.x * b1 + quadCorner.y * b2) / (viewportSize * 0.5)` [S1, unofficial].
    *   Set final position: `gl_Position = vec4(ndc.xy + deltaNdc, ndc.z, 1.0)` [S1, unofficial].
    *   Pass varyings: `splatCoord = quadCorner`, `splatColor = evaluateSphericalHarmonics()`, `splatOpacity = 1.0 / (1.0 + exp(-opacity))` [S1, unofficial].

6.  **Fragment Shader Execution (Per Pixel):**
    *   Compute power term: `power = -4.5 * dot(splatCoord, splatCoord)` [S1, unofficial].
    *   Discard if `power < -9.0` (approx $e^{-9} \approx 0$) [S1, unofficial].
    *   Compute final alpha: `alpha = min(0.99, splatOpacity * exp(power))` [S1, unofficial].
    *   Discard if `alpha < 0.001` [S1, unofficial].
    *   Output color: `FragColor = vec4(splatColor, alpha)` [S1, unofficial].

7.  **Alpha Compositing:**
    *   Disable depth test: `glDisable(GL_DEPTH_TEST)` [S1, unofficial].
    *   Enable premultiplied-alpha blending: `glEnable(GL_BLEND); glBlendFunc(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA)` [S1, unofficial].
    *   Draw instanced quads: `glDrawArraysInstanced(GL_TRIANGLE_STRIP, 0, 4, instanceCount)` [S1, unofficial].
    *   For batches, use `blendOrder` if instance-order violations occur [S5, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_cpu_sorted_instanced(gaussians, camera, viewport_size):
    """
    Render 3D Gaussians using CPU sorting and instanced drawing.
    gaussians: List of Gaussian objects with means, covariances, opacities, sh_coeffs.
    camera: Camera object with view, projection, and intrinsics.
    viewport_size: (width, height) in pixels.
    """
    # 1. Transform to Camera Space
    means_cam = []
    covariances_cam = []
    for g in gaussians:
        # Transform mean
        mu_cam = camera.transform_world_to_cam(g.mean)
        means_cam.append(mu_cam)
        
        # Transform covariance
        # R and S are derived from g.rotation and g.scale (Parent)
        R = rotation_matrix_from_quaternion(g.rotation)
        S = diag_matrix(exp(g.scale))
        Sigma_3D = R @ S @ S.T @ R.T
        V_3x3 = camera.view_matrix[:3, :3]
        Sigma_cam = V_3x3 @ Sigma_3D @ V_3x3.T
        covariances_cam.append(Sigma_cam)

    # 2. Project to Screen Space
    means_2d = []
    covariances_2d = []
    depths = []
    
    for i, g in enumerate(gaussians):
        mu_c = means_cam[i]
        Sigma_c = covariances_cam[i]
        
        # Frustum Culling (Optional but recommended)
        if not camera.is_in_frustum(mu_c):
            continue
            
        # Project Mean
        mu_2d = camera.project_cam_to_image(mu_c)
        means_2d.append(mu_2d)
        depths.append(mu_c[2]) # Camera Z
        
        # Project Covariance (EWA Approximation)
        J = compute_jacobian(mu_c, camera.focal_length)
        Sigma_2d = J @ Sigma_c @ J.T
        
        # Add jitter to prevent singularities
        Sigma_2d[0,0] += 0.3
        Sigma_2d[1,1] += 0.3
        
        covariances_2d.append(Sigma_2d)

    # 3. CPU Sorting
    # Create indices and sort by depth (far to near, i.e., increasing Z in camera space)
    indices = list(range(len(gaussians)))
    indices.sort(key=lambda i: depths[i])
    
    # 4. Prepare Data for GPU
    # Upload sorted data to textures or buffers
    # Split into batches if N > 65535
    batch_size = 65535
    batches = [indices[i:i+batch_size] for i in range(0, len(indices), batch_size)]
    
    # 5. Render
    glDisable(GL_DEPTH_TEST)
    glEnable(GL_BLEND)
    glBlendFunc(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA)
    
    for batch in batches:
        # Upload batch indices and attributes to GPU buffers/textures
        upload_splat_data(batch, gaussians, means_2d, covariances_2d)
        
        # Draw Instanced Quads
        glDrawArraysInstanced(GL_TRIANGLE_STRIP, 0, 4, len(batch))

def compute_jacobian(mu_cam, fx, fy):
    """Compute Jacobian of projective projection."""
    z = mu_cam[2]
    x = mu_cam[0]
    y = mu_cam[1]
    return np.array([
        [fx / z, 0, -fx * x / (z * z)],
        [0, fy / z, -fy * y / (z * z)]
    ])

def rotation_matrix_from_quaternion(q):
    """Convert quaternion to rotation matrix."""
    # Standard quaternion-to-matrix conversion
    x, y, z, w = q
    return np.array([
        [1 - 2*(y*y + z*z), 2*(x*y - z*w), 2*(x*z + y*w)],
        [2*(x*y + z*w), 1 - 2*(x*x + z*z), 2*(y*z - x*w)],
        [2*(x*z - y*w), 2*(y*z + x*w), 1 - 2*(x*x + y*y)]
    ])
```

## Subtopics

### [Tiled 2D Gaussian Splatting Rasterization](2061-tiled-2d-gaussian-splatting-rasterization.md)
Tiled 2D Gaussian Splatting Rasterization is a GPU-parallel rendering method for 3D Gaussian Splatting (3DGS) that partitions the image plane into macro-tiles to manage memory and processing load, assigning Gaussians to tiles based on their projected axis-aligned bounding boxes. It solves the performance bottleneck of per-pixel sorting by using a global radix sort to bin splats by tile and sort them by depth simultaneously, enabling real-time performance for millions of splats. It differs from its parent [3D Gaussian Splatting Rendering (CPU-Sorted, Instanced)](2058-3d-gaussian-splatting-rendering-cpu-sorted-instanced.md) by performing the sorting and pixel-level alpha blending on the GPU within thread blocks, rather than on the CPU [G]. It differs from [Gaussian Splat Rendering (Screen-Space Quad)](2075-gaussian-splat-rendering-screen-space-quad.md) by using a tile-based cooperative execution model with shared memory rather than standard screen-space quad primitives [G].

### [Gaussian Splat Rendering (Screen-Space Quad)](2075-gaussian-splat-rendering-screen-space-quad.md)
Gaussian Splat Rendering (Screen-Space Quad) is a display-only rendering technique that projects 3D Gaussian distributions into 2D screen-space quads, sorts them by camera-space depth, and renders them using instanced drawing with standard alpha blending. It solves the problem of visualizing large-scale, photorealistic 3D scenes reconstructed via Gaussian Splatting on client-side hardware without requiring neural network inference or server-side ray tracing. It sets itself apart from its siblings by utilizing a CPU-side depth sorting algorithm (often offloaded to a background worker) to resolve the non-commutative nature of alpha blending, whereas the parent [3D Gaussian Splatting Rendering (CPU-Sorted, Instanced)](2058-3d-gaussian-splatting-rendering-cpu-sorted-instanced.md) describes the general projection and quad-drawing mechanism without specifying the sorting implementation. Unlike [Tiled 2D Gaussian Splatting Rasterization](2061-tiled-2d-gaussian-splatting-rasterization.md), this method relies on explicit per-splat sorting rather than tile-based accumulation.

### [Depth-First Splat Sorting](2850-depth-first-splat-sorting.md)
Depth-First Splat Sorting is a preprocessing step for 3D Gaussian Splatting rendering that sorts visible Gaussian splats by camera-space depth before they are expanded into per-tile fragments. It solves the problem of high sorting overhead inherent in the original 3DGS pipeline, where fragments (one per tile a splat touches) are sorted using a combined 48-bit key. By sorting the smaller set of visible splats first, this method reduces the number of items to sort by a factor of 1.5 to 4, significantly lowering computational cost while maintaining correct depth ordering for subsequent rasterization.

### [Splatapult Gaussian Splat Renderer](6749-splatapult-gaussian-splat-renderer.md)
A renderer for 3D Gaussian Splatting (3DGS) files that supports VR, camera navigation, and optional spherical harmonic (SH) rendering, using either a C++ radix sort or a GPU-based multi-radix sort for depth sorting. It decodes a 3D Gaussian Splat radiance-field cloud from a PLY file and renders it as hardware-instanced, camera-facing quads on-device. It distinguishes itself from general-purpose GPU-sorted renderers by offloading the depth-sorting of splats to a background CPU thread at a configurable rate, allowing the GPU shader to focus solely on rasterization and alpha-blending. This approach trades off sorting latency for reduced GPU overhead, making it suitable for mobile XR hardware where computational budgets are shared with other system tasks.

### [SortFree Weighting](8006-sortfree-weighting.md)
SortFree Weighting is an Order-Independent Transparency (OIT) approximation for 3D Gaussian Splatting that eliminates the computational bottleneck of depth sorting by replacing non-commutative alpha blending with a commutative weighted-sum blending scheme. Unlike standard 3D Gaussian Splatting which requires CPU-side depth sorting of splats per pixel, SortFree allows parallel accumulation of Gaussian colors in a single pass, trading strict physical accuracy for rendering efficiency and temporal stability. It sets itself apart from siblings like [Depth-aware Order-independent Rendering](9460-depth-aware-order-independent-rendering.md) by using a fixed analytical depth falloff rather than neural networks to predict view-dependent weights.

### [Unity Mobile Implementation](9536-unity-mobile-implementation.md)
This method implements a display-only renderer for trained 3D Gaussian Splatting scenes within the Unity engine, optimized for mobile compatibility. It solves the problem of rendering high-fidelity Gaussian avatars on constrained hardware by relying exclusively on a warped canonical mesh for rigging and using a standard front-to-back alpha blending pipeline. It sets itself apart from its siblings by prioritizing maximum compatibility with existing game engine workflows (Mixamo rigging) and mobile stability over the raw performance optimizations found in tiled or sort-free variants.

### [CPU-based Gaussian Splatting Thumbnailer](14038-cpu-based-gaussian-splatting-thumbnailer.md)
This method generates low-resolution preview images (thumbnails) from trained 3D Gaussian Splatting scenes using a pure CPU pipeline optimized for speed via Just-In-Time (JIT) compilation. It solves the problem of slow interactive preview generation by simplifying the rendering model—omitting Gaussian rotation and complex shading—and leveraging Numba to accelerate Python-based numerical loops. It differs from its parent, [3D Gaussian Splatting Rendering (CPU-Sorted, Instanced)](2058-3d-gaussian-splatting-rendering-cpu-sorted-instanced.md), by explicitly excluding rotation data and using a simplified, approximate blending function rather than full alpha compositing, prioritizing execution time over visual fidelity.

### [Support Guard](17686-support-guard.md)
The Support Guard is a fault-mitigation technique for 3D Gaussian Splatting (3DGS) that prevents catastrophic rendering errors caused by Single-Event Upsets (SEUs), such as bit flips in memory, by clamping primitive parameters to the valid range observed during training. It solves the problem of "frame-spanning corruption," where a single bit flip in a scale parameter can exponentially inflate a primitive into a translucent sheet covering most of the screen. Unlike activation range supervision used in CNNs, which clamps neuron outputs to statistical distributions, the Support Guard clamps geometric and appearance parameters to the bounded parameter space of the optimized scene, providing a geometric correctness argument. It is distinct from its siblings in that it requires zero additional memory overhead and operates as a pure per-primitive clamp, making it suitable for deployment on spaceborne, robotic, and clustered hardware where reliability is critical.

## Operating characteristics
*   **Time Complexity:** Sorting is $O(N \log N)$ on the CPU. Rendering is $O(N)$ for the vertex shader and $O(P)$ for the fragment shader, where $P$ is the number of pixels covered by splats.
*   **Memory Complexity:** $O(N)$ for storing splat data and sorted indices. Textures require ~28 bytes/splat for SH0, ~14 MB VRAM for 500k splats [S5, unofficial].
*   **Performance:** CPU radix sort of ~500k depth keys takes a few milliseconds on a worker thread [S5, unofficial]. Re-sorting is gated on camera movement to avoid per-frame overhead [S5, unofficial].
*   **Strengths:** Simple implementation, works on hardware without compute shaders (e.g., older mobile GPUs, WebGL2), leverages standard alpha blending.
*   **Limitations:** Less efficient than GPU-sorted tile-based methods for very large scenes ($>1$ million splats) due to CPU sorting overhead and lack of early ray termination [S1, S5, unofficial].
*   **When to Use:** Mobile/web rendering, scenes with moderate splat counts, or when GPU compute capabilities are limited.
*   **When Not to Use:** Real-time rendering of massive scenes ($>1$M splats) where GPU sorting and tile-based rasterization are required for performance.

## Background
Originating from the paper "3D Gaussian Splatting for Real-Time Radiance Field Rendering" by Kerbl et al. (2023) [S1, unofficial], this rendering implementation was developed to provide a display-only renderer for trained 3DGS scenes. It was notably implemented in the Filament engine (Android/Web) and RViz (ROS) to enable visualization of Gaussian Splatting clouds on client-side hardware [S2, S5, unofficial]. The method addresses the limitation of the original CUDA-based rasterizer by providing a CPU-sorted, instanced approach suitable for broader graphics APIs [S1, S5, unofficial].

## Sources
[S1] 3D Gaussian Splatting in a Weekend, https://bfeldman.me/3dgs-weekend/, fetched 2026-10-26 (unofficial)
[S2] splat-ros/RVizSplat, https://github.com/splat-ros/RVizSplat, fetched 2026-10-27 (unofficial)
[S3] akbartus/Gaussian-Splatting-WebViewers, https://github.com/akbartus/Gaussian-Splatting-WebViewers, fetched 2026-10-27 (unofficial)
[S4] sascha-kirch/splaty, https://github.com/sascha-kirch/splaty, fetched 2026-10-27 (unofficial)
[S5] feat(sceneview): SplatNode — 3D Gaussian Splatting rendering (Android Filament custom material; iOS RealityKit native; web follows), https://github.com/sceneview/sceneview/issues/2646, fetched 2026-10-27 (unofficial)
[S6] feat(sceneview): SplatNode P1 — static 3D Gaussian Splat rendering ( #…, https://github.com/sceneview/sceneview/commit/9dc024089729c3822a48893901c09dbb0c1a04d4, fetched 2026-10-29 (unofficial)
