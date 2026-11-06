# Performance Optimization Techniques
## Overview
Performance Optimization Techniques are a collection of heuristic strategies designed to mitigate the high computational cost of raymarching volumetric textures by reducing the number of shader evaluations and memory accesses per frame [S1, unofficial]. Unlike structural acceleration methods that rely on spatial data structures for culling, these techniques operate primarily on the rendering pipeline and sampling parameters, often trading visual fidelity for frame rate stability [S1, unofficial]. They are distinct from sibling methods like [Occlusion Clipping](990-occlusion-clipping-with-orthogonal-opacity-map.md) or [Acceleration Structure Rebuild](3266-acceleration-structure-rebuild.md) by focusing on resolution, temporal coherence, and step-size adaptation rather than geometric traversal optimization [S1, unofficial].

## How it works
The method consists of four distinct optimization passes or parameters applied during the rendering loop.

**Inputs:**
*   Display resolution $(W, H)$.
*   Camera motion vectors.
*   Previous frame's color and depth buffers.
*   Volumetric density field.

**Outputs:**
*   Final high-resolution color buffer.

**Data Structures:**
*   **Low-Resolution Buffer:** A render target at $W/2 \times H/2$ or $W/4 \times H/4$ storing intermediate color and depth.
*   **History Buffer:** Stores reprojected color and depth from previous frames.
*   **Noise Pattern:** A 2D blue noise texture used for jittering.

**Steps:**

1.  **Quarter-Resolution Raymarching:**
    Execute the raymarching compute shader at half or quarter display resolution. For each pixel in the low-resolution grid, cast a ray and accumulate color $C$ and depth $D$ using standard front-to-back compositing [S1, unofficial].
    $$ C_{low} = \sum_{i} C_i \cdot \alpha_i \cdot \prod_{j=0}^{i-1} (1 - \alpha_j) $$
    $$ D_{low} = \text{depth at final sample} $$

2.  **Adaptive Stepping:**
    During the raymarching loop, dynamically adjust the step size $\Delta t$.
    *   If the current density $\rho(t) \approx 0$, double the step size: $\Delta t_{new} = 2 \cdot \Delta t_{old}$.
    *   If $\rho(t) > \epsilon$ (non-zero density), immediately revert to the base fine step size.
    This reduces evaluations in empty space while maintaining detail in dense regions [S1, unofficial].

3.  **Temporal Reprojection:**
    *   Jitter the ray step offsets in each frame using a 2D blue noise pattern to decorrelate temporal aliasing.
    *   Use camera motion vectors to reproject the low-resolution color and depth from the previous frame into the current frame's coordinate space.
    *   Blend the reprojected history with the current low-resolution result to stabilize the image over time [S1, unofficial].

4.  **Bilateral Upsampling:**
    Upscale the temporally stabilized low-resolution result to full display resolution $(W, H)$.
    *   Apply a depth-aware bilateral filter.
    *   The filter weights neighbors based on spatial distance, color difference, and depth difference.
    *   This prevents edge bleeding against terrain or other geometry by respecting depth discontinuities [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_frame(camera, prev_color, prev_depth, noise_pattern):
    # 1. Quarter-Resolution Raymarching with Adaptive Stepping
    low_res_w = display_w // 2
    low_res_h = display_h // 2
    low_res_color = zeros((low_res_h, low_res_w, 3))
    low_res_depth = zeros((low_res_h, low_res_w))
    
    base_step = 0.1
    for y in range(0, low_res_h):
        for x in range(0, low_res_w):
            ray = camera.get_ray(x * 2, y * 2) # Sample center of 2x2 block
            step = base_step
            acc_color = 0.0
            acc_alpha = 0.0
            t = ray.origin
            
            while t < ray.max_dist and acc_alpha < 0.99:
                density = sample_volume(ray.origin + ray.direction * t)
                
                # Adaptive Stepping
                if density < 1e-4:
                    step *= 2.0
                else:
                    step = base_step
                
                color_sample, alpha_sample = evaluate_voxel(ray.origin + ray.direction * t)
                
                # Front-to-back compositing
                contribution = color_sample * alpha_sample * (1.0 - acc_alpha)
                acc_color += contribution
                acc_alpha += alpha_sample * (1.0 - acc_alpha)
                
                t += step
            
            low_res_color[y, x] = acc_color
            low_res_depth[y, x] = t

    # 2. Temporal Reprojection
    reprojected_color = reproject(prev_color, prev_depth, camera.motion_vectors)
    # Blend current low-res with reprojected history
    stabilized_color = blend(low_res_color, reprojected_color, alpha=0.5)

    # 3. Bilateral Upsampling
    full_res_color = bilateral_upsample(stabilized_color, low_res_depth, display_w, display_h)
    
    return full_res_color
```

## Operating characteristics
*   **Time Complexity:** Reduces the number of raymarching samples by a factor of 4 (due to quarter resolution) and further reduces samples via adaptive stepping in empty space [S1, unofficial].
*   **Memory Complexity:** Requires storage for low-resolution buffers, history buffers, and motion vectors. The bilateral filter requires access to depth and color neighbors [S1, unofficial].
*   **Strengths:** Significant performance gains on hardware with limited compute throughput; effective for large, sparse volumes like clouds [S1, unofficial].
*   **Limitations:** Introduces temporal flickering if reprojection fails; edge bleeding if depth-awareness is ignored; loss of high-frequency detail due to downsampling [S1, unofficial].
*   **When to use:** Real-time rendering of volumetric clouds or smoke where high resolution is not strictly required for every pixel [S1, unofficial].

## Background
These techniques were popularized in the context of real-time volumetric cloud rendering in OpenGL, specifically in the article "Volumetric Clouds with 3D Perlin-Worley Noise in OpenGL" [S1, unofficial]. The approach combines established real-time rendering tricks (temporal reprojection, bilateral filtering) with volume-specific adaptations (adaptive stepping) to achieve interactive frame rates [S1, unofficial].

## Sources
[S1] Volumetric Clouds with 3D Perlin-Worley Noise in OpenGL, https://salivity.github.io/opengl/article/volumetric-clouds-with-3-d-perlin-worley-noise-in-open-gl, fetched 2026-10-29 (unofficial)
