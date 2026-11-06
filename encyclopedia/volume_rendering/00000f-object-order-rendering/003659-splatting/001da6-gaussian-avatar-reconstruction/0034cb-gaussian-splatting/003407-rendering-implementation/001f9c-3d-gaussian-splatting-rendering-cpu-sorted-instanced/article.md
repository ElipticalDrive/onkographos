# 3D Gaussian Splatting Rendering (CPU-Sorted, Instanced)

Also called: 3D Gaussian Splatting rendering on Android, 3DGS Weekend Renderer, Depth-based Splat Sorting, Instanced Gaussian Splatting with Back-to-Front Sorting, SplatNode, SplatNode P1

## Overview

This method renders trained 3D Gaussian Splatting (3DGS) scenes by projecting 3D Gaussian distributions into 2D screen-space quads, sorting them by camera-space depth on the CPU, and rendering them using hardware instancing with standard alpha blending [S1, unofficial][S4, unofficial]. It solves the transparency ordering problem inherent in translucent splat rendering, where simple depth testing is insufficient because multiple splats contribute to a single pixel along the same ray [S1, unofficial]. Unlike the official tile-based CUDA rasterizer which uses GPU sorting and front-to-back blending, this approach relies on CPU-side back-to-front sorting and standard OpenGL/WebGL blending, making it simpler to implement but less efficient for large scenes [S1, unofficial]. It builds on [Gaussian Splatting](946-gaussian-splatting.md) by utilizing volume splatting for efficient rasterization.

## How it works

### Inputs and Outputs
**Inputs:**
*   A set of $M$ 3D Gaussians, each defined by:
    *   Mean position $\mu \in \mathbb{R}^3$ (world space).
    *   Opacity $\alpha \in \mathbb{R}$.
    *   Scale $s \in \mathbb{R}^3$ and Rotation $q \in \mathbb{R}^4$ (defining the 3D covariance $\Sigma$).
    *   Spherical Harmonics (SH) coefficients for color (optional; P1 scope uses SH degree 0, i.e., constant color) [S4, unofficial].
*   Camera parameters: View matrix $V$, Projection matrix $P$, and viewport size $(W, H)$ [S1, unofficial].

**Outputs:**
*   A 2D image with RGB color and alpha channel, composited over a background (typically black) [S3, unofficial].

### Data Structures

**1. Gaussian Data Layout (CPU/GPU Upload)**
The per-splat data is typically stored in a contiguous array or buffer. A common layout for upload to the GPU (e.g., `GpuGaussian` struct) is [S1, unofficial]:
*   `centroid`: `vec3` (12 bytes)
*   `opacity`: `float` (4 bytes)
*   `scale`: `array<float, 3>` (12 bytes)
*   `rotation`: `array<float, 4>` (16 bytes)
*   **Total size:** 44 bytes per splat.

For instanced rendering with data textures (e.g., Filament/SceneView), two square `RGBA16F` textures are used [S4, unofficial]:
*   `splatPositionScale`: Per-splat `xyz` = center (model space), `w` = billboard half-extent.
*   `splatColorOpacity`: Per-splat `rgb` = linear color, `a` = straight opacity.

**2. Sorting Buffer**
To sort splats by depth, a parallel array is maintained. In JavaScript/Three.js implementations, this is a `BigInt64Array` (viewed as `Float32Array` and `Uint32Array`) of size $2 \times M$ [S2, unofficial]:
*   Even indices ($2j$): Original splat index $j$.
*   Odd indices ($2j+1$): Packed depth value.

In C++/OpenGL implementations, a struct `SplatSortEntry` is used [S1, unofficial]:
*   `cameraZ`: `float` (4 bytes)
*   `splatIndex`: `size_t` (8 bytes)

### Numbered Steps

1.  **Frustum Culling:** Transform 3D means to camera space using the view matrix. Project camera-space means to the 2D image plane. Perform frustum culling to remove off-screen Gaussians, including image bounds and near/far planes with a FOV margin ratio (typically 0.3) [S3, unofficial].
2.  **Depth Sorting (CPU):**
    *   Transform splat centroids to camera space: $C_i = V \cdot [\mu_i, 1]^T$.
    *   Extract camera-space Z coordinate: $z_i = C_i.z$. Note: In standard OpenGL camera space, the camera looks down the negative Z axis, so farther splats have more negative Z.
    *   Sort splats by increasing camera-space Z (far-to-near order) [S1, unofficial].
    *   *Optimization:* Throttle sorting on minor camera movements by comparing the current view projection components against the last sorted view. If the dot product difference is small (e.g., $< 0.01$), skip sorting [S2, unofficial].
3.  **Rebuild Upload Data:** Reorder the Gaussian and SH data based on the sorted indices to create contiguous arrays for the GPU [S1, unofficial]. If the number of splats exceeds the hardware instancing limit (e.g., 65,535 in Filament), split the cloud into multiple batches, adjusting the `instanceOffset` for each batch [S4, unofficial].
4.  **Vertex Shader Execution (Per Instance):**
    *   Compute clip space position: $clip = P \cdot V \cdot [\mu, 1]^T$.
    *   Compute camera space centroid: $center_{cam} = V \cdot [\mu, 1]^T$.
    *   Compute 2D covariance by projecting the 3D covariance to screen: $\Sigma_{2D} = J M \Sigma (J M)^T$ [S1, unofficial].
    *   Extract 2x2 covariance components with jitter (0.3) added to diagonals to prevent singularity: $a = \Sigma_{2D}[0][0] + 0.3$, $b = \Sigma_{2D}[1][0]$, $d = \Sigma_{2D}[1][1] + 0.3$ [S1, unofficial].
    *   Perform closed-form eigen-decomposition of the 2x2 covariance to get eigenvalues $\lambda_1, \lambda_2$ and eigenvectors $e_1, e_2$ [S1, unofficial]:
        $$
        \text{det} = a \cdot d - b^2, \quad \text{mid} = 0.5(a + d), \quad \text{rad} = \sqrt{\max(\text{mid}^2 - \text{det}, 0.0)}
        $$
        $$
        \lambda_1 = \max(\text{mid} + \text{rad}, 0.01), \quad \lambda_2 = \max(\text{mid} - \text{rad}, 0.01)
        $$
        $$
        e_1 = \begin{cases} \text{normalize}(b, \lambda_1 - a) & \text{if } |b| > 10^{-5} \\ (1, 0) & \text{if } a \ge d \\ (0, 1) & \text{otherwise} \end{cases}, \quad e_2 = (-e_1.y, e_1.x)
        $$
    *   Compute 3-sigma principal half-axes in pixels: $b_1 = 3\sqrt{\lambda_1}e_1$, $b_2 = 3\sqrt{\lambda_2}e_2$ [S1, unofficial].
    *   Compute vertex offset in NDC from the unit quad corner $(q_x, q_y)$:
        $$
        \delta_{pixel} = q_x b_1 + q_y b_2, \quad \delta_{ndc} = \frac{\delta_{pixel}}{0.5 \cdot \text{viewportSize}}
        $$
    *   Set final position: $gl\_Position = [ndc.x + \delta_{ndc}.x, ndc.y + \delta_{ndc}.y, ndc.z, 1.0]^T$.
    *   Pass varyings: `splatCoord` = $(q_x, q_y)$, `splatColor` = SH evaluation, `splatOpacity` = $\sigma(\alpha)$ [S1, unofficial].
5.  **Fragment Shader Execution (Per Pixel):**
    *   Compute power term: $power = -4.5 \cdot (splatCoord.x^2 + splatCoord.y^2)$ [S1, unofficial].
    *   Discard if $power < -9.0$ (approximates $e^{-9} \approx 0$) [S1, unofficial].
    *   Compute final alpha: $alpha = \min(0.99, splatOpacity \cdot e^{power})$ [S1, unofficial].
    *   Discard if $alpha < 0.001$ [S1, unofficial].
    *   Output color: $FragColor = [splatColor, alpha]$ [S1, unofficial].
6.  **Rendering Setup:**
    *   Disable depth test: `glDisable(GL_DEPTH_TEST)`.
    *   Enable alpha blending: `glEnable(GL_BLEND)`, `glBlendFunc(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA)` [S1, unofficial].
    *   Draw all splats in a single instanced call: `glDrawArraysInstanced(GL_TRIANGLE_STRIP, 0, 4, M)` [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def render_cpu_sorted_instanced(gs_means, gs_opacities, gs_scales, gs_quats, gs_sh_coeffs, 
                                camera_view, camera_proj, viewport_size, background_color=np.zeros(3)):
    """
    Renders 3DGS using CPU sorting and instanced drawing.
    gs_means: (N, 3)
    gs_opacities: (N,)
    gs_scales: (N, 3)
    gs_quats: (N, 4)
    gs_sh_coeffs: (N, K) - K depends on SH degree. For SH0, K=3 (RGB).
    camera_view: (4, 4)
    camera_proj: (4, 4)
    viewport_size: (W, H)
    """
    N = gs_means.shape[0]
    
    # 1. Transform to Camera Space
    # Homogeneous coordinates
    ones = np.ones((N, 1))
    means_hom = np.hstack([gs_means, ones])
    
    # Camera space means
    means_cam_hom = (camera_view @ means_hom.T).T
    means_cam = means_cam_hom[:, :3]
    camera_z = means_cam_hom[:, 2]
    
    # 2. Frustum Culling (Simplified: Image Bounds + Near/Far)
    # Project to NDC
    clip_hom = (camera_proj @ means_cam_hom.T).T
    clip_ndc = clip_hom[:, :3] / clip_hom[:, 3:4]
    
    # Image bounds check
    in_bounds = (np.abs(clip_ndc[:, 0]) < 1.0) & (np.abs(clip_ndc[:, 1]) < 1.0)
    # Near/Far plane check (assuming standard OpenGL clip space Z: -1 to 1)
    # Note: Original sources mention FOV margin ratio 0.3, but standard implementation often just checks bounds.
    # We'll stick to basic bounds for P1 scope as per S4.
    visible_mask = in_bounds
    
    # Filter visible splats
    visible_indices = np.where(visible_mask)[0]
    if len(visible_indices) == 0:
        return background_color.reshape(3, 1, 1) * np.ones((1, viewport_size[1], viewport_size[0]))

    v_means = gs_means[visible_indices]
    v_opacities = gs_opacities[visible_indices]
    v_scales = gs_scales[visible_indices]
    v_quats = gs_quats[visible_indices]
    v_sh = gs_sh_coeffs[visible_indices]
    v_cam_z = camera_z[visible_indices]
    
    # 3. CPU Sorting (Back-to-Front / Far-to-Near)
    # Sort by increasing camera Z (more negative is further)
    sort_order = np.argsort(v_cam_z)
    
    sorted_indices = visible_indices[sort_order]
    sorted_means = v_means[sort_order]
    sorted_opacities = v_opacities[sort_order]
    sorted_scales = v_scales[sort_order]
    sorted_quats = v_quats[sort_order]
    sorted_sh = v_sh[sort_order]
    sorted_cam_z = v_cam_z[sort_order]
    
    # 4. Prepare Data for Instancing
    # In a real GPU implementation, this data is uploaded to buffers/textures.
    # Here we simulate the vertex shader logic for a single quad per splat 
    # to demonstrate the math, though actual rendering uses glDrawArraysInstanced.
    
    # Unit quad corners
    quad_corners = np.array([[-1, -1], [1, -1], [1, 1], [-1, 1]])
    
    # Precompute 2D Covariance and Eigen-decomposition for each splat
    # This is typically done in the Vertex Shader, but we compute it here for clarity
    # or assume it's passed as attributes.
    
    # For the sake of the "CPU-Sorted" definition, the key is the sorting step.
    # The rendering step is standard instanced draw.
    
    # 5. Rendering (Conceptual)
    # glDisable(GL_DEPTH_TEST)
    # glEnable(GL_BLEND)
    # glBlendFunc(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA)
    # glDrawArraysInstanced(GL_TRIANGLE_STRIP, 0, 4, len(sorted_means))
    
    # Return placeholder for the actual pixel shader output
    # The actual pixel values depend on the fragment shader evaluation
    # which requires iterating over pixels. For this method description,
    # we define the pipeline.
    
    return {
        "sorted_means": sorted_means,
        "sorted_opacities": sorted_opacities,
        "sorted_sh": sorted_sh,
        "sorted_cam_z": sorted_cam_z,
        "quad_corners": quad_corners
    }

def fragment_shader_eval(splat_coord, splat_opacity, splat_color):
    """
    Evaluates the fragment shader for a given splat at a specific quad coordinate.
    splat_coord: (2,) - (px, py) in [-1, 1]
    splat_opacity: float - sigmoid(opacity)
    splat_color: (3,) - RGB color
    """
    power = -4.5 * np.dot(splat_coord, splat_coord)
    
    if power < -9.0:
        return None, None # Discard
    
    alpha = min(0.99, splat_opacity * np.exp(power))
    
    if alpha < 0.001:
        return None, None # Discard
        
    return splat_color * alpha, alpha
```

### Parameters
*   **Jitter:** 0.3 added to 2D covariance diagonal elements to prevent singularity [S1, unofficial].
*   **Minimum Eigenvalue:** 0.01 clamp on $\lambda_1, \lambda_2$ [S1, unofficial].
*   **Alpha Clamping:** Max alpha 0.99 [S1, unofficial].
*   **Alpha Discard Threshold:** 0.001 [S1, unofficial].
*   **Power Discard Threshold:** -9.0 [S1, unofficial].
*   **Sigma Multiplier:** 3.0 for 3-sigma extent (used in bounding box calculation for CPU rasterization variants, but in instanced rendering, the quad size is determined by the eigenvalues) [S3, unofficial].
*   **Radius Scaling:** In some CPU rasterization variants, radius is `radius_px / depth`, clamped between 1.0 and `radius_px` [S3, unofficial].

### Edge Cases and Pitfalls
*   **Precision Issues:** Fragments with power < -9.0 are discarded to avoid precision issues and unnecessary computation [S1, unofficial].
*   **Redundant Sorting:** Minor camera movements can cause redundant sorting. This is handled by checking the dot product of view projection components against the last sorted view. If the difference is small (< 0.01), sorting is skipped [S2, unofficial].
*   **Visual Popping:** Without a `cameraPositionProvider` (or equivalent mechanism) to trigger re-sorting on camera motion, the compositing order remains as loaded, causing visual popping when orbiting the camera [S4, unofficial].
*   **Texture Filtering:** Textures must be sampled with `NEAREST` filtering and `CLAMP_TO_EDGE` wrapping. Interpolation would blend adjacent splats' attributes into garbage because texels are addressed exactly at their centers [S4, unofficial].
*   **Instance Limit:** Clouds exceeding the hardware instancing limit (e.g., 65,535 in Filament) must be split into multiple batches [S4, unofficial].

## Subtopics

### [Tiled 2D Gaussian Splatting Rasterization](2061-tiled-2d-gaussian-splatting-rasterization.md)
Tiled 2D Gaussian Splatting Rasterization is a GPU-parallel rendering method for 3D Gaussian Splatting that divides the image into fixed-size tiles to manage memory and processing load, assigning Gaussians to tiles based on their axis-aligned bounding boxes (AABBs). It solves the performance bottleneck of sorting millions of splats per pixel by grouping work into tiles, enabling real-time rendering. Unlike non-tiled approaches that iterate over all Gaussians per pixel or use screen-space quads without tiling, this method restricts pixel iteration to relevant Gaussians within each tile, significantly reducing redundant computations.

### [Gaussian Splat Rendering (Screen-Space Quad)](2075-gaussian-splat-rendering-screen-space-quad.md)
This method renders 3D Gaussian Splatting scenes by projecting 3D Gaussian distributions into 2D screen-space quads, utilizing a pre-sorted order buffer and view-space projection data to handle transparency via alpha blending. Unlike its parent [3D Gaussian Splatting Rendering (CPU-Sorted, Instanced)](2058-3d-gaussian-splatting-rendering-cpu-sorted-instanced.md), which uses instanced drawing with standard alpha blending, this method explicitly constructs screen-space quads in the vertex shader to allow for precise frustum culling and selection highlighting, and it differs from the sibling [Tiled 2D Gaussian Splatting Rasterization](2061-tiled-2d-gaussian-splatting-rasterization.md) by not using tiled memory optimizations or screen-space depth sorting.

### [Depth-First Splat Sorting](2850-depth-first-splat-sorting.md)
Depth-First Splat Sorting is a preprocessing step for 3D Gaussian Splatting rendering that sorts visible Gaussian splats by camera-space depth before they are expanded into per-tile fragments. It solves the problem of high sorting overhead inherent in the original 3DGS pipeline, where fragments (one per tile a splat touches) are sorted using a combined 48-bit key. By sorting the smaller set of visible splats first, this method reduces the number of items to sort by a factor of 1.5 to 4, significantly lowering computational cost while maintaining correct depth ordering for subsequent rasterization.

### [Splatapult Gaussian Splat Renderer](6749-splatapult-gaussian-splat-renderer.md)
Splatapult is a CPU-sorted, instanced renderer for 3D Gaussian Splatting (3DGS) scenes that performs depth sorting on the GPU using a multi-radix sort algorithm or falls back to a CPU-based radix sort. It projects 3D Gaussians into screen-space quads and renders them using standard alpha blending, supporting VR via OpenXR and optional full spherical harmonic (SH) color evaluation. It distinguishes itself from sibling renderers by providing a complete, standalone implementation of the GPU-based sorting pipeline required for correct depth ordering in instanced rendering, rather than relying on external libraries or CPU-only sorting.

## Operating characteristics

*   **Time Complexity:** Sorting takes $O(M \log M)$ where $M$ is the number of splats. Rendering is $O(M)$ for the draw call, but the fragment shader work is proportional to the number of pixels covered by splats.
*   **Memory Complexity:** $O(M)$ for storing Gaussian data and sorting buffers.
*   **Measured Performance:** This method is less efficient than the official 3DGS renderer's tile-based CUDA rasterizer with GPU sorting and front-to-back blending [S1, unofficial]. It works fine for scenes that are not too large but will be fairly slow for large scenes due to CPU sorting overhead [S1, unofficial].
*   **Strengths:** Simple to implement using standard OpenGL/WebGL features. No need for custom CUDA kernels or complex GPU sorting algorithms.
*   **Limitations:** CPU sorting becomes a bottleneck for large scenes. Instancing limits the number of splats per draw call. Standard alpha blending is less efficient than front-to-back blending with early ray termination.
*   **When to Use:** Small to medium scenes, educational purposes, or environments where GPU sorting is not available or too complex to implement.
*   **When Not to Use:** Large-scale real-time applications requiring high performance and large scene sizes.

## Background

This method is based on the paper '3D Gaussian Splatting for Real-Time Radiance Field Rendering' by Kerbl et al. (2023) [S1, unofficial]. It was developed as a simpler alternative to the official GPU-based rasterizer, often referred to as the "3DGS Weekend Renderer" [S1, unofficial]. It has been implemented in various contexts, including Android rendering via Filament (SplatNode) [S4, unofficial] and WebGL viewers [S2, unofficial].

## Sources

[S1] 3D Gaussian Splatting in a Weekend, https://bfeldman.me/3dgs-weekend/, fetched 2026-10-26 (unofficial)
[S2] akbartus/Gaussian-Splatting-WebViewers, https://github.com/akbartus/Gaussian-Splatting-WebViewers, fetched 2026-10-27 (unofficial)
[S3] sascha-kirch/splaty, https://github.com/sascha-kirch/splaty, fetched 2026-10-27 (unofficial)
[S4] feat(sceneview): SplatNode P1 — static 3D Gaussian Splat rendering ( #…, https://github.com/sceneview/sceneview/commit/9dc024089729c3822a48893901c09dbb0c1a04d4, fetched 2026-10-29 (unofficial)
