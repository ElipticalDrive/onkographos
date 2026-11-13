# Shadow Mapping

Also called: Depth Pass FBO Setup, Shadow FBO Initialization, Shadow Map Setup, Shadow Mapping for Cast Shadows

## Overview
Shadow Mapping is a technique to detect cast shadows by rendering a depth map from the light's viewpoint and comparing it with the 3D points obtained from the camera's depth map [S1, S2, S3, S4, S5, unofficial]. It solves the problem of determining visibility between a surface point and a light source without the computational cost of ray casting for every fragment. It sets itself apart from [Shadow Ray Casting](2796-shadow-ray-casting.md) by using a pre-rendered depth texture (shadow map) for batched visibility tests, and from [Two-Pass Shadow Casting (Kajiya and Von Herzen)](2196-two-pass-shadow-casting-kajiya-and-von-herzen.md) by operating in screen space rather than object space.

## How it works
**Inputs:** Scene geometry, light source parameters (position, direction, projection type), camera parameters.
**Outputs:** A shadow map (depth texture) and a final image with shadow occlusion values.

**Data Structures:**
1.  **Shadow Map Texture:**
    *   Type: `GL_TEXTURE_2D` [S2, S5, unofficial].
    *   Internal Format: `GL_DEPTH_COMPONENT` [S2, unofficial].
    *   Data Type: `GL_FLOAT` (32-bit) [S2, S5, unofficial].
    *   Dimensions: Square, typically $N \times N$. Common values: 4096x4096 [S2, S4, unofficial], or 1024, 2048, 8192 [S2, unofficial].
    *   Wrap Mode: `GL_CLAMP_TO_BORDER` [S2, unofficial] or `GL_CLAMP_TO_EDGE` [S5, unofficial].
    *   Border Color: `{1.0f, 1.0f, 1.0f, 1.0f}` [S2, unofficial].
    *   Filtering: `GL_NEAREST` for min/mag filters [S2, S5, unofficial].
2.  **Shadow Map FBO (Framebuffer Object):**
    *   Contains a single `GL_DEPTH_ATTACHMENT` bound to the Shadow Map Texture [S2, S5, unofficial].
    *   Draw Buffer: `GL_NONE` [S2, S5, unofficial].
    *   Read Buffer: `GL_NONE` [S2, S5, unofficial].
    *   Status: Must be `GL_FRAMEBUFFER_COMPLETE` [S2, S5, unofficial].

**Steps:**

1.  **Initialize Shadow Map FBO:**
    *   Generate a Framebuffer Object ID [S2, S5, unofficial].
    *   Generate a 2D Texture ID [S2, S5, unofficial].
    *   Bind the texture and allocate memory using `TexImage2D` with format `GL_DEPTH_COMPONENT` and type `GL_FLOAT` [S2, S5, unofficial].
    *   Set texture parameters: Wrap (`GL_CLAMP_TO_EDGE` or `GL_CLAMP_TO_BORDER`), Min/Mag Filter (`GL_NEAREST`) [S2, S5, unofficial].
    *   Bind the FBO [S2, S5, unofficial].
    *   Attach the texture to `GL_DEPTH_ATTACHMENT` [S2, S5, unofficial].
    *   Set draw and read buffers to `GL_NONE` [S2, S5, unofficial].
    *   Check framebuffer completeness. If not `GL_FRAMEBUFFER_COMPLETE`, throw an exception or handle error [S2, S5, unofficial].
    *   Unbind FBO [S5, unofficial].

2.  **Generate Light Projection Matrix:**
    *   For a spotlight, calculate the `up` vector:
        $$ \text{up} = \text{normalize}(\text{cross}(\mathbf{dir}, \mathbf{u})) $$
        where $\mathbf{u} = (0,0,1)$ if $|\mathbf{dir} \cdot (0,0,1)| < 0.99$, else $\mathbf{u} = (1,0,0)$ [S3, unofficial].
    *   Create View Matrix: $\mathbf{V}_{light} = \text{lookAt}(\mathbf{pos}_{light}, \mathbf{pos}_{light} + \mathbf{dir}, \text{up})$ [S3, unofficial].
    *   Create Projection Matrix: $\mathbf{P}_{light} = \text{perspective}(\text{angle}, \text{aspect}, \text{near}, \text{far})$ [S3, unofficial].
    *   Combine: $\mathbf{M}_{light} = \mathbf{P}_{light} \times \mathbf{V}_{light}$ [S3, unofficial].
    *   *Note:* For directional lights, an orthogonal projection is often used [S2, unofficial].

3.  **Render Shadow Map (Depth Pass):**
    *   Bind the Shadow Map FBO [S2, S5, unofficial].
    *   Set viewport to shadow map resolution ($N \times N$) [S2, unofficial].
    *   Clear the depth buffer [S2, unofficial].
    *   Render the scene geometry using a depth-only shader and the light's projection matrix $\mathbf{M}_{light}$ [S3, unofficial].
    *   This stores the closest depth value from the light's perspective in the shadow map texture [S1, S3, unofficial].
    *   Unbind FBO [S5, unofficial].

4.  **Sample Shadow Map (Main Pass):**
    *   For each fragment being shaded, transform its world position $\mathbf{p}$ into light space:
        $$ \mathbf{p}_{light} = \mathbf{M}_{light} \times \mathbf{p} $$
    *   Perform perspective division to get normalized device coordinates (NDC) in light space:
        $$ \mathbf{p}_{ndc} = \left( \frac{\mathbf{p}_{light}.x}{\mathbf{p}_{light}.w}, \frac{\mathbf{p}_{light}.y}{\mathbf{p}_{light}.w}, \frac{\mathbf{p}_{light}.z}{\mathbf{p}_{light}.w} \right) $$
    *   Convert to texture coordinates (assuming OpenGL depth range $[0, 1]$):
        $$ \mathbf{c} = \mathbf{p}_{ndc} \times 0.5 + 0.5 $$
        $$ \mathbf{c}_{xy} = (c_x, c_y) $$
    *   Sample the stored depth from the shadow map:
        $$ d_{shadow} = \text{texture}(\text{shadowMap}, \mathbf{c}_{xy}).r $$
    *   Compare the fragment's light-space depth ($c_z$) with the stored depth ($d_{shadow}$) plus a bias $b$:
        $$ \text{shadowVal} = \begin{cases} 1.0 & \text{if } c_z \ge d_{shadow} + b \\ 0.0 & \text{otherwise} \end{cases} $$
        [S3, unofficial].
    *   The bias $b$ is a small positive value added to avoid shadow acne caused by quantization errors [S3, unofficial].

**Parameters:**
*   **Bias:** A value added to the shadow depth comparison to avoid self-shadowing artifacts (shadow acne) [S3, unofficial]. Value not specified in sources [G].
*   **Shadow Map Resolution ($N$):** Square texture size. Typical values: 4096 [S2, S4, unofficial], 1024, 2048, 8192 [S2, unofficial].
*   **IsShadow:** Boolean flag enabling/disabling the shadow rendering pass [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Framebuffer Incomplete:** If the FBO status is not `GL_FRAMEBUFFER_COMPLETE` after attachment, the method fails [S2, S5, unofficial].
*   **Shadow Acne:** Caused by precision errors in depth comparison. Mitigated by bias [S3, unofficial].
*   **Perspective Distortion:** Directional lights may require orthogonal projections to avoid perspective distortion at large distances, while spotlights benefit from perspective projections for indoor scenes [S3, unofficial].
*   **Boundary Artifacts:** Texture wrapping modes (`CLAMP_TO_EDGE` or `CLAMP_TO_BORDER`) must be set to handle fragments outside the light's view frustum [S2, S5, unofficial].

## Subtopics

### [Shadow Receiving for Volumes](956-shadow-receiving-for-volumes.md)
Shadow Receiving for Volumes allows volumetric data to receive shadows cast by polygonal actors without rendering the volume twice. It integrates shadow occlusion checks directly into the view-space ray traversal against a pre-computed shadow map, solving the performance bottleneck of double traversal. Choose this when rendering volumes that interact with external geometry shadows.

### [Variance Shadow Mapping for NeRFs](3561-variance-shadow-mapping-for-nerfs.md)
This technique renders cast shadows in neural radiance fields by leveraging depth statistics from a light-source viewpoint. It avoids cumulative transmittance costs by comparing depth inconsistencies between camera and light views, producing soft shadows with continuous visibility values in $[0, 1]$. Choose this for NeRFs requiring soft shadows.

### [Variance Shadow Mapping](5686-variance-shadow-mapping.md)
VSM smoothens depth values to calculate the probability of occlusion using Chebyshev's inequality, producing soft shadow edges instead of abrupt 0/1 values. It requires storing and filtering two depth maps. Choose this when soft shadows are needed with minimal overhead.

### [Reflective Shadow Maps](9391-reflective-shadow-maps.md)
RSM approximates single-bounce indirect illumination by reusing shadow map data, storing surface properties (albedo, normal, direct illumination) alongside depth. It treats texels as virtual point lights during shading, providing diffuse interreflection without volumetric storage or ray-tracing costs. Choose this for global illumination effects.

### [Shadow Caching](9725-shadow-caching.md)
Shadow Caching exploits coherence of shadow rays for point lights occluded by the same object, storing results in a spatial data structure to avoid redundant ray casting. Unlike Shadow Mapping, it stores the traversal result itself. Choose this for scenes with high ray coherence.

### [PCF Shadow Lookup with Slope Scaling and Edge Fade](10994-pcf-shadow-lookup-with-slope-scaling-and-edge-fade.md)
This method computes sun shadow occlusion using Percentage-Closer Filtering (PCF) on a shadow map, augmented with slope-scaled depth bias and distance-based edge fading. It solves shadow acne/peter-panning and harsh shadow boundaries at map edges. Choose this for high-quality sun shadows.

### Moment-Based Order-Independent Transparency
A method for compact storage and fast reconstruction of depth-dependent transmittance per pixel, allowing compositing of transparent surfaces in arbitrary order [S6, unofficial]. It works with the logarithm of transmittance for additive accumulation, using moments (4, 6, 8 power, or trigonometric) for reconstruction [S6, unofficial]. It is completely order-independent and works for participating media [S6, unofficial]. Choose this for order-independent transparency in volumes.

## Operating characteristics
*   **Time Complexity:** $O(N^2)$ for shadow map generation (one render pass) + $O(F)$ for shadow lookup (one texture sample per fragment), where $N$ is shadow map resolution and $F$ is fragment count.
*   **Memory Complexity:** $O(N^2)$ for the shadow map texture [S2, S5, unofficial].
*   **Performance:** On an RTX 3050, rendering 5,000,000 blocks with a 4096x4096 shadow map achieved over 200 fps [S4, unofficial].
*   **Strengths:** Efficient for static scenes; hardware-accelerated depth comparison; simple to implement [S1, S3, S4, unofficial].
*   **Limitations:** Resolution-dependent artifacts (pixelation); perspective aliasing; requires careful bias tuning [S3, S5, unofficial].
*   **When to Use:** Real-time rendering with static or moderately dynamic geometry [S3, S4, unofficial].
*   **When Not to Use:** Highly dynamic scenes with frequent light movement may incur high update costs; soft shadows require variants like VSM [S1, unofficial].

## Background
Shadow Mapping was introduced by Lance Williams in 1978 [S1, S6, unofficial]. It is a standard technique in real-time graphics for rendering cast shadows [S1, S2, S3, S4, S5, unofficial]. It is widely used in voxel engines and general 3D rendering pipelines [S3, S4, S5, unofficial].

## Sources
[S1] Stéréophotométrie par Gaussian Splatting et rendu inverse, https://hal.science/hal-05151330v1, fetched 2026-10-26 (unofficial)
[S2] SuperAnt30/Vge, https://github.com/SuperAnt30/Vge, fetched 2026-10-27 (unofficial)
[S3] Voxel Cone Tracing - GitHub, https://raw.githubusercontent.com/righier/gidemo/master/report.pdf, fetched 2026-10-27 (unofficial)
[S4] Creating a Voxel Engine with OpenGL | Walker McGilvary, https://theslabby.github.io/2023/12/24/start-voxelengine.html, fetched 2026-10-28 (unofficial)
[S5] GitHub Repo for VoxelEngine, https://github.com/TheSlabby/VoxelEngine, fetched 2026-10-29 (unofficial)
[S6] (PDF) Image-space visibility ordering for cell projection volume ..., https://www.researchgate.net/publication/8193687_Image-space_visibility_ordering_for_cell_projection_volume_rendering_of_unstructured_data, fetched 2026-10-29 (unofficial)
