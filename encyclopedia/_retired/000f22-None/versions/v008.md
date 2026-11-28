# Deferred Surface Splatting

Also called: EWA surface splatting with deferred shading, Finalization Shader, Lighting Pass, Point Cloud Splatting, Point cloud splatting to G-buffer, Surface Splatting Finalization Pass

## Overview
Deferred Surface Splatting is a rendering technique that converts unstructured point clouds into continuous, shaded surfaces by projecting points as splats into a deferred rendering pipeline [S2, unofficial]. It solves the problem of sparse dot cloud aliasing and transparency artifacts by assigning each point sample an extent drawn as a small oriented primitive (a splat) carrying a radially decreasing reconstruction kernel [S2, unofficial]. Unlike [Phong Splatting](2802-phong-splatting.md), which computes lighting per-splat during rasterization, Deferred Surface Splatting decouples geometric projection from lighting calculation, allowing multiple light sources to be applied efficiently in screen space without re-projecting geometry [G]. It builds on [Surface Splatting and Lighting](9275-surface-splatting-and-lighting.md) by deferring the illumination step to a final pass over accumulated G-buffer attributes.

## How it works
**Inputs and Outputs:**
*   **Input:** A set of point cloud samples $P = \{p_i\}$, where each sample has position $\mathbf{p}$, color $\mathbf{c}$, and optionally normal $\mathbf{n}$ [S2, unofficial].
*   **Output:** A final image with continuous surface shading, depth, and normals reconstructed from the splat buffers [S1, S2, unofficial].

**Data Structures:**
The method uses a G-Buffer consisting of three attachments [S2, unofficial]:
1.  **Depth Attachment:** `GL_DEPTH_ATTACHMENT` using `GL_DEPTH_COMPONENT32F` texture. Stores front-surface depth from the visibility pass.
2.  **Normal Attachment:** `GL_COLOR_ATTACHMENT0` using `GL_RGBA32F` texture. Stores view-space normals. The Alpha channel is allocated but unused [S2, unofficial].
3.  **Color/Weight Attachment:** `GL_COLOR_ATTACHMENT1` using `GL_RGBA32F` texture. Stores $(R, G, B, W)$, where $R,G,B$ are the weight-normalized color and $W$ is the accumulated weight [S2, unofficial].

**Uniform Blocks (std140, column_major):**
*   `Camera`: `mat4 modelview_matrix`, `mat4 modelview_matrix_it`, `mat4 projection_matrix` [S1, unofficial].
*   `Raycast`: `mat4 projection_matrix_inv`, `vec4 viewport` [S1, unofficial].
*   `Parameter`: `vec3 material_color`, `float material_shininess`, `float radius_scale`, `float ewa_radius`, `float epsilon` [S1, unofficial].

**Parameters:**
*   `epsilon`: A float parameter calculated as `_renderingParams->_epsilonMultiplier * _renderingParams->_pointSize`. It defines the depth slab thickness for surface reconstruction [S2, unofficial].
*   `particle_radius`: The splat size, passed to vertex and fragment shaders [S2, unofficial].
*   `SMOOTH`: Boolean define (1 if smooth shading, 0 if flat) [S1, unofficial].
*   `MULTISAMPLING`: Boolean define (1 if multisampling is enabled) [S1, unofficial].

**Steps:**

1.  **Initialization:**
    *   Load shaders: `finalization_vs_glsl`, `finalization_fs_glsl`, `lighting_glsl` [S1, unofficial].
    *   Compile shaders with defines `SMOOTH` and `MULTISAMPLING` [S1, unofficial].
    *   Link program and bind uniform blocks: `Camera` to binding point 0, `Raycast` to 1, `Parameter` to 3 [S1, unofficial].
    *   Initialize OpenGL state: Enable depth test (`GL_LESS`), enable vertex program point size, set point sprite coordinate origin to `GL_LOWER_LEFT` [S2, unofficial].

2.  **Depth Pass (Visibility):**
    *   Bind G-Buffer. Set color mask to false for RGB (only write depth) [S2, unofficial].
    *   Draw scene using `_shader_depth` (`splat.vert`, `splat_depth.frag`) [S2, unofficial].
    *   Each splat is rasterized as a sphere impostor. Fragments outside the unit disk are discarded. The analytic front-surface depth is computed as:
        $$z = z_{center} - \sqrt{r^2 - x^2 - y^2}$$
    *   This depth is pushed back by $\varepsilon$ before being written to the depth buffer, creating a thin depth slab around the visible surface [S2, unofficial].
    *   Uniforms passed: `view_proj_matrix`, `fov_y`, `viewport_size`, `epsilon`, `particle_radius`, `z_near`, `z_far` [S2, unofficial].

3.  **Attribute Blend Pass (G-Buffer Accumulation):**
    *   Bind G-Buffer. Bind `_gDepthTex` to `GL_TEXTURE0` [S2, unofficial].
    *   Set color mask to true for all channels. Disable depth mask. Disable depth test. Enable blending with `GL_ONE, GL_ONE` (pure additive) [S2, unofficial].
    *   Draw scene using `_shader_attr` (`splat.vert`, `splat_attributes.frag`) [S2, unofficial].
    *   Each fragment compares its own sphere depth against the depth texture from Pass 1. It discards fragments that fall behind the $\varepsilon$ slab (i.e., $z_{frag} < z_{surface} - \varepsilon$) [S2, unofficial].
    *   Surviving fragments (those belonging to the same visible surface) are accumulated. The weight $w$ is the product of radial and depth falloffs:
        $$w_{radial} = 1 - \frac{d}{r}$$
        $$w_{depth} = \frac{z_{surface} - z_{frag}}{\varepsilon}$$
        $$w = w_{radial} \cdot w_{depth}$$
    *   Where $d$ is the distance from the splat center in screen space, and $r$ is the splat radius [S2, unofficial].
    *   Color and normal attributes are accumulated additively, weighted by $w$. The final G-buffer stores the sum of weighted attributes and the sum of weights [S2, unofficial].
    *   Uniforms passed: `view_proj_matrix`, `fov_y`, `viewport_size`, `epsilon`, `particle_radius`, `z_near`, `z_far` [S2, unofficial].

4.  **Rendering Pass (Finalization/Lighting):**
    *   Bind default framebuffer (0). Clear color buffer [S2, unofficial].
    *   Select shader based on display mode: `DEPTH` (reads `_gDepthTex`), `COLORS` (reads `_gColorTex`), `NORMALS` (reads `_gNormalTex`), or `FULL` (reads `_gNormalTex` and `_gColorTex`, applies lighting) [S2, unofficial].
    *   Draw a full-screen quad using `_quadVAO` [S2, unofficial].
    *   **Fragment Shader Logic:**
        *   Initialize result accumulator `res` to `vec4(0.0)` [S1, unofficial].
        *   If `MULTISAMPLING` is defined:
            *   Convert texture UVs to integer coordinates `itexture_uv` using `textureSize` and `In.texture_uv` [S1, unofficial].
            *   Loop `i` from 0 to 3:
                *   Fetch color pixel from `color_texture` using `texelFetch` with index `i` [S1, unofficial].
                *   If `SMOOTH` is defined:
                    *   Fetch normal from `normal_texture` (normalized) and depth from `depth_texture` using `texelFetch` with index `i` [S1, unofficial].
                *   If pixel alpha (weight) > 0.0:
                    *   Reconstruct NDC position `p_ndc`: $x,y$ from viewport and `gl_FragCoord`, $z$ from depth and `gl_DepthRange`, $w=1.0$ [S1, unofficial].
                    *   Transform `p_ndc` to eye space `v_eye` using `projection_matrix_inv` and divide by $w$ [S1, unofficial].
                    *   If `SMOOTH`: Add lighting result to `res`: `lighting(normal, v_eye.xyz, pixel.rgb / pixel.a, material_shininess)`, with alpha 1.0 [S1, unofficial].
                    *   If not `SMOOTH`: Add premultiplied color to `res`: `pixel.rgb / pixel.a` with alpha 1.0 [S1, unofficial].
                *   If pixel alpha <= 0.0: Add white `(1.0)` to `res` [S1, unofficial].
            *   Output `frag_color` as `sqrt(res / 4.0)` [S1, unofficial].
        *   If `MULTISAMPLING` is not defined:
            *   Fetch color pixel from `color_texture` using `texture` with `In.texture_uv` [S1, unofficial].
            *   If `SMOOTH` is defined:
                *   Fetch normal from `normal_texture` (normalized) and depth from `depth_texture` using `texture` [S1, unofficial].
            *   If pixel alpha > 0.0:
                *   Reconstruct NDC position `p_ndc` and transform to eye space `v_eye` as above [S1, unofficial].
                *   If `SMOOTH`: Add lighting result to `res`: `lighting(normal, v_eye.xyz, pixel.rgb / pixel.a, material_shininess)`, with alpha 1.0 [S1, unofficial].
                *   If not `SMOOTH`: Add premultiplied color to `res`: `pixel.rgb / pixel.a` with alpha 1.0 [S1, unofficial].
            *   If pixel alpha <= 0.0: Add white `(1.0)` to `res` [S1, unofficial].
            *   Output `frag_color` as `sqrt(res)` [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Normal Buffer Format:** The implementation uses `GL_RGBA32F` for the normal buffer instead of `GL_RGB32F` because `RGB` doesn't work on some laptops. The Alpha channel is allocated but not used for data [S2, unofficial].
*   **Raw Points Mode:** If `_renderingParams->_rawPoints` is true, the splatting pipeline is skipped. The scene is drawn directly to the screen using `_shader_points` (`splat.vert`, `points.frag`) with additive blending disabled and depth test enabled, using the same vertex shader to maintain consistent screen-space point sizing [S2, unofficial]. This avoids the overhead of splatting but results in hard disks without area or connectivity, causing aliasing and seeing-through artifacts [S2, unofficial].

## Subtopics

### [Deferred Lighting Resolve](1401-deferred-lighting-resolve.md)
Deferred Lighting Resolve is the final shading pass in a deferred surface splatting pipeline. It reconstructs the final pixel color by applying a lighting model (ambient, diffuse, specular) to the weighted color and surface normals accumulated in the G-buffers during the splatting phase. It differs from forward rendering by decoupling the geometric projection (splatting) from the lighting calculation, allowing multiple light sources to be applied efficiently in screen space without re-projecting geometry [G]. See [Deferred Lighting Resolve](1401-deferred-lighting-resolve.md).

### [Point-Based Rendering](2827-point-based-rendering.md)
Point-Based Rendering (PBR) is a rendering technique that approximates continuous surfaces or dense volumetric distributions using a set of discrete points (splats) rather than explicit polygonal meshes. It solves the problem of rendering large, unstructured point clouds or implicit surfaces efficiently by rasterizing these points as disc-like primitives. Unlike its sibling [Deferred Surface Splatting](1072-deferred-surface-splatting.md), which focuses on deferred shading pipelines for surface normals and depth, standard PBR often relies on opaque primitives to avoid sorting costs, though this introduces specific aliasing trade-offs. It builds on the concept of particle systems introduced by Reeves and was formalized for large point-set data by Rusinkiewicz and Levoy. See [Point-Based Rendering](2827-point-based-rendering.md).

### [Hardware EWA Splatting Pipeline](13219-hardware-ewa-splatting-pipeline.md)
The Hardware EWA Splatting Pipeline is a specialized GPU architecture designed to render Elliptical Weighted Average (EWA) splats in a single pass, addressing the inefficiencies of multi-pass software implementations on standard graphics hardware. It solves the problem of accurate attribute accumulation and normalization for point-based surfaces by introducing a frame buffer-sized surface reconstruction buffer and a ternary depth test, allowing splats to be processed with maximum reuse of existing GPU units. Unlike standard triangle rasterization, which processes discrete primitives, this pipeline defers shading until after splat contributions are accumulated, enabling any fragment shading program to be used on the reconstructed surface. See [Hardware EWA Splatting Pipeline](13219-hardware-ewa-splatting-pipeline.md).

### Weighted Splatting with Depth Proximity
A rendering technique where point clouds are rendered as spheres (splats) in a multi-pass pipeline. It accumulates weighted attributes (color, normal) into a G-buffer to handle heavy overlap and reconstruct surface properties, deferring lighting to a final resolve pass [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** The method requires three passes: Depth, Attribute Blend, and Finalization. The complexity is $O(N)$ for the splatting passes (where $N$ is the number of splats) and $O(P)$ for the finalization pass (where $P$ is the number of pixels).
*   **Memory Complexity:** Requires storage for the G-Buffer (Depth, Normal, Color/Weight textures). The size is proportional to the screen resolution.
*   **Measured Performance:** Performance depends on the number of splats and the screen resolution. The use of a deferred pipeline allows efficient handling of multiple lights without re-rasterizing geometry [S2, unofficial].
*   **Scaling:** Scales well with the number of lights due to the deferred shading approach. The use of an octree for Level of Detail (LoD) and CPU Frustum Culling can reduce the number of splats rendered for large point clouds [S2, unofficial].
*   **Strengths:** Handles heavy overlap and transparency artifacts effectively. Decouples geometry projection from lighting, allowing for flexible lighting models. Reconstructs smooth, hole-free surfaces from sparse point clouds [S2, unofficial].
*   **Limitations:** Requires multiple passes, which can be less efficient than single-pass forward rendering for simple scenes. The quality of the reconstructed surface depends on the splat size and the $\varepsilon$ parameter. Aliasing can occur if the splat size is too small relative to the pixel size [S2, unofficial].
*   **When to use:** Suitable for rendering large, unstructured point clouds where surface continuity and high-quality lighting are required.
*   **When not to use:** Not suitable for scenes with very few points or where single-pass performance is critical and lighting complexity is low.

## Background
Deferred Surface Splatting is based on the Surface Splatting technique introduced by Zwicker, Pfister, van Baar, and Gross in *Surface Splatting*, SIGGRAPH 2001 [S2, unofficial]. The multi-pass GPU pipeline was further developed by Botsch, Hornung, Zwicker, and Kobbelt in *High-Quality Surface Splatting on Today's GPUs*, PBG 2005 [S2, unofficial]. The specific implementation described in the sources is an unofficial OpenGL renderer that turns raw, unstructured point clouds into continuous, shaded surfaces using splatting combined with a deferred shading pipeline [S2, unofficial].

## Sources
[S1] sebastianlipponer/surface_splatting, https://github.com/sebastianlipponer/surface_splatting, fetched 2026-10-28 (unofficial)
[S2] arminkz/PointCloudRendering, https://github.com/arminkz/PointCloudRendering, fetched 2026-10-28 (unofficial)
