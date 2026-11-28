# 3D Gaussian Splatting Rendering (CPU-Sorted, Instanced)

Also called: 3DGS Weekend Renderer, Gaussian Splatting Instanced Draw, RViz Splat Cloud, SplatCloud Instanced Rendering

## Overview
This method renders trained 3D Gaussian Splatting scenes by projecting 3D Gaussian primitives into 2D screen-space quads, sorting them by camera-space depth on the CPU, and drawing them using hardware-instanced rendering with standard alpha blending [S1, S2, S3, unofficial]. It solves the problem of visualizing large-scale, photorealistic 3D scenes on client-side hardware (such as mobile devices or web browsers) without requiring neural network inference or server-side ray tracing [S1, S3, unofficial]. It sets itself apart from its sibling [Tiled 2D Gaussian Splatting Rasterization](2061-tiled-2d-gaussian-splatting-rasterization.md) by moving the sorting and blending logic entirely to the CPU and using standard fixed-function alpha blending rather than tile-based GPU accumulation, and from [Gaussian Splat Rendering (Screen-Space Quad)](2075-gaussian-splat-rendering-screen-space-quad.md) by explicitly defining the instanced draw call structure and texture-based attribute fetching required for large splat counts [S2, S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of $N$ 3D Gaussians defined by mean position $\boldsymbol{\mu}$, rotation $q$, scale $s$, opacity $\alpha$, and spherical harmonic (SH) coefficients.
*   **Output:** A 2D image with color $C$ and alpha $\alpha$.

**Data Structures:**
1.  **Gaussian Primitive:** Defined by parameters transformed into effective values [2812-3d-gaussian-splatting.md][2058-3d-gaussian-splatting-rendering-cpu-sorted-instanced.md]:
    *   Rotation: $q_{\text{norm}} = q / \|q\|$ [2812-3d-gaussian-splatting.md][2058-3d-gaussian-splatting-rendering-cpu-sorted-instanced.md].
    *   Scale: $s_{\text{eff}} = \exp(s)$ [2812-3d-gaussian-splatting.md][2058-3d-gaussian-splatting-rendering-cpu-sorted-instanced.md].
    *   Opacity: $\alpha_{\text{eff}} = \sigma(\alpha)$ [2812-3d-gaussian-splatting.md][2058-3d-gaussian-splatting-rendering-cpu-sorted-instanced.md].
    *   3D Covariance: $\boldsymbol{\Sigma}_{3D} = \mathbf{R} \mathbf{S} \mathbf{S}^T \mathbf{R}^T$, where $\mathbf{R}$ is derived from $q_{\text{norm}}$ and $\mathbf{S} = \text{diag}(s_{\text{eff}})$ [2812-3d-gaussian-splatting.md][2058-3d-gaussian-splatting-rendering-cpu-sorted-instanced.md].
2.  **GPU Buffers (Instanced Rendering):**
    *   **Quad VBO:** Static buffer containing 4 vertices of a unit quad with corners at $(-1,-1), (1,-1), (-1,1), (1,1)$ [S1, S5, unofficial].
    *   **Index Buffer (IBO):** 6 indices forming two triangles [S2, S5, unofficial].
    *   **Per-Instance VBO:** A dynamic buffer of size $N \times \text{sizeof(float)}$ containing sorted indices $0 \dots N-1$ [S2, unofficial]. Step rate is 1 [S2, unofficial].
    *   **Texture Buffer Objects (TBOs) / Data Textures:**
        *   `splatPositionScale` (or TBO Base): Stores per-splat centroid (xyz) and scale (w) in RGBA16F format [S5, S6, unofficial]. Size: $N \times 32$ bytes if packed as UI, or $N$ texels [S2, S5, unofficial].
        *   `splatColorOpacity` (or TBO SH): Stores SH coefficients (RGB) and opacity (A) in RGBA16F format [S5, S6, unofficial]. Size: $N \times \text{sh\_coeffs\_per\_splat} \times 8$ bytes [S2, unofficial].
        *   Sampling mode: `NEAREST` filtering and `CLAMP_TO_EDGE` wrapping [S5, S6, unofficial].

**Numbered Steps:**

1.  **Parameter Activation:** Transform raw training parameters into geometric primitives [2812-3d-gaussian-splatting.md][2058-3d-gaussian-splatting-rendering-cpu-sorted-instanced.md].
2.  **Camera Space Transformation:**
    *   Transform mean to camera space: $\boldsymbol{\mu}' = \mathbf{V} \boldsymbol{\mu}$ [2812-3d-gaussian-splatting.md][2058-3d-gaussian-splatting-rendering-cpu-sorted-instanced.md].
    *   Transform covariance: $\boldsymbol{\Sigma}' = \mathbf{V}_{1:3, 1:3} \boldsymbol{\Sigma}_{3D} \mathbf{V}_{1:3, 1:3}^T$ [2812-3d-gaussian-splatting.md][2058-3d-gaussian-splatting-rendering-cpu-sorted-instanced.md].
3.  **Screen-Space Projection:**
    *   Compute Jacobian $\mathbf{J}$ at $\boldsymbol{\mu}'$:
        $$ \mathbf{J} = \begin{pmatrix} \frac{f_x}{\mu'_z} & 0 & -\frac{f_x \mu'_x}{\mu'_z^2} \\ 0 & \frac{f_y}{\mu'_z} & -\frac{f_y \mu'_y}{\mu'_z^2} \end{pmatrix} $$
        [2812-3d-gaussian-splatting.md][2058-3d-gaussian-splatting-rendering-cpu-sorted-instanced.md].
    *   Compute 2D covariance: $\boldsymbol{\Sigma}_{2D} = \mathbf{J} \boldsymbol{\Sigma}' \mathbf{J}^T$ [2812-3d-gaussian-splatting.md][2058-3d-gaussian-splatting-rendering-cpu-sorted-instanced.md].
    *   Compute 2D mean: $\boldsymbol{\mu}_{2D} = \begin{pmatrix} \frac{f_x \mu'_x}{\mu'_z} + c_x \\ \frac{f_y \mu'_y}{\mu'_z} + c_y \end{pmatrix}$ [2812-3d-gaussian-splatting.md][2058-3d-gaussian-splatting-rendering-cpu-sorted-instanced.md].
4.  **Frustum Culling (Optional but Recommended):** Remove Gaussians outside image bounds and near/far planes (e.g., with a FOV margin ratio of 0.3) [S4, unofficial].
5.  **CPU Sorting:**
    *   Compute depth for each splat $i$: $z_i = \boldsymbol{\mu}'_i \cdot \mathbf{v}_{\text{forward}}$ (camera-space Z) [S1, S3, unofficial].
    *   Sort splats by increasing camera-space Z (far-to-near order, assuming camera looks down negative Z) [S1, unofficial].
    *   Reorder Gaussian and SH data based on sorted indices [S1, unofficial].
    *   *Optimization:* Re-sort only when camera direction delta exceeds a threshold (e.g., 1% of bounding radius) to avoid redundant computation [S5, S6, unofficial].
6.  **Upload Sorted Data:**
    *   Update the Per-Instance VBO with sorted indices [S2, unofficial].
    *   Update TBOs/Data Textures with sorted centroid, scale, color, and opacity data [S5, S6, unofficial].
    *   If $N > 65535$ (16-bit index limit), split into multiple batches [S5, S6, unofficial].
7.  **Rendering Setup:**
    *   Disable depth test: `glDisable(GL_DEPTH_TEST)` [S1, unofficial].
    *   Enable alpha blending: `glEnable(GL_BLEND)` and `glBlendFunc(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA)` [S1, unofficial].
8.  **Instanced Draw Call:**
    *   Issue `glDrawArraysInstanced(GL_TRIANGLE_STRIP, 0, 4, N)` (or per batch) [S1, S5, unofficial].
    *   Vertex Shader computes clip position and quad expansion based on 2D covariance eigenvalues [S1, unofficial].
    *   Fragment Shader computes Gaussian falloff and outputs premultiplied color/alpha [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import torch

def render_cpu_sorted_instanced(gs_means, gs_quats, gs_scales, gs_opacities, gs_sh_coeffs, camera, image_width, image_height):
    """
    Render 3D Gaussians using CPU sorting and instanced drawing logic.
    Note: This pseudocode simulates the data preparation and sorting steps.
    Actual rendering requires a graphics API (OpenGL/Vulkan/WebGPU).
    """
    # 1. Parameter Activation (from parent article)
    # q_norm, s_eff, alpha_eff, Sigma_3D computed as per [2812-3d-gaussian-splatting.md]
    
    # 2. Camera Space Transformation
    # V is the view matrix from camera
    V = camera.get_view_matrix()
    means_cam = V @ np.concatenate([gs_means, np.ones((gs_means.shape[0], 1))], axis=1).T
    means_cam = means_cam[:3, :].T  # Shape: (N, 3)
    
    # 3. Screen-Space Projection
    # Compute 2D covariance and mean as per parent article formulas
    # J, Sigma_2D, mu_2D computed
    
    # 4. Frustum Culling (Simplified)
    # Filter out points with z <= 0 or outside image bounds
    valid_indices = []
    for i in range(len(gs_means)):
        z = means_cam[i, 2]
        if z > 0.1: # Near plane check
            u, v = means_cam[i, 0] / z, means_cam[i, 1] / z
            # Check bounds (simplified)
            if 0 <= u < image_width and 0 <= v < image_height:
                valid_indices.append(i)
    
    gs_means = gs_means[valid_indices]
    gs_quats = gs_quats[valid_indices]
    gs_scales = gs_scales[valid_indices]
    gs_opacities = gs_opacities[valid_indices]
    gs_sh_coeffs = gs_sh_coeffs[valid_indices]
    means_cam = means_cam[valid_indices]
    
    N = len(gs_means)
    if N == 0:
        return np.zeros((image_height, image_width, 3))

    # 5. CPU Sorting
    # Sort by camera-space Z (far to near)
    depths = means_cam[:, 2]
    sorted_indices = np.argsort(depths) # Increasing Z (far first)
    
    # Reorder data
    gs_means = gs_means[sorted_indices]
    gs_quats = gs_quats[sorted_indices]
    gs_scales = gs_scales[sorted_indices]
    gs_opacities = gs_opacities[sorted_indices]
    gs_sh_coeffs = gs_sh_coeffs[sorted_indices]
    means_cam = means_cam[sorted_indices]
    
    # 6. Prepare Instance Data (Simulating VBO/TBO upload)
    # In reality, this data is uploaded to GPU buffers
    instance_indices = np.arange(N, dtype=np.float32)
    
    # 7. Rendering Loop (Simulating the draw call and fragment shader)
    # Since we are simulating the "Instanced" logic in CPU for verification,
    # we iterate through splats and composite.
    # In the actual GPU method, this loop is implicit in the rasterizer.
    
    output_color = np.zeros((image_height, image_width, 3), dtype=np.float32)
    output_alpha = np.zeros((image_height, image_width), dtype=np.float32)
    
    for i in range(N):
        # Skip low opacity
        if gs_opacities[i] < 0.05:
            continue
            
        # Get projected mean and covariance (from step 3)
        # For simulation, we assume we have u_mean, v_mean, Sigma_2D
        u_mean, v_mean = means_cam[i, 0] / means_cam[i, 2], means_cam[i, 1] / means_cam[i, 2]
        
        # Compute bounding box (3-sigma)
        # Eigen-decomposition of Sigma_2D
        a = Sigma_2D[i, 0, 0] + 0.3
        b = Sigma_2D[i, 1, 0]
        d = Sigma_2D[i, 1, 1] + 0.3
        det = a * d - b * b
        mid = 0.5 * (a + d)
        radius = np.sqrt(max(mid * mid - det, 0.0))
        lam1 = max(mid + radius, 0.01)
        lam2 = max(mid - radius, 0.01)
        
        # 3-sigma axes
        # e1, e2 computation...
        # b1 = 3.0 * sqrt(lam1) * e1
        # b2 = 3.0 * sqrt(lam2) * e2
        
        # Bounding box
        u_min = int(np.floor(u_mean - 3 * np.sqrt(lam1)))
        u_max = int(np.ceil(u_mean + 3 * np.sqrt(lam1)))
        v_min = int(np.floor(v_mean - 3 * np.sqrt(lam2)))
        v_max = int(np.ceil(v_mean + 3 * np.sqrt(lam2)))
        
        # Clip to image bounds
        u_min = max(0, u_min)
        u_max = min(image_width, u_max)
        v_min = max(0, v_min)
        v_max = min(image_height, v_max)
        
        # Evaluate SH Color
        # direction = camera.view_direction_cam_to_mean_world(...)
        # sh_color = eval_spherical_harmonics(gs_sh_coeffs[i], direction)
        sh_color = np.array([0.5, 0.5, 0.5]) # Placeholder
        
        # Alpha compositing (Painter's Algorithm)
        # In GPU, this is done per-pixel by the fragment shader
        # Here we simulate the accumulation for verification
        
        for u in range(u_min, u_max):
            for v in range(v_min, v_max):
                # Pixel centering
                du = u + 0.5 - u_mean
                dv = v + 0.5 - v_mean
                
                # Gaussian value at fragment
                # G(px, py) = exp(-4.5(px^2 + py^2)) where px, py are normalized quad coords
                # px = du / (3*sqrt(lam1)), py = dv / (3*sqrt(lam2))
                px = du / (3.0 * np.sqrt(lam1))
                py = dv / (3.0 * np.sqrt(lam2))
                
                power = -4.5 * (px**2 + py**2)
                
                if power < -9.0:
                    continue
                    
                alpha_base = gs_opacities[i]
                alpha_pixel = min(0.99, alpha_base * np.exp(power))
                
                if alpha_pixel < 0.001:
                    continue
                
                # Premultiplied alpha blending
                # C_out = C_in * (1 - alpha) + C_splat * alpha
                # T_out = T_in * (1 - alpha)
                
                # Current transmittance
                T = 1.0 - output_alpha[v, u]
                
                if T < 0.05: # Early termination
                    continue
                
                output_color[v, u] = output_color[v, u] * (1 - alpha_pixel) + sh_color * alpha_pixel
                output_alpha[v, u] = output_alpha[v, u] + alpha_pixel * T

    return output_color
```

**Parameters:**
*   **Jitter:** 0.3 added to 2D covariance diagonal elements (`covariance2D[0][0]` and `[1][1]`) to prevent singularity [S1, unofficial].
*   **Minimum Eigenvalue:** 0.01 clamp on `lam1` and `lam2` [S1, unofficial].
*   **Alpha Clamp:** Maximum alpha 0.99 [S1, unofficial].
*   **Alpha Discard:** Fragments with alpha < 0.001 are discarded [S1, unofficial].
*   **Power Discard:** Fragments with power < -9.0 are discarded [S1, unofficial].
*   **Sort Re-trigger Threshold:** Camera movement must exceed 1% of bounding radius to re-sort [S5, S6, unofficial].
*   **Max Instances per Batch:** 65535 (due to 16-bit index buffer) [S5, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Use `NEAREST` filtering and `CLAMP_TO_EDGE` wrapping for data textures to avoid blending adjacent splat attributes [S6, unofficial].
*   **Bit Flips:** Implement a Support Guard to clamp parameters to valid ranges to prevent catastrophic rendering errors from SEUs [17686-support-guard.md].
*   **Large Scenes:** If $N > 65535$, split into multiple batches and use `gl_InstanceID` offset to index into textures [S5, S6, unofficial].
*   **Web Runtime ABI:** Ensure material compilation matches the runtime version (e.g., Filament.js 1.70.2 vs Android 1.71.5) [S5, unofficial].

## Subtopics

### [Tiled 2D Gaussian Splatting Rasterization](2061-tiled-2d-gaussian-splatting-rasterization.md)
GPU-accelerated, tile-based parallel processing. Choose for real-time performance on high-end GPUs.

### [Gaussian Splat Rendering (Screen-Space Quad)](2075-gaussian-splat-rendering-screen-space-quad.md)
Display-only, CPU-side sorting. Choose for client-side visualization without complex GPU shaders.

### [Depth-First Splat Sorting](2850-depth-first-splat-sorting.md)
Preprocessing step to reduce sorting overhead. Choose when fragment count is high.

### [Splatapult Gaussian Splat Renderer](6749-splatapult-gaussian-splat-renderer.md)
VR/AR focused, background CPU sorting. Choose for mobile XR hardware.

### [SortFree Weighting](8006-sortfree-weighting.md)
Order-independent transparency approximation. Choose when sorting latency is prohibitive.

### [Unity Mobile Implementation](9536-unity-mobile-implementation.md)
Unity engine optimized. Choose for mobile avatar rendering in Unity.

### [CPU-based Gaussian Splatting Thumbnailer](14038-cpu-based-gaussian-splatting-thumbnailer.md)
Pure CPU, JIT-compiled. Choose for fast preview generation.

### [Support Guard](17686-support-guard.md)
Fault mitigation for SEUs. Choose for spaceborne/robotic hardware.

### [SimpleRenderer](19069-simplerenderer.md)
CPU-based, non-differentiable. Choose for debugging or no-GPU environments.

## Operating characteristics
*   **Time Complexity:** Sorting is $O(N \log N)$ on the CPU [S5, unofficial]. Rendering is $O(N \cdot A)$ where $A$ is average splat area in pixels, but offloaded to GPU rasterizer [S1, unofficial].
*   **Memory Complexity:** $O(N)$ for storing splat data in textures/VBOs [S2, S5, unofficial]. SH0 splats require ~28 bytes/splat in RGBA16F textures [S5, unofficial].
*   **Measured Performance:** CPU radix sort of ~500k depth keys takes a few milliseconds on a worker thread [S5, unofficial]. Mobile-GS reports up to 127 FPS on modern mobile GPUs using this instanced approach with OIT [S5, unofficial].
*   **Strengths:** Simple implementation, works on standard graphics APIs (OpenGL/Vulkan/WebGPU) without compute shaders [S5, unofficial], suitable for mobile and web [S5, S6, unofficial].
*   **Limitations:** Less efficient than tile-based GPU sorting for very large scenes [S1, unofficial], sorting latency on CPU can cause frame drops if not throttled [S5, S6, unofficial].
*   **When to Use:** Mid-range mobile devices, web browsers, or environments lacking compute shader support [S5, unofficial].
*   **When Not to Use:** High-end desktop rendering where tile-based GPU rasterization offers better performance [S1, unofficial].

## Background
Originated from the paper "3D Gaussian Splatting for Real-Time Radiance Field Rendering" by Kerbl et al. (2023) [S1, unofficial]. This specific implementation variant was developed to enable Gaussian Splatting rendering on platforms without advanced GPU compute capabilities, such as mobile devices and web browsers, using standard instanced drawing techniques [S5, S6, unofficial]. It is used in Android SplatNode, Filament-based renderers, and web viewers [S5, S6, unofficial].

## Sources
[S1] 3D Gaussian Splatting in a Weekend, https://bfeldman.me/3dgs-weekend/, fetched 2026-10-26 (unofficial)
[S2] splat-ros/RVizSplat, https://github.com/splat-ros/RVizSplat, fetched 2026-10-27 (unofficial)
[S3] akbartus/Gaussian-Splatting-WebViewers, https://github.com/akbartus/Gaussian-Splatting-WebViewers, fetched 2026-10-27 (unofficial)
[S4] sascha-kirch/splaty, https://github.com/sascha-kirch/splaty, fetched 2026-10-27 (unofficial)
[S5] feat(sceneview): SplatNode — 3D Gaussian Splatting rendering (Android Filament custom material; iOS RealityKit native; web follows), https://github.com/sceneview/sceneview/issues/2646, fetched 2026-10-27 (unofficial)
[S6] feat(sceneview): SplatNode P1 — static 3D Gaussian Splat rendering ( #…, https://github.com/sceneview/sceneview/commit/9dc024089729c3822a48893901c09dbb0c1a04d4, fetched 2026-10-29 (unofficial)
