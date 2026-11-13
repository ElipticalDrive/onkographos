# 3D Gaussian Splatting Rendering (CPU-Sorted, Instanced)

Also called: 3DGS Weekend Renderer, Gaussian Splatting Instanced Draw, RViz Splat Cloud, SplatCloud Instanced Rendering

## Overview
This method renders trained 3D Gaussian Splatting scenes by projecting 3D Gaussians into 2D screen-space quads, sorting them by camera-space depth on the CPU, and drawing them using hardware-instanced rendering with standard alpha blending [S1, S2, S5, S6, unofficial]. It solves the problem of rendering translucent Gaussian splats correctly without relying on GPU compute shaders or tile-based rasterization, making it suitable for environments where GPU sorting is unavailable or impractical (e.g., older WebGL contexts, specific engine constraints) [S1, S5, unofficial]. Unlike [Tiled 2D Gaussian Splatting Rasterization](2061-tiled-2d-gaussian-splatting-rasterization.md), which uses GPU-side tile-based sorting and compositing, this approach performs a global back-to-front sort on the CPU and relies on the graphics pipeline's fixed-function alpha blending [S1, S2, unofficial]. It differs from [Gaussian Splat Rendering (Screen-Space Quad)](2075-gaussian-splat-rendering-screen-space-quad.md) by using generic instanced drawing with texture-based attribute fetches rather than explicit screen-space delta construction in the vertex shader for perspective correction [S2, S5, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of $N$ 3D Gaussians defined by mean $\mu$, scale $s$, rotation $q$, opacity $\alpha$, and spherical harmonics (SH) coefficients $c_{\ell m}$ [S1, S4, unofficial].
*   **Output:** A 2D image with correct alpha compositing order.

**Data Structures:**
1.  **Gaussian Data:** Stored in Texture Buffer Objects (TBOs) or 2D textures to bypass uniform array limits.
    *   `splatPositionScale`: RGBA16F texture. Layout: $xyz$ = center position (model space), $w$ = billboard half-extent/scale [S5, S6, unofficial].
    *   `splatColorOpacity`: RGBA16F texture. Layout: $rgb$ = SH0 color (linear), $a$ = straight opacity [S5, S6, unofficial].
    *   *Note:* For SH1 or higher, coefficients are packed into TBOs or additional textures [S2, S5, unofficial]. SH0 requires ~28 bytes/splat [S5, unofficial].
2.  **Geometry:**
    *   **Quad VBO:** 4 vertices (unit quad corners: $(-1,-1), (1,-1), (-1,1), (1,1)$) [S1, S2, unofficial].
    *   **Index Buffer (IBO):** 6 indices forming two triangles [S2, unofficial].
    *   **Per-Instance VBO:** Contains sorted splat indices (or is implicit via texture sampling using `gl_InstanceID`) [S2, S5, unofficial].
3.  **Sorting Buffer:**
    *   `SplatSortEntry`: Struct containing `cameraZ` (float, 4 bytes) and `splatIndex` (size_t, 8 bytes) [S1, unofficial].
    *   Alternatively, a packed array `depthMix` (BigInt64Array/Float32Array view) where even offsets store the index and odd offsets store the depth key [S3, unofficial].

**Steps:**

1.  **Frustum Culling and Preprocessing:**
    *   Transform Gaussian means to camera space: $\mu' = V \mu$ [S4, unofficial].
    *   Cull Gaussians behind the near plane ($\mu'_z < z_{near}$) or outside the view frustum [S3, S4, unofficial].
    *   Cull Gaussians with opacity below a threshold (e.g., 0.01 or 0.05) [S4, S1, unofficial].

2.  **CPU Sorting:**
    *   Compute the camera-space depth $z$ for each visible Gaussian [S1, unofficial].
    *   Sort the Gaussians in **back-to-front** order (increasing camera-space $z$, since the camera looks down the negative $z$ axis in OpenGL convention) [S1, S3, unofficial].
    *   *Optimization:* Re-sort only when the camera direction changes significantly (e.g., dot product of view projection components changes by > 0.01) to avoid redundant computation [S3, S5, unofficial].
    *   *Implementation:* Use CPU radix sort or standard sort on a worker thread [S5, unofficial].

3.  **Data Upload:**
    *   Reorder the Gaussian and SH data according to the sorted indices [S1, unofficial].
    *   Upload the sorted data to the GPU textures/TBOs [S5, unofficial].
    *   If the number of splats $N$ exceeds the hardware limit (e.g., 65,535 for 16-bit indices), split into multiple batches [S5, S6, unofficial].

4.  **Rendering Setup:**
    *   Disable depth testing: `glDisable(GL_DEPTH_TEST)` [S1, unofficial].
    *   Enable alpha blending: `glEnable(GL_BLEND)` with `glBlendFunc(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA)` [S1, unofficial].

5.  **Vertex Shader Execution (Per Instance):**
    *   Fetch per-splat attributes (position, scale, color, opacity) from textures using `gl_InstanceID` [S5, unofficial].
    *   Compute clip-space position: $clip = P \cdot V \cdot [\mu, 1]^T$ [S1, unofficial].
    *   Compute camera-space centroid: $\mu_{cam} = V \cdot [\mu, 1]^T$ [S1, unofficial].
    *   Project 3D covariance to 2D screen-space covariance $\Sigma_{2D}$ using the Jacobian $J$ of the projection [S1, S3, unofficial].
    *   Compute eigenvalues $\lambda_1, \lambda_2$ and eigenvectors of $\Sigma_{2D}$ [S1, unofficial].
        *   Jitter diagonal elements by 0.3 to prevent instability [S1, unofficial].
        *   Clamp eigenvalues to a minimum of 0.01 [S1, unofficial].
    *   Compute 3-sigma principal half-axes: $b_1 = 3\sqrt{\lambda_1}e_1$, $b_2 = 3\sqrt{\lambda_2}e_2$ [S1, unofficial].
    *   Compute vertex offset in NDC: $\delta_{NDC} = (quadCorner \cdot [b_1, b_2]) / (viewportSize \cdot 0.5)$ [S1, unofficial].
    *   Set final position: $gl\_Position = [ndc.xy + \delta_{NDC}, ndc.z, 1.0]$ [S1, unofficial].
    *   Pass varying `splatCoord` (quadCorner) and `splatColor` (evaluated SH) to fragment shader [S1, unofficial].

6.  **Fragment Shader Execution:**
    *   Compute power term: $power = -4.5 \cdot (p_x^2 + p_y^2)$, where $(p_x, p_y)$ is the fragment's position within the unit quad [S1, unofficial].
    *   Discard if $power < -9.0$ (equivalent to $\alpha \approx 0$) [S1, unofficial].
    *   Compute final alpha: $\alpha = \min(0.99, \alpha_{base} \cdot \exp(power))$ [S1, unofficial].
    *   Discard if $\alpha < 0.001$ [S1, unofficial].
    *   Output color: $FragColor = [color, \alpha]$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_cpu_sorted_instanced(gaussians, camera, viewport_size):
    """
    Render 3D Gaussians using CPU sorting and instanced drawing.
    """
    # 1. Transform and Cull
    means_cam = camera.world_to_cam(gaussians.means)
    
    # Frustum culling and opacity threshold
    visible_indices = []
    for i, mean in enumerate(means_cam):
        if mean.z < camera.near_plane:
            continue
        if gaussians.opacities[i] < 0.05:
            continue
        visible_indices.append(i)
        
    if not visible_indices:
        return empty_image()

    # 2. CPU Sorting (Back-to-Front)
    # Camera looks down negative Z, so increasing Z is far-to-near
    sort_keys = [(means_cam[i].z, i) for i in visible_indices]
    sort_keys.sort(key=lambda x: x[0]) # Ascending Z
    
    sorted_indices = [idx for _, idx in sort_keys]
    
    # 3. Prepare Data for GPU
    # Reorder attributes based on sorted_indices
    sorted_means = gaussians.means[sorted_indices]
    sorted_opacities = gaussians.opacities[sorted_indices]
    sorted_colors = evaluate_sh_colors(gaussians.sh_coeffs, sorted_indices, camera)
    
    # Upload to textures/TBOs
    upload_texture("splatPositionScale", sorted_means, gaussians.scales[sorted_indices])
    upload_texture("splatColorOpacity", sorted_colors, sorted_opacities)
    
    # Handle batching if N > 65535
    batch_size = 65535
    num_batches = (len(sorted_indices) + batch_size - 1) // batch_size
    
    # 4. Render Setup
    glDisable(GL_DEPTH_TEST)
    glEnable(GL_BLEND)
    glBlendFunc(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA)
    
    for batch in range(num_batches):
        start = batch * batch_size
        end = min(start + batch_size, len(sorted_indices))
        count = end - start
        
        # 5. Draw Instanced
        # Vertex shader uses gl_InstanceID to fetch from textures
        glDrawArraysInstanced(GL_TRIANGLE_STRIP, 0, 4, count)
        
    return final_image
```

## Subtopics

### [Tiled 2D Gaussian Splatting Rasterization](2061-tiled-2d-gaussian-splatting-rasterization.md)
Uses GPU-side tile-based sorting and compositing for higher performance.

### [Gaussian Splat Rendering (Screen-Space Quad)](2075-gaussian-splat-rendering-screen-space-quad.md)
Uses explicit screen-space deltas in the vertex shader for perspective correction.

### [Depth-First Splat Sorting](2850-depth-first-splat-sorting.md)
Optimizes sorting by sorting visible splats before tile expansion.

### [Splatapult Gaussian Splat Renderer](6749-splatapult-gaussian-splat-renderer.md)
Standalone renderer with GPU sorting fallback.

### [SortFree Weighting](8006-sortfree-weighting.md)
Order-independent transparency approximation avoiding sorting.

### [Unity Mobile Implementation](9536-unity-mobile-implementation.md)
Mobile-optimized engine integration.

### [CPU-based Gaussian Splatting Thumbnailer](14038-cpu-based-gaussian-splatting-thumbnailer.md)
Simplified CPU-only preview renderer.

## Operating characteristics
*   **Time Complexity:** Sorting is $O(N \log N)$ on the CPU [S5, unofficial]. Rendering is $O(N)$ for the draw call, but fragment shader work is proportional to the number of splats covering pixels [S1, unofficial].
*   **Memory Complexity:** $O(N)$ for storing Gaussian data in textures/TBOs [S5, unofficial]. SH0 requires ~28 bytes/splat [S5, unofficial].
*   **Performance:** Sorting ~500k splats on CPU takes a few milliseconds on a worker thread [S5, unofficial]. Re-sorting is throttled to camera motion events [S5, unofficial].
*   **Strengths:** Simple implementation, works on hardware without compute shaders (e.g., WebGL1/older OpenGL), easy to integrate into existing engines [S1, S5, unofficial].
*   **Limitations:** Less efficient than GPU-tiled methods due to CPU sorting overhead and lack of early ray termination [S1, unofficial]. Limited by 16-bit index buffer (65,535 instances per draw call) [S5, unofficial].
*   **When to use:** When GPU compute shaders are unavailable, or for simple/preview renderers [S1, S5, unofficial].
*   **When not to use:** For high-performance real-time rendering of large scenes (>1M splats) where GPU sorting and tiled rasterization are required [S1, unofficial].

## Background
The method is based on the paper "3D Gaussian Splatting for Real-Time Radiance Field Rendering" by Kerbl et al. (2023) [S1, unofficial]. It was popularized by the "3D Gaussian Splatting in a Weekend" tutorial by Feldman, which demonstrated a CPU-sorted, OpenGL-based implementation [S1, unofficial]. It has been implemented in various contexts, including RViz for ROS [S2, unofficial], web viewers [S3, unofficial], and mobile engines like Unity via Filament [S5, S6, unofficial].

## Sources
[S1] 3D Gaussian Splatting in a Weekend, https://bfeldman.me/3dgs-weekend/, fetched 2026-10-26 (unofficial)
[S2] splat-ros/RVizSplat, https://github.com/splat-ros/RVizSplat, fetched 2026-10-27 (unofficial)
[S3] akbartus/Gaussian-Splatting-WebViewers, https://github.com/akbartus/Gaussian-Splatting-WebViewers, fetched 2026-10-27 (unofficial)
[S4] sascha-kirch/splaty, https://github.com/sascha-kirch/splaty, fetched 2026-10-27 (unofficial)
[S5] feat(sceneview): SplatNode — 3D Gaussian Splatting rendering (Android Filament custom material; iOS RealityKit native; web follows), https://github.com/sceneview/sceneview/issues/2646, fetched 2026-10-27 (unofficial)
[S6] feat(sceneview): SplatNode P1 — static 3D Gaussian Splat rendering ( #…, https://github.com/sceneview/sceneview/commit/9dc024089729c3822a48893901c09dbb0c1a04d4, fetched 2026-10-29 (unofficial)
