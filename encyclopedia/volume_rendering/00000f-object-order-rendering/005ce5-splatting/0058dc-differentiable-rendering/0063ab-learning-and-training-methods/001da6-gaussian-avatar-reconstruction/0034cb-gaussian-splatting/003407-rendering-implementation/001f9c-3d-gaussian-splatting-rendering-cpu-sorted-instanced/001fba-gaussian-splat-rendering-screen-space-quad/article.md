# Gaussian Splat Rendering (Screen-Space Quad)

Also called: Gaussian Splat Rendering Pipeline, RenderGaussianSplats, Splatv Render Program, VideoRenderProgram, WebGL Instanced Gaussian Splat Rendering

## Overview
Gaussian Splat Rendering (Screen-Space Quad) is a display-only rendering technique that projects 3D Gaussian distributions into 2D screen-space quads, sorts them by camera-space depth, and renders them using instanced drawing with standard alpha blending [S2, S3, unofficial]. It solves the problem of visualizing large-scale, photorealistic 3D scenes reconstructed via Gaussian Splatting on client-side hardware without requiring neural network inference or server-side ray tracing [S1, S3, unofficial]. It sets itself apart from its siblings by utilizing a CPU-side depth sorting algorithm (often offloaded to a background worker) to resolve the non-commutative nature of alpha blending, whereas the parent [3D Gaussian Splatting Rendering (CPU-Sorted, Instanced)](2058-3d-gaussian-splatting-rendering-cpu-sorted-instanced.md) describes the general projection and quad-drawing mechanism without specifying the sorting implementation [S2, S3, unofficial]. Unlike [Tiled 2D Gaussian Splatting Rasterization](2061-tiled-2d-gaussian-splatting-rasterization.md), this method relies on explicit per-splat sorting rather than tile-based accumulation [S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A depth-sorted index buffer (Uint32Array) mapping instance IDs to splat indices; view-space projection data (position, axes, color, opacity) stored in textures or buffers; camera matrices (Model-View, Projection); screen resolution parameters.
*   **Outputs:** A composited color buffer with pre-multiplied alpha blending.

**Data Structures:**
*   **Vertex Buffer:** Contains 8 floats representing a unit quad: `[-2, -2, 2, -2, 2, 2, -2, 2]` [S3, unofficial].
*   **Index Buffer (Depth Index):** A `Uint32Array` containing the depth-sorted splat indices, updated via worker messages [S3, unofficial].
*   **Splat Texture:** Created with `gl.RGBA32UI` internal format, `gl.RGBA_INTEGER` format, and `gl.UNSIGNED_INT` type, sized to the render data width and height [S3, unofficial]. For animated or complex variants, additional textures (e.g., `splatRotation` as RGBA16F) may be used [S5, unofficial].
*   **View Data:** Contains per-splat attributes including position, rotation (quaternion), scale, and color (SH degree 0) [S4, S5, unofficial].

**Numbered Steps:**
1.  **Initialization:** Create the vertex buffer with quad coordinates and the index buffer for sorted splat indices [S3, unofficial]. Initialize the scene, camera, and render data [S3, unofficial].
2.  **Sorting (CPU/Worker):** Send initial sort data (positions, dummy transforms, transform indices) to a background worker [S3, unofficial]. On each render frame, update the camera and send the view-projection matrix to the worker [S3, unofficial]. The worker computes the depth-sorted order and returns the `depthIndex` buffer [S3, unofficial].
3.  **Setup Blending:** Configure the graphics pipeline for alpha blending: `gl.blendFuncSeparate(gl.ONE_MINUS_DST_ALPHA, gl.ONE, gl.ONE_MINUS_DST_ALPHA, gl.ONE)` [S3, unofficial].
4.  **Bind Attributes:** Bind the index buffer using `gl.vertexAttribIPointer` with a divisor of 1 to enable instanced drawing [S3, unofficial].
5.  **Vertex Shader Execution:**
    *   Map the instance ID to the splat index using the order buffer: `instID = _OrderBuffer[instID]` [S2, unofficial].
    *   Load view data for the splat: `SplatViewData view = _SplatViewData[instID]` [S2, unofficial].
    *   Check if the splat is behind the camera (`centerClipPos.w <= 0`). If so, output a NaN vertex (`asfloat(0x7fc00000)`) to discard the primitive [S2, unofficial].
    *   Transform the voxel position to view space: `vec4 voxelPosViewSpace = MV * gl_in[0].gl_Position` [S1, unofficial].
    *   Transform to clip space and normalize: `vec4 voxelPosClipSpace = P * voxelPosViewSpace; voxelPosClipSpace /= voxelPosClipSpace.w` [S1, unofficial].
    *   Perform clip-space frustum culling. Calculate the voxel offset: $voxelOffset = splatSize \times (scale/gridLength) \times \sqrt{2}$ [S1, unofficial]. Calculate the half-edge length in clip space: $voxelHalfEdgeLengthClipSpace = distance(voxelPosClipSpace, voxelRightEdgeMidPointClipSpace)$ [S1, unofficial]. Define the frustum culling border: $voxelFrustumCullingBorder = 1 + voxelHalfEdgeLengthClipSpace$ [S1, unofficial]. Discard if `voxelPosClipSpace.x` or `y` is outside `[-voxelFrustumCullingBorder, voxelFrustumCullingBorder]` [S1, unofficial].
    *   Calculate the quad position in clip space. For WebGL implementations, map instance ID to quad corners: `quadPos = float2(idx&1, (idx>>1)&1) * 2.0 - 1.0; quadPos *= 2.0` [S2, unofficial]. Calculate screen delta: `deltaScreenPos = (quadPos.x * view.axis1 + quadPos.y * view.axis2) * 2 / _ScreenParams.xy` [S2, unofficial]. Final vertex: `o.vertex.xy += deltaScreenPos * centerClipPos.w` [S2, unofficial].
    *   (Optional) Handle selection bits if `_SplatBitsValid` is true, marking selected splats with specific color values [S2, unofficial].
6.  **Fragment Shader Execution:**
    *   Calculate the Gaussian falloff: `power = -dot(i.pos, i.pos); alpha = exp(power)` [S2, unofficial].
    *   If not selected, saturate alpha: `alpha = saturate(alpha * i.col.a)` [S2, unofficial].
    *   If selected, apply magenta outline logic: if `alpha > 7.0/255.0`, set `alpha = 1` and `i.col.rgb = selectedColor` if `alpha < 10.0/255.0`; otherwise `alpha = saturate(alpha + 0.3)` and tint `i.col.rgb = lerp(i.col.rgb, selectedColor, 0.5)` [S2, unofficial].
    *   Discard fragment if `alpha < 1.0/255.0` [S2, unofficial].
    *   Output final color: `res = half4(i.col.rgb * alpha, alpha)` [S2, unofficial].
7.  **Draw Call:** Call `gl.drawArraysInstanced(gl.TRIANGLE_FAN, 0, 4, vertexCount)` [S3, unofficial].

**Formulas:**
*   Voxel Offset: $voxelOffset = splatSize \times \frac{scale}{gridLength} \times \sqrt{2}$ [S1, unofficial].
*   Frustum Culling Border: $voxelFrustumCullingBorder = 1 + distance(voxelPosClipSpace, voxelRightEdgeMidPointClipSpace)$ [S1, unofficial].
*   Gaussian Falloff: $\alpha = e^{-d^T \Sigma_2^{-1} d}$ (evaluated as `exp(-dot(i.pos, i.pos))` in simplified implementations) [S2, S5, unofficial].

**Parameters:**
*   `_ScreenParams.xy`: Screen resolution in pixels, used to normalize the screen-space delta of the quad [S2, unofficial].
*   `u_focal`: WebGLUniformLocation for camera focal lengths $f_x, f_y$ [S3, unofficial].
*   `splatCount`: The number of splats to render, coerced to the range `0..splatCloud.count`. Defaults to the full cloud count. Used for LOD or reveal effects [S4, unofficial].
*   `u_time`: WebGLUniformLocation for animation time, typically set to `Math.sin(Date.now() / 1000) / 2 + 1 / 2` [S3, unofficial].
*   `u_viewport`: WebGLUniformLocation for canvas dimensions `[canvas.width, canvas.height]` [S3, unofficial].

**Edge Cases and Pitfalls:**
*   **Behind Camera:** Splats with clip-space $w \le 0$ must be discarded to prevent rendering artifacts. This is handled by outputting a NaN vertex in the vertex shader [S2, unofficial].
*   **Multiple Objects:** Multiple Splatv objects are not supported in some implementations (e.g., gsplat.js), triggering a console warning [S3, unofficial].
*   **Worker Initialization:** The background worker must be initialized before rendering; checks for `renderer.renderProgram.worker === null` and returns an error if null [S3, unofficial].
*   **Tone Mapping:** The material cancels the View's tone mapping so colors are displayed as stored in the splat data (SH degree 0 only) [S4, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Vertex Shader Logic (Simplified WebGL/Unity Hybrid)
def vertex_shader(instance_id, quad_corner_index):
    # 1. Map instance to splat index
    splat_index = order_buffer[instance_id]
    
    # 2. Load view data
    view_data = splat_view_data[splat_index]
    
    # 3. Check behind camera
    center_clip_w = view_data.center_clip_pos.w
    if center_clip_w <= 0:
        return NaN_VERTEX # Discard primitive
    
    # 4. Transform to view/clip space
    voxel_pos_view = mv_matrix * view_data.position
    voxel_pos_clip = projection_matrix * voxel_pos_view
    voxel_pos_clip /= voxel_pos_clip.w
    
    # 5. Frustum Culling
    voxel_offset = splat_size * (view_data.scale / grid_length) * sqrt(2)
    right_edge_mid = projection_matrix * (voxel_pos_view.xy + vec2(0.5, 0) * voxel_offset)
    right_edge_mid /= right_edge_mid.w
    half_edge_len = distance(voxel_pos_clip, right_edge_mid)
    culling_border = 1.0 + half_edge_len
    
    if abs(voxel_pos_clip.x) > culling_border or abs(voxel_pos_clip.y) > culling_border:
        return NaN_VERTEX # Discard primitive
    
    # 6. Calculate Quad Position
    # Map quad_corner_index (0-3) to [-2, -2], [2, -2], [2, 2], [-2, 2]
    quad_x = (quad_corner_index & 1) * 2.0 - 1.0
    quad_y = ((quad_corner_index >> 1) & 1) * 2.0 - 1.0
    quad_pos = vec2(quad_x, quad_y) * 2.0
    
    # Screen delta calculation
    screen_delta = (quad_pos.x * view_data.axis1 + quad_pos.y * view_data.axis2) * 2.0 / screen_params.xy
    final_vertex = voxel_pos_clip.xy + screen_delta * center_clip_w
    
    return final_vertex

# Fragment Shader Logic
def fragment_shader(interpolated_pos, interpolated_color):
    # Calculate Gaussian falloff
    power = -dot(interpolated_pos, interpolated_pos)
    alpha = exp(power)
    
    # Handle selection
    if interpolated_color.a < 0: # Selected
        if alpha > 7.0/255.0:
            if alpha < 10.0/255.0:
                alpha = 1.0
                rgb = selected_color # Magenta
            else:
                alpha = saturate(alpha + 0.3)
                rgb = lerp(interpolated_color.rgb, selected_color, 0.5)
        else:
            discard_fragment()
    else: # Not selected
        alpha = saturate(alpha * interpolated_color.a)
        rgb = interpolated_color.rgb
        
    if alpha < 1.0/255.0:
        discard_fragment()
        
    return half4(rgb * alpha, alpha)
```

## Subtopics

### [WebGL Gaussian Splatting](2084-webgl-gaussian-splatting.md)
WebGL Gaussian Splatting is a real-time rendering technique for 3D Gaussian Splatting (3DGS) implemented in web browsers using WebGL 1.0, with variants utilizing the Three.js library or running without external dependencies. It solves the problem of displaying large-scale, photorealistic 3D scenes reconstructed via Gaussian Splatting on client-side hardware without requiring neural network inference or server-side ray tracing. It sets itself apart from its siblings by utilizing a CPU-side depth sorting algorithm (often offloaded to a Web Worker) to resolve the non-commutative nature of alpha blending, whereas the parent [Gaussian Splat Rendering (Screen-Space Quad)](2075-gaussian-splat-rendering-screen-space-quad.md) describes the general projection and quad-drawing mechanism without specifying the sorting implementation. Unlike [Animated 3D Gaussian Splatting (Splatv rendering)](7758-animated-3d-gaussian-splatting-splatv-rendering.md), this method typically omits spherical harmonics to reduce file size and memory footprint for web delivery.

### [Debug Splat Point Rendering](2101-debug-splat-point-rendering.md)
Debug Splat Point Rendering is a diagnostic visualization technique that renders individual 3D Gaussian splats as small screen-space quads centered at their projected positions. Unlike the production rendering path which draws full elliptical splats for opacity blending, this method isolates the center location and basic attributes of each splat, allowing developers to inspect splat distribution, density, and attribute correctness. It serves as a lightweight alternative to full splat rendering for verifying the correctness of the splat data pipeline and camera projection logic.

### [Gaussian Splat Fragment Shader](3258-gaussian-splat-fragment-shader.md)
The Gaussian Splat Fragment Shader evaluates the 2D Gaussian probability density function (PDF) at each fragment of a screen-space quad to determine splat coverage and modulate fragment color. It composites the new splat over the existing framebuffer color using pre-multiplied alpha blending. It is distinguished from its parent [Gaussian Splat Rendering (Screen-Space Quad)](2075-gaussian-splat-rendering-screen-space-quad.md) by performing the per-pixel alpha evaluation and blending logic directly in the fragment shader, handling transparency via alpha cutoffs and discarding fragments behind the camera or with negligible alpha.

### [Gaussian Splat Shader Property Configuration](3596-gaussian-splat-shader-property-configuration.md)
This method configures the GPU render pass by populating a material property block (or uniform buffer) with splat count, spherical harmonic (SH) degree, brightness, scale, antialiasing settings, transform matrices, and optional skinning or volume lighting data. It solves the problem of efficiently passing per-renderer and global state to the fragment shader without causing state overwrites in multi-camera setups or shader compilation errors when optional features (like skinning) are disabled. It differs from standard mesh rendering by explicitly handling the decoupling of jittered projections and ensuring valid sampled resources even for unused features.

### [Animated 3D Gaussian Splatting (Splatv rendering)](7758-animated-3d-gaussian-splatting-splatv-rendering.md)
Animated 3D Gaussian Splatting (Splatv rendering) is a GPU-accelerated rendering method for time-varying 3D Gaussian splats that computes time-dependent position, rotation, and scale deltas from packed texture data to enable smooth animation without per-frame CPU updates. It differs from static 3D Gaussian Splatting by evaluating polynomial motion deltas and applying temporal opacity decay in the vertex and fragment shaders, building on the screen-space quad rendering of [Gaussian Splat Rendering (Screen-Space Quad)](2075-gaussian-splat-rendering-screen-space-quad.md).

### [Gaussian Splat URP Render Pass](17074-gaussian-splat-urp-render-pass.md)
A Unity Universal Render Pipeline (URP) `ScriptableRendererFeature` that renders pre-sorted 3D Gaussian splats into a temporary render target and composites the result onto the camera color buffer. It differs from raw shader implementations by integrating into the URP render graph, handling XR multipass separation, and managing the intermediate render target lifecycle automatically.

### Anisotropic 3D Gaussian Splatting Rendering
A rendering method for Gaussian splats that computes anisotropic covariance matrices to handle perspective distortion and view-dependent scaling, improving edge crispness and reducing haze compared to isotropic approximations [S5, unofficial]. In the vertex shader, it computes the 3D covariance matrix $\Sigma_3 = R \cdot S^2 \cdot R^T$, transforms it to view space, computes the perspective Jacobian $J$, applies a low-pass filter ($+0.3 \text{ px}^2$) to $\Sigma_2$, and eigen-decomposes it to place quad corners at $\text{centre} \pm 3\sqrt{\lambda}v$ [S5, unofficial]. It requires a third RGBA16F texture (`splatRotation`) for quaternions [S5, unofficial].

### Isotropic Gaussian Splatting Rendering
A rendering technique for 3D Gaussian Splatting where each splat is rendered as an isotropic billboard (circle) using the maximum of the three axis scales, ignoring the rotation for the shape geometry [S5, unofficial]. This results in a hazy, over-bright appearance (the 'P1 isotropic artefact') [S5, unofficial]. It fetches per-splat data from two RGBA16F textures, determines billboard size using max axis scale, and renders as hardware-instanced camera-facing quads with batches of at most 65535 instances per draw call [S5, unofficial]. Sorting complexity is $O(n \log n)$ on the CPU [S5, unofficial].

### Anisotropic Gaussian Splatting Projection
A rendering technique that projects 3D Gaussian ellipses onto the screen space by computing a 2x2 covariance matrix, allowing for accurate anisotropic rendering of splats as ellipses rather than circles [S5, unofficial]. It computes $\Sigma = R \cdot S \cdot S^T \cdot R^T$, projects it using the view Jacobian, eigen-decomposes to find major/minor axes, and places quad corners on these axes [S5, unofficial]. It differs from isotropic rendering by using the full covariance matrix, reducing overdraw and providing a sharper image [S5, unofficial]. It requires a third RGBA16F texture for rotation/scale data [S5, unofficial].

### Gaussian Splat Vertex Shader
A vertex shader for rendering Gaussian splats as quads [S6, unofficial]. It takes `SplatCount` (uint32), `DebugMode` (uint32: 0=off, 1=cluster colors), and `EnableNanite` (uint32: 1=Nanite enabled) as parameters [S6, unofficial]. It inputs `ViewDataBuffer` (StructuredBuffer<FGaussianSplatViewData>) and `SortKeysBuffer` (StructuredBuffer<uint>) [S6, unofficial].

## Operating characteristics
*   **Time Complexity:** Sorting is $O(n \log n)$ on the CPU [S5, unofficial]. Rendering is $O(n)$ for the draw call, but fragment processing is proportional to the number of overlapping pixels.
*   **Memory Complexity:** Requires storage for splat data (positions, rotations, scales, colors) and the sorted index buffer. For 233k splats, sorting involves ~5.6 MB of texture re-upload per frame [S5, unofficial].
*   **Measured Performance:** On mid-range phones, fill rate and overdraw dominate performance [S5, unofficial]. Anisotropic ellipses reduce overdraw compared to max-axis discs, offering a performance win [S5, unofficial]. The CPU sort is feasible up to ~300–500k splats; beyond that, LOD/pruning is needed [S5, unofficial].
*   **Strengths:** Simple implementation using standard graphics APIs (WebGL, Unity URP); leverages existing instanced drawing capabilities; effective for real-time web and mobile visualization [S1, S3, S4, unofficial].
*   **Limitations:** CPU sorting becomes a bottleneck at very high splat counts (>500k) [S5, unofficial]; isotropic variants suffer from visual artifacts (haze) [S5, unofficial]; multiple Splatv objects are not supported in some WebGL implementations [S3, unofficial].
*   **When to Use:** Real-time visualization of Gaussian Splatting scenes in web browsers (WebGL) or Unity URP environments where CPU sorting is acceptable [S1, S3, S4, unofficial].
*   **When Not to Use:** Scenes with millions of splats requiring real-time performance without LOD; platforms where CPU sorting is too slow or unavailable (e.g., pure GPU sorting required).

## Background
Gaussian Splat Rendering (Screen-Space Quad) is a rendering technique identical in geometry generation to Circle Splat, used for Gaussian Splat visualization, projecting voxels as 2D quads [S1, unofficial]. It is used in `gsplat.js` for real-time WebGL rendering of Gaussian splats [S3, unofficial]. The method builds on the general 3D Gaussian Splatting rendering pipeline, adapting it for screen-space quad rendering with CPU-side sorting [S2, S3, unofficial]. It is commonly used in web-based viewers (e.g., gsplat.js) and Unity URP implementations [S3, S4, unofficial].

## Sources
[S1] mmalesadev/svo_renderer, https://github.com/mmalesadev/svo_renderer, fetched 2026-10-26 (unofficial)
[S2] aras-p/UnityGaussianSplatting, https://github.com/aras-p/UnityGaussianSplatting, fetched 2026-10-26 (unofficial)
[S3] huggingface/gsplat.js, https://github.com/huggingface/gsplat.js, fetched 2026-10-27 (unofficial)
[S4] feat(sceneview): sort splats for the scene camera by default and add …, https://github.com/sceneview/sceneview/commit/5f2a4dd1f683ddb444669be54b78ce7913b55b70, fetched 2026-10-30 (unofficial)
[S5] Explore: Gaussian splatting, OpenUSD, AI world generation, volumetric video — study, https://github.com/sceneview/sceneview/issues/4023, fetched 2026-10-27 (unofficial)
[S6] YuanBaoSMadLab/OpenSplat4D, https://github.com/YuanBaoSMadLab/OpenSplat4D, fetched 2026-10-27 (unofficial)
