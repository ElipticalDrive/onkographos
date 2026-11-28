# Shadow Mapping

Also called: Shadow Map, Depth Buffer Shadowing

## Overview
Shadow Mapping is a technique to efficiently render shadows by rendering the scene from the light's point of view to generate a depth map (shadow map), which determines if a point is occluded by another object from the light's perspective [S1, S2, unofficial]. It sets itself apart from sibling methods like [Two-Pass Shadow Casting](2196-two-pass-shadow-casting-kajiya-and-von-herzen.md) and [Ray-marched Volumetric Lighting with Density Thresholding](2456-ray-marched-volumetric-lighting-with-density-thresholding.md) by decoupling shadow generation from the view-space traversal, using a 2D texture lookup for visibility testing rather than integrating transmittance or casting explicit rays per pixel [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Scene geometry, light source parameters (position, direction, projection type), camera parameters.
*   **Outputs:** A shadow map (depth texture) and a final image with shadowed lighting.

**Data Structures:**
*   **Shadow Map:** A 2D texture (typically `GL_DEPTH_COMPONENT` or `GL_R32F`) storing the depth values of the closest surface to the light source for each texel.
*   **Frame Buffer Object (FBO):** A render target used to capture the depth buffer from the light's perspective [S2, unofficial].

**Steps:**

1.  **Generate Light-Space Projection Matrix:**
    For a spotlight, construct the view and projection matrices.
    *   Calculate the up vector to avoid gimbal lock when the light direction is parallel to the world up vector:
        $$ \text{up} = \text{normalize}(\text{cross}(\text{dir}, \text{world\_up})) $$
        Where `world_up` is chosen as $(0,0,1)$ if $|\text{dot}(\text{dir}, (0,0,1))| < 0.99$, otherwise $(1,0,0)$ [S1, unofficial].
    *   Create the view matrix using `lookAt(light\_pos, light\_pos + light\_dir, up)` [S1, unofficial].
    *   Create the perspective projection matrix using the light's aperture angle, aspect ratio (usually 1 for square shadow maps), and near/far planes (e.g., 0.01 to 3.0) [S1, unofficial].
    *   Combine into a single model-view-projection matrix: $M_{light} = P_{light} \times V_{light}$ [S1, unofficial].

2.  **Render Shadow Map:**
    *   Bind the FBO containing the depth texture [S2, unofficial].
    *   Render the scene geometry using $M_{light}$, writing only depth values to the framebuffer [S1, S2, unofficial].

3.  **Sample Shadow Map during Shading:**
    For each fragment being shaded:
    *   Transform the fragment's world position $\mathbf{p}$ into light space:
        $$ \mathbf{p}_{light} = M_{light} \times \mathbf{p} $$
    *   Perform perspective division to get normalized device coordinates (NDC):
        $$ \mathbf{p}_{ndc} = \frac{\mathbf{p}_{light}.xyz}{\mathbf{p}_{light}.w} $$
    *   Convert NDC $[-1, 1]$ to texture coordinates $[0, 1]$:
        $$ \mathbf{c} = \mathbf{p}_{ndc} \times 0.5 + 0.5 $$
    *   Sample the stored depth from the shadow map:
        $$ d_{shadow} = \text{texture}(u_{shadowMap}, \mathbf{c}.xy).r $$
    *   Compare the fragment's light-space depth ($\mathbf{c}.z$) with the stored depth. A bias is added to prevent self-shadowing artifacts (shadow acne) [S1, S2, unofficial]:
        $$ \text{shadowVal} = \begin{cases} 1.0 & \text{if } \mathbf{c}.z \ge d_{shadow} + \text{bias} \\ 0.0 & \text{otherwise} \end{cases} $$
    *   The final illumination is modulated by `shadowVal` [S1, S2, unofficial].

**Parameters:**
*   **Bias:** A small positive value added to the shadow depth comparison to account for quantization errors and angle-dependent shadow acne [S1, unofficial]. Typical values depend on the depth precision and distance from the light.
*   **Resolution:** The dimensions of the shadow map texture (e.g., 4096x4096) [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Shadow Acne:** Caused by quantization errors in the depth buffer where a surface incorrectly shadows itself. Mitigated by bias [S1, unofficial].
*   **Peter-panning:** Shadows appearing to float above surfaces. Mitigated by adjusting bias based on distance or surface orientation [S1, unofficial].
*   **Perspective Distortion:** Standard shadow maps suffer from uneven sampling density. Variants like [PCF Shadow Lookup with Slope Scaling and Edge Fade](10994-pcf-shadow-lookup-with-slope-scaling-and-edge-fade.md) address this with slope scaling and edge fading.

## Subtopics

### [Shadow Receiving for Volumes](956-shadow-receiving-for-volumes.md)
A technique that allows volumetric data to receive shadows cast by polygonal actors without requiring the volume to be rendered twice. It solves the performance bottleneck of traditional shadow pipelines by integrating shadow occlusion checks directly into the view-space ray traversal against a pre-computed shadow map. Unlike [Lookup Table Shadow Rendering](3588-lookup-table-shadow-rendering.md) and [Ray Marching with Shadow Mapping](1415-ray-marching-with-shadow-mapping.md), it relies on standard shadow map textures rather than 3D transmittance textures or light-strength buffers.

### [Variance Shadow Mapping for NeRFs](3561-variance-shadow-mapping-for-nerfs.md)
A technique to render cast shadows in neural radiance fields by leveraging depth statistics from a light-source viewpoint, inspired by traditional variance shadow mapping. It avoids the computational cost of cumulative transmittance along shadow rays by comparing depth inconsistencies between the camera and light views, producing soft shadows with continuous visibility values in $[0, 1]$ rather than the hard shadows produced by naive depth comparison.

### [Variance Shadow Mapping](5686-variance-shadow-mapping.md)
A shadow mapping technique that smoothens depth values to calculate the probability of occlusion using Chebyshev's inequality, producing soft shadow edges instead of the abrupt 0/1 values of naive shadow mapping. It sets itself apart from standard shadow mapping by enabling efficient soft shadows with minimal computational overhead, requiring only the storage and filtering of two depth maps.

### [PCF Shadow Lookup with Slope Scaling and Edge Fade](10994-pcf-shadow-lookup-with-slope-scaling-and-edge-fade.md)
Computes sun shadow occlusion at a world position using a comparison sampler and a dynamic PCF kernel, with slope-scaled depth bias and distance-based edge fading to prevent peter-panning and hard shadow edges. It differs from standard shadow mapping by offsetting the sample position along the surface normal rather than the light direction to maintain contact stability, and by fading occlusion near the shadow map boundaries to avoid black artifacts at volume edges.

### Reflective Shadow Maps
A variant that stores surface properties (albedo, normal, direct illumination) alongside depth from the light's perspective [S3, unofficial]. During final shading, texels of the reflective shadow map are treated as point lights to provide single-bounce indirect illumination [S3, unofficial]. Pitfalls include excessive performance cost if all texels are used, requiring heuristics to select regions, and temporal stability issues (flickering) if the number of virtual lights is insufficient [S3, unofficial].

### Shadow Caching
A technique that exploits the inherent coherence of shadow rays for a point light source that are occluded by the same object [S4, unofficial]. It subdivides the scene and keeps a list of light sources that will be fully occluded, fully visible, or partially visible to each region of space, using approximations based on shadow rays for dynamic scenes [S4, unofficial].

### Moment-Based Order-Independent Transparency
A method for compact storage and fast reconstruction of the depth-dependent transmittance function per pixel, allowing compositing of transparent surfaces rendered in arbitrary order [S5, unofficial]. It works with the logarithm of the transmittance such that the function may be accumulated additively rather than multiplicatively [S5, unofficial]. Steps include an additive rendering pass to yield moments and a second additive pass for compositing using moment-based reconstruction [S5, unofficial]. Variants utilize four, six, eight power moments, or trigonometric moments [S5, unofficial]. It is completely order-independent and works well for participating media [S5, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ for shadow map generation (where $N$ is the number of polygons) and $O(1)$ per pixel for shadow lookup during shading.
*   **Memory Complexity:** $O(W \times H)$ for the shadow map texture, where $W$ and $H$ are the resolution of the shadow map [S2, unofficial].
*   **Performance:** On an RTX 3050, rendering 5,000,000 blocks at 4096x4096 resolution with shadow mapping achieved over 200 fps [S2, unofficial].
*   **Strengths:** Efficient for hard shadows; widely supported; decouples shadow generation from view-space traversal.
*   **Limitations:** Hard shadows only (without VSM); resolution-dependent artifacts (pixelation); perspective distortion; requires careful bias tuning.
*   **When to use:** When hard shadows are acceptable and performance is critical.
*   **When not to use:** When soft shadows are required (use VSM or [Volumetric Soft Shadowing](4507-volumetric-soft-shadowing.md)) or when high-quality self-shadowing in complex volumes is needed without resolution artifacts.

## Background
Shadow Mapping was introduced as a standard technique for efficient shadow rendering. It is widely used in real-time graphics, including voxel engines and games, to add visual depth and realism [S2, unofficial]. The technique has evolved to include variants like Variance Shadow Mapping for soft shadows and Reflective Shadow Maps for indirect illumination [S1, S3, S5, unofficial].

## Sources
[S1] Voxel Cone Tracing - GitHub, https://raw.githubusercontent.com/righier/gidemo/master/report.pdf, fetched 2026-10-27 (unofficial)
[S2] Creating a Voxel Engine with OpenGL | Walker McGilvary, https://theslabby.github.io/2023/12/24/start-voxelengine.html, fetched 2026-10-28 (unofficial)
[S3] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
[S4] Massive Data Management for Distributed Volume Visualization, http://hdl.handle.net/1951/47671, fetched 2026-10-26 (unofficial)
[S5] (PDF) Image-space visibility ordering for cell projection volume ..., https://www.researchgate.net/publication/8193687_Image-space_visibility_ordering_for_cell_projection_volume_rendering_of_unstructured_data, fetched 2026-10-29 (unofficial)
