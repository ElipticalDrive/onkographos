# Shadow Mapping

Also called: Shadow Map, Depth Buffer Shadowing

## Overview
Shadow Mapping is a technique to efficiently render shadows by rendering the scene from the light's point of view to generate a depth map (shadow map), which determines if a point is occluded by another object from the light's perspective [S1, S2, unofficial]. It solves the problem of determining visibility from a light source without performing expensive ray casting for every fragment. It sets itself apart from [Shadow Ray Casting](2796-shadow-ray-casting.md) by using a pre-rendered depth buffer instead of per-pixel ray traversal, and from [Two-Pass Shadow Casting (Kajiya and Von Herzen)](2196-two-pass-shadow-casting-kajiya-and-von-herzen.md) by relying on standard 2D depth textures rather than volume transmittance buffers or specific volume rendering passes [S1, unofficial]. Trade-offs include the potential for aliasing artifacts and the need for bias adjustments, which are detailed in Operating characteristics.

## How it works
**Inputs and Outputs:**
- **Inputs:** Scene geometry, light source parameters (position, direction, aperture angle for spotlights), camera parameters.
- **Outputs:** A shadow map (depth texture) and a final image with shadow occlusion values.

**Data Structures:**
1.  **Shadow Map:** A 2D texture (typically `GL_DEPTH_COMPONENT` or `GL_R32F`) storing depth values from the light's perspective. Resolution is typically $N \times N$ (e.g., 1024x1024 to 4096x4096) [S2, unofficial].
2.  **Light Space Projection Matrix:** A $4 \times 4$ matrix combining view and projection transformations for the light source.

**Numbered Steps:**

1.  **Generate Light Space Projection Matrix:**
    For a spotlight, calculate the up vector and create the view and projection matrices.
    -   Let $\mathbf{dir}$ be the normalized direction of the light.
    -   Let $\mathbf{pos}$ be the position of the light.
    -   Let $\theta$ be the aperture angle in radians.
    -   Calculate the up vector $\mathbf{up}$:
        $$ \mathbf{up} = \begin{cases} \text{normalize}(\text{cross}(\mathbf{dir}, \mathbf{e}_z)) & \text{if } |\mathbf{dir} \cdot \mathbf{e}_z| < 0.99 \\ \mathbf{e}_x & \text{otherwise} \end{cases} $$
        where $\mathbf{e}_z = (0,0,1)$ and $\mathbf{e}_x = (1,0,0)$ [S1, unofficial].
    -   Create the view matrix $\mathbf{V}_{light}$ using $\text{lookAt}(\mathbf{pos}, \mathbf{pos} + \mathbf{dir}, \mathbf{up})$ [S1, unofficial].
    -   Create the projection matrix $\mathbf{P}_{light}$ using $\text{perspective}(\theta, 1.0, z_{near}, z_{far})$ [S1, unofficial].
    -   The final projection matrix is $\mathbf{M}_{light} = \mathbf{P}_{light} \mathbf{V}_{light}$ [S1, unofficial].

2.  **Render Shadow Map:**
    -   Bind a Framebuffer Object (FBO) with a depth attachment (the shadow map texture) [S2, unofficial].
    -   Set the viewport to the resolution of the shadow map.
    -   Set the active shader to output depth values.
    -   Render the scene geometry using $\mathbf{M}_{light}$ as the model-view-projection matrix. This stores the minimum depth from the light's origin to any surface in the shadow map [S1, S2, unofficial].

3.  **Sample Shadow Map during Main Pass:**
    -   For each fragment being shaded, transform its world position $\mathbf{p}_{world}$ into light space:
        $$ \mathbf{p}_{light} = \mathbf{M}_{light} \mathbf{p}_{world} $$
    -   Perform perspective division to get normalized device coordinates (NDC) in light space:
        $$ \mathbf{p}_{ndc} = \left( \frac{\mathbf{p}_{light}.x}{\mathbf{p}_{light}.w}, \frac{\mathbf{p}_{light}.y}{\mathbf{p}_{light}.w}, \frac{\mathbf{p}_{light}.z}{\mathbf{p}_{light}.w} \right) $$
    -   Convert NDC to texture coordinates (assuming OpenGL-style depth range $[0, 1]$):
        $$ \mathbf{c}_{tex} = \mathbf{p}_{ndc} \cdot 0.5 + 0.5 $$
        where $\mathbf{c}_{tex}.xy$ are the texture coordinates and $\mathbf{c}_{tex}.z$ is the fragment's depth in light space [S1, unofficial].
    -   Sample the stored depth from the shadow map:
        $$ d_{shadow} = \text{texture}(\text{u\_shadowMap}, \mathbf{c}_{tex}.xy).r $$
    -   Compare the fragment's depth with the stored depth to determine occlusion. A bias $b$ is added to prevent self-shadowing artifacts (shadow acne) [S1, unofficial]:
        $$ \text{shadowVal} = \begin{cases} 1.0 & \text{if } \mathbf{c}_{tex}.z \ge d_{shadow} + b \\ 0.0 & \text{otherwise} \end{cases} $$
    -   The final illumination is modulated by $\text{shadowVal}$ [S1, unofficial].

**Parameters:**
-   **Bias ($b$):** A small positive value added to the shadow depth comparison to avoid shadow acne caused by quantization errors and precision issues. Typical values depend on the scene scale and depth buffer precision [S1, unofficial].
-   **Aperture Angle ($\theta$):** The field of view of the spotlight. Determines the frustum shape [S1, unofficial].
-   **Near/Far Planes ($z_{near}, z_{far}$):** Depth clipping planes for the light's camera. Must tightly bound the scene to maximize depth buffer precision [S1, unofficial].

**Edge Cases and Pitfalls:**
-   **Shadow Acne:** Caused by precision errors where a surface incorrectly occludes itself. Mitigated by bias [S1, unofficial].
-   **Peter Panning:** Caused by excessive bias, making shadows appear to float above surfaces.
-   **Perspective Distortion:** Standard shadow maps suffer from aliasing when the light is far from the object or at grazing angles.
-   **Light Space Coordinates:** Incorrect handling of the $w$-division or texture coordinate conversion leads to incorrect shadow mapping [S1, unofficial].

## Subtopics

### [Shadow Receiving for Volumes](956-shadow-receiving-for-volumes.md)
Shadow Receiving for Volumes allows volumetric data to receive shadows cast by polygonal actors without requiring the volume to be rendered twice. It integrates shadow occlusion checks directly into the view-space ray traversal against a pre-computed shadow map, solving the performance bottleneck of double traversal. It is distinct from [Lookup Table Shadow Rendering](3588-lookup-table-shadow-rendering.md) and [Ray Marching with Shadow Mapping](1415-ray-marching-with-shadow-mapping.md) by relying on standard shadow map textures rather than 3D transmittance textures or light-strength buffers.

### [Variance Shadow Mapping for NeRFs](3561-variance-shadow-mapping-for-nerfs.md)
Variance Shadow Mapping for NeRFs renders cast shadows in neural radiance fields by leveraging depth statistics from a light-source viewpoint. It avoids the computational cost of cumulative transmittance along shadow rays by comparing depth inconsistencies between the camera and light views, producing soft shadows with continuous visibility values in $[0, 1]$ rather than the hard shadows produced by naive depth comparison.

### [Variance Shadow Mapping](5686-variance-shadow-mapping.md)
Variance Shadow Mapping (VSM) smoothens depth values to calculate the probability of occlusion using Chebyshev's inequality, producing soft shadow edges instead of the abrupt 0/1 values of naive shadow mapping. It sets itself apart from standard shadow mapping by enabling efficient soft shadows with minimal computational overhead, requiring only the storage and filtering of two depth maps.

### [Shadow Caching](9725-shadow-caching.md)
Shadow Caching exploits the inherent coherence of shadow rays for a point light source that are occluded by the same object. It improves performance by storing shadow test results in a spatial data structure, avoiding redundant ray casting for identical occlusion queries. Unlike Shadow Mapping, which stores depth values for binary occlusion decisions against opaque occluders, Shadow Caching stores the result of the shadow ray traversal itself.

### [PCF Shadow Lookup with Slope Scaling and Edge Fade](10994-pcf-shadow-lookup-with-slope-scaling-and-edge-fade.md)
Computes sun shadow occlusion at a world position using a comparison sampler and a dynamic PCF kernel, with slope-scaled depth bias and distance-based edge fading to prevent peter-panning and hard shadow edges. It differs from standard shadow mapping by offsetting the sample position along the surface normal rather than the light direction to maintain contact stability, and by fading occlusion near the shadow map boundaries to avoid black artifacts at volume edges.

### Reflective Shadow Maps
Reflective Shadow Maps perform a direct lighting pass of the scene and store the result in textures, then render from the point of view of the light source, storing depth information and surface properties (albedo, normal, direct illumination) [S3, unofficial]. During final shading, texels of the reflective shadow map are treated as point lights to provide single bounce indirect illumination [S3, unofficial]. Optimization involves reversing processing: creating several lights based on the entire map and splatting them in screen-space [S3, unofficial]. Pitfalls include excessive performance cost if all texels are used, requiring heuristics to choose regions, and temporal stability issues/flickering if the number of lights is insufficient [S3, unofficial].

### Moment-Based Order-Independent Transparency
A method for compact storage and fast reconstruction of the depth-dependent transmittance function per pixel, allowing compositing of transparent surfaces rendered in arbitrary order [S4, unofficial]. It works with the logarithm of the transmittance such that the function may be accumulated additively rather than multiplicatively [S4, unofficial]. Steps include performing an additive rendering pass for all transparent surfaces to yield moments, and using moment-based reconstruction algorithms to provide approximations to the original function for compositing in a second additive pass [S4, unofficial]. Variants utilize existing algorithms with four or six power moments, or new algorithms using eight power moments or up to four trigonometric moments [S4, unofficial]. It is completely order-independent, unlike methods requiring sorting, and works well for participating media as well as transparent surfaces [S4, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
-   **Time:** $O(N_{light\_pass} + N_{main\_pass})$, where $N$ is the number of pixels/fragments. The shadow map generation is a single render pass. The main pass involves a texture lookup per fragment, which is $O(1)$ per pixel.
-   **Memory:** $O(W \times H)$ for the shadow map, where $W, H$ are the shadow map dimensions. Typically 4-16 MB for 1024x1024 to 4096x4096 depth textures [S2, unofficial].

**Measured Performance:**
-   On an RTX 3050 laptop, rendering 5,000,000 blocks with 4096x4096 shadow map resolution achieved over 200 fps [S2, unofficial].

**Scaling:**
-   Performance scales linearly with shadow map resolution and scene complexity (number of draw calls).
-   Aliasing artifacts become more pronounced as the distance between the light and the object increases or at grazing angles.

**Strengths and Limitations:**
-   **Strengths:** Efficient, widely supported, simple to implement, works well for opaque geometry [S1, S2, unofficial].
-   **Limitations:** Hard shadows (unless using VSM/PCF), aliasing, bias tuning required, perspective distortion, not naturally suited for volumetric self-shadowing without extensions [S1, unofficial].

**When to use/not use:**
-   Use for efficient shadow casting from point/spot/directional lights onto opaque surfaces.
-   Not ideal for soft shadows without additional techniques (VSM, PCF) or for highly dynamic scenes with many lights due to multiple shadow map passes.

## Background
Shadow Mapping was introduced as a standard technique for real-time shadow rendering. It is widely used in computer graphics for its efficiency. The method described here, particularly the spotlight projection matrix generation, is documented in various unofficial reports and tutorials [S1, S2, unofficial]. Reflective Shadow Maps were inspired by work on virtual point lights and used in productions like Shrek 2 [S3, unofficial]. Moment-based transparency techniques build upon moment shadow mapping principles [S4, unofficial].

## Sources
[S1] Voxel Cone Tracing - GitHub, https://raw.githubusercontent.com/righier/gidemo/master/report.pdf, fetched 2026-10-27 (unofficial)
[S2] Creating a Voxel Engine with OpenGL | Walker McGilvary, https://theslabby.github.io/2023/12/24/start-voxelengine.html, fetched 2026-10-28 (unofficial)
[S3] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
[S4] (PDF) Image-space visibility ordering for cell projection volume ..., https://www.researchgate.net/publication/8193687_Image-space_visibility_ordering_for_cell_projection_volume_rendering_of_unstructured_data, fetched 2026-10-29 (unofficial)
