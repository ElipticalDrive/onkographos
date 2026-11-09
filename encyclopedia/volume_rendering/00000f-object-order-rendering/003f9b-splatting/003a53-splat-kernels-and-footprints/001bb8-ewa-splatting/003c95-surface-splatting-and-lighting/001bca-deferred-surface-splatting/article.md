# Deferred Surface Splatting

Also called: EWA surface splatting with deferred shading, Finalization Shader, Lighting Pass, Point Cloud Splatting, Point cloud splatting to G-buffer, Surface Splatting Finalization Pass

## Overview
Deferred Surface Splatting is a rendering technique that converts unstructured point clouds into continuous, shaded surfaces by projecting points as splats into a deferred rendering pipeline [S2, unofficial]. It solves the aliasing and transparency artifacts inherent in raw point clouds by assigning each sample an extent drawn as a small oriented primitive (a splat) carrying a radially decreasing reconstruction kernel [S2, unofficial]. Unlike forward splatting methods that apply lighting during rasterization, this method decouples geometric projection from lighting calculation, accumulating surface attributes (depth, normals, color/weight) into a G-buffer and applying lighting in a final screen-space pass [S2, unofficial]. This approach efficiently handles heavy overlap and multiple light sources without re-projecting geometry [G].

## How it works
The method operates in three distinct phases: depth visibility, attribute accumulation, and deferred lighting resolve. It builds on the surface splatting primitives defined in [Surface Splatting and Lighting](9275-surface-splatting-and-lighting.md).

### Inputs and Outputs
*   **Input:** An unstructured point cloud (positions, colors, optional normals) and camera parameters (view-projection matrix, viewport dimensions).
*   **Output:** A final shaded image.
*   **Intermediate Data:** A G-buffer consisting of three textures:
    1.  **Depth:** `GL_DEPTH_COMPONENT32F` storing the front-surface depth of the visible splats.
    2.  **Normals:** `GL_RGBA32F` storing view-space surface normals (Alpha channel is unused but allocated) [S2, unofficial].
    3.  **Color/Weight:** `GL_RGBA32F` storing the weighted color `(R, G, B, Weight)` [S2, unofficial].

### Data Structures
The implementation uses OpenGL Uniform Blocks (`std140`, column-major) to pass parameters to shaders [S1, unofficial]:

1.  **`Camera` Block:**
    *   `mat4 modelview_matrix`
    *   `mat4 modelview_matrix_it` (Inverse Transpose)
    *   `mat4 projection_matrix`
2.  **`Raycast` Block:**
    *   `mat4 projection_matrix_inv`
    *   `vec4 viewport` (width, height, inv_width, inv_height)
3.  **`Parameter` Block:**
    *   `vec3 material_color`
    *   `float material_shininess`
    *   `float radius_scale`
    *   `float ewa_radius`
    *   `float epsilon` (Depth slab thickness)

### Steps

1.  **Initialization:**
    *   Load and compile shaders with compile-time defines: `SMOOTH` (1 for smooth shading, 0 for flat), `MULTISAMPLING` (1 if enabled), `EWA_FILTER`, `BACKFACE_CULLING`, `VISIBILITY_PASS`, `COLOR_MATERIAL` [S1, unofficial].
    *   Initialize OpenGL state: Enable `GL_DEPTH_TEST` (`GL_LESS`), `GL_VERTEX_PROGRAM_POINT_SIZE`, set point sprite origin to `GL_LOWER_LEFT` [S2, unofficial].

2.  **Depth Pass (Visibility):**
    *   Bind the G-Buffer. Set color mask to `false` for RGB channels (only depth is written).
    *   Draw the point cloud using the depth shader (`splat.vert`, `splat_depth.frag`).
    *   Each splat is rendered as a sphere impostor. The fragment shader calculates the analytic front-surface depth of a sphere:
        $$z = z_{center} - \sqrt{r^2 - x^2 - y^2}$$
    *   This depth is pushed back by $\varepsilon$ before writing to the depth buffer, creating a thin depth slab around the visible surface [S2, unofficial].
    *   Uniforms passed: `view_proj_matrix`, `fov_y`, `viewport_size`, `epsilon`, `particle_radius`, `z_near`, `z_far` [S2, unofficial].

3.  **Attribute Blend Pass (G-Buffer Accumulation):**
    *   Bind the G-Buffer. Bind the depth texture from Pass 1 to `GL_TEXTURE0`.
    *   Set color mask to `true` for all channels. Disable depth mask and depth test. Enable blending with `GL_ONE, GL_ONE` (pure additive) [S2, unofficial].
    *   Draw the point cloud using the attribute shader (`splat.vert`, `splat_attributes.frag`).
    *   For each fragment, compare its sphere depth against the depth texture from Pass 1. Discard fragments falling behind the $\varepsilon$ slab [S2, unofficial].
    *   Surviving fragments accumulate attributes. The weight $w$ is the product of radial and depth falloffs:
        $$w_{radial} = 1 - \frac{d}{r}$$
        $$w_{depth} = \frac{z_{surface} - z_{frag}}{\varepsilon}$$
        $$w = w_{radial} \cdot w_{depth}$$
    *   Where $d$ is the distance from the splat center in screen space, $r$ is the splat radius, $z_{surface}$ is the depth from the depth pass, and $z_{frag}$ is the current fragment's depth [S2, unofficial].
    *   Accumulate `Color * w` into the Color/Weight attachment and `Normal * w` into the Normal attachment [S2, unofficial].

4.  **Deferred Lighting Resolve (Finalization):**
    *   Bind the default framebuffer. Clear the color buffer.
    *   Draw a full-screen quad using the finalization shader (`finalization_vs_glsl`, `finalization_fs_glsl`, `lighting_glsl`) [S1, unofficial].
    *   The shader reads the G-Buffer textures. If `MULTISAMPLING` is defined, loop $i$ from 0 to 3, fetching texels using `texelFetch` with integer coordinates derived from `textureSize` and `In.texture_uv` [S1, unofficial]. Otherwise, use `texture` with `In.texture_uv` [S1, unofficial].
    *   Initialize accumulator `res = vec4(0.0)` [S1, unofficial].
    *   For each sample (or single sample if not multisampling):
        *   Fetch `pixel` from Color/Weight texture.
        *   If `SMOOTH` is defined:
            *   Fetch `normal` from Normal texture (normalized).
            *   Fetch `depth` from Depth texture.
            *   If `pixel.a > 0.0`:
                *   Reconstruct NDC position `p_ndc`: $x,y$ from viewport and `gl_FragCoord`, $z$ from depth and `gl_DepthRange`, $w=1.0$.
                *   Transform `p_ndc` to eye space `v_eye` using `projection_matrix_inv` and divide by $w$.
                *   Compute lighting: `lighting(normal, v_eye.xyz, pixel.rgb / pixel.a, material_shininess)`.
                *   Add to `res`: `vec4(lighting_result, 1.0)`.
            *   Else:
                *   Add `vec4(1.0, 1.0, 1.0, 1.0)` to `res` [S1, unofficial].
        *   If `SMOOTH` is not defined:
            *   If `pixel.a > 0.0`:
                *   Add `vec4(pixel.rgb / pixel.a, 1.0)` to `res` (premultiplied color) [S1, unofficial].
            *   Else:
                *   Add `vec4(1.0, 1.0, 1.0, 1.0)` to `res` [S1, unofficial].
    *   After processing all samples:
        *   If `MULTISAMPLING`: Output `frag_color = sqrt(res / 4.0)` [S1, unofficial].
        *   Else: Output `frag_color = sqrt(res)` [S1, unofficial].

### Parameters
*   `epsilon`: Float parameter calculated as `_renderingParams->_epsilonMultiplier * _renderingParams->_pointSize`. Controls the depth slab thickness [S2, unofficial].
*   `particle_radius`: Float representing the splat size [S2, unofficial].
*   `material_shininess`: Float for specular lighting exponent [S1, unofficial].
*   `material_color`: Vec3 for base material color [S1, unofficial].

### Edge Cases and Pitfalls
*   **Normal Buffer Format:** The implementation uses `GL_RGBA32F` for the normal buffer instead of `GL_RGB32F` due to hardware compatibility issues ("RGB doesn't work on my laptop"). The Alpha channel is allocated but unused [S2, unofficial].
*   **Raw Points Mode:** If `_renderingParams->_rawPoints` is true, the splatting pipeline is skipped. The scene is drawn directly using `GL_POINTS` with additive blending disabled and depth test enabled, using the same vertex shader for consistent screen-space sizing [S2, unofficial].
*   **Precision:** The depth slab approach relies on precise depth comparisons. Insufficient `epsilon` may cause holes; excessive `epsilon` may cause z-fighting or incorrect normal reconstruction [S2, unofficial].

## Subtopics

### [Deferred Lighting Resolve](1401-deferred-lighting-resolve.md)
Deferred Lighting Resolve is the final shading pass in a deferred surface splatting pipeline. It reconstructs the final pixel color by applying a lighting model (ambient, diffuse, specular) to the weighted color and surface normals accumulated in the G-buffers during the splatting phase. It differs from forward rendering by decoupling the geometric projection (splatting) from the lighting calculation, allowing multiple light sources to be applied efficiently in screen space without re-projecting geometry [G].

### [Point-Based Rendering](2827-point-based-rendering.md)
A method to approximate an implicit surface with nonpolygonal proxy geometry (points, billboards, or volumetric representation) when the underlying discrete geometry is sufficiently dense. Deferred Surface Splatting is a specific implementation within this family that uses deferred shading for lighting.

### Weighted Splatting with Depth Proximity
A rendering technique where point clouds are rendered as spheres (splats) in a multi-pass pipeline. It accumulates weighted attributes (color, normal) into a G-buffer to handle heavy overlap and reconstruct surface properties, deferring lighting to a final resolve pass [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N + P)$, where $N$ is the number of points and $P$ is the number of pixels. The splatting pass is proportional to the number of points (rasterizing splats), and the resolve pass is proportional to the screen resolution.
*   **Memory Complexity:** $O(P)$ for the G-buffer (Depth, Normal, Color/Weight textures).
*   **Performance:** Efficient for large point clouds because lighting is computed once per pixel in screen space, regardless of the number of overlapping splats [S2, unofficial].
*   **Strengths:** Handles heavy overlap gracefully; supports smooth shading; decouples geometry from lighting; allows easy visualization of intermediate buffers (albedo, normals, depth) [S2, unofficial].
*   **Limitations:** Requires multiple render passes; G-buffer memory usage increases with screen resolution; depth slab approach may introduce artifacts if `epsilon` is not tuned correctly [S2, unofficial].
*   **When to use:** Real-time visualization of large, unstructured point clouds where surface continuity and lighting quality are important.
*   **When not to use:** When memory is extremely constrained (due to G-buffer) or when the point cloud is too sparse to benefit from surface reconstruction.

## Background
The technique is based on the Surface Splatting formulation by Zwicker, Pfister, van Baar, and Gross (SIGGRAPH 2001) and the multi-pass GPU pipeline described by Botsch, Hornung, Zwicker, and Kobbelt (PBG 2005) [S2, unofficial]. It extends these concepts by integrating a deferred shading pipeline to handle lighting efficiently. The specific implementation details provided in the sources are from unofficial repositories: `sebastianlipponer/surface_splatting` [S1, unofficial] and `arminkz/PointCloudRendering` [S2, unofficial].

## Sources
[S1] sebastianlipponer/surface_splatting, https://github.com/sebastianlipponer/surface_splatting, fetched 2026-10-28 (unofficial)
[S2] arminkz/PointCloudRendering, https://github.com/arminkz/PointCloudRendering, fetched 2026-10-28 (unofficial)
