# Shadow Mapping

Also called: Depth Pass FBO Setup, Shadow FBO Initialization, Shadow Map Setup

## Overview
Shadow Mapping is a technique to efficiently render shadows by rendering the scene from the light's point of view to generate a depth map (shadow map) [S1, S2, S3, unofficial]. This depth map determines if a point is occluded by another object from the light's perspective [S1, unofficial]. It sets itself apart from sibling methods like [Ray-marched Volumetric Lighting with Density Thresholding](2456-ray-marched-volumetric-lighting-with-density-thresholding.md) and [Ray-marching with Shadow Mapping](1415-ray-marching-with-shadow-mapping.md) by relying on standard 2D shadow map textures rather than 3D transmittance textures or light-strength buffers [S1, unofficial]. Unlike [Two-Pass Shadow Casting (Kajiya and Von Herzen)](2196-two-pass-shadow-casting-kajiya-and-von-herzen.md), which may involve complex volume traversal, Shadow Mapping decouples the shadow generation from the final shading pass, allowing for binary occlusion decisions against opaque occluders [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Scene geometry, light source parameters (position, direction, angle for spotlights), and camera parameters.
*   **Output:** A shadow map (2D texture storing depth values) and a final shaded image where shadow values are applied.

**Data Structures:**
*   **Shadow Map:** A 2D texture storing depth values. Typically uses 32-bit floating-point format (`GL_DEPTH_COMPONENT32`) to store depth from the light's perspective [S3, unofficial].
*   **Frame Buffer Object (FBO):** A framebuffer object bound to the shadow map texture to capture the depth buffer during the light-space render pass [S2, S3, unofficial].

**Steps:**

1.  **Generate Light-Space Projection Matrix:**
    For a spotlight, calculate the view and projection matrices.
    *   Calculate the `up` vector:
        $$ \text{up} = \text{normalize}(\text{cross}(\text{dir}, \text{up\_ref})) $$
        where $\text{up\_ref}$ is $\langle 0,0,1 \rangle$ if $|\text{dot}(\text{dir}, \langle 0,0,1 \rangle)| < 0.99$, otherwise $\langle 1,0,0 \rangle$ [S1, unofficial].
    *   Create the view matrix using $\text{glm::lookAt}(\text{pos}, \text{pos}+\text{dir}, \text{up})$ [S1, unofficial].
    *   Create the projection matrix using $\text{glm::perspective}(\text{angle}, 1.0, 0.01, 3.0)$ [S1, unofficial].
    *   Combine into a projection matrix: $M_{proj} = P \times V$ [S1, unofficial].

2.  **Setup Shadow FBO:**
    *   Generate and bind an FBO: `glGenFramebuffers(1, &depthMapFBO); glBindFramebuffer(GL_FRAMEBUFFER, depthMapFBO);` [S3, unofficial].
    *   Generate a 2D texture for depth storage: `glGenTextures(1, &depthMap);` [S3, unofficial].
    *   Allocate texture memory with 32-bit float depth format: `glTexImage2D(GL_TEXTURE_2D, 0, GL_DEPTH_COMPONENT32, shadowWidth, shadowHeight, 0, GL_DEPTH_COMPONENT, GL_FLOAT, 0);` [S3, unofficial].
    *   Set texture filtering and wrapping:
        *   Min/Mag filter: `GL_NEAREST` [S3, unofficial].
        *   Wrap S/T: `GL_CLAMP_TO_EDGE` [S3, unofficial].
    *   Attach the texture to the FBO's depth attachment point: `glFramebufferTexture2D(GL_FRAMEBUFFER, GL_DEPTH_ATTACHMENT, GL_TEXTURE_2D, depthMap, 0);` [S3, unofficial].
    *   Disable color buffers: `glDrawBuffer(GL_NONE); glReadBuffer(GL_NONE);` [S3, unofficial].
    *   Validate framebuffer completeness: `glCheckFramebufferStatus(GL_FRAMEBUFFER)` [S3, unofficial].
    *   Unbind FBO: `glBindFramebuffer(GL_FRAMEBUFFER, 0);` [S3, unofficial].

3.  **Render Shadow Map (Light Pass):**
    *   Bind the shadow FBO.
    *   Set the viewport to the shadow map resolution.
    *   Clear the depth buffer.
    *   Render the scene geometry using the light-space projection matrix ($M_{proj}$) calculated in Step 1. This populates the shadow map with depth values from the light's perspective.

4.  **Sample Shadow Map (Shading Pass):**
    *   For each fragment being shaded, transform its world position $\mathbf{p}$ into light space:
        $$ \mathbf{p}_{light} = M_{proj} \times \mathbf{p} $$
    *   Perform perspective division to get normalized device coordinates (NDC):
        $$ \mathbf{p}_{ndc} = \frac{\mathbf{p}_{light}.xyz}{\mathbf{p}_{light}.w} $$
    *   Transform NDC to texture coordinates (assuming depth range [0, 1]):
        $$ \text{coords} = \mathbf{p}_{ndc} \times 0.5 + 0.5 $$
        [S1, unofficial]
    *   Sample the shadow map depth:
        $$ \text{shadowDepth} = \text{texture}(u_{shadowMap}, \text{coords}.xy).r $$
        [S1, unofficial]
    *   Compare the fragment's light-space depth ($\text{coords}.z$) with the sampled depth, applying a bias to avoid shadow acne:
        $$ \text{shadowVal} = \begin{cases} 1.0 & \text{if } \text{coords}.z \ge \text{shadowDepth} + \text{bias} \\ 0.0 & \text{otherwise} \end{cases} $$
        [S1, unofficial]
    *   The `bias` parameter is a small value added to the shadow depth comparison to avoid quantization errors [S1, unofficial].

**Parameters:**
*   **Bias:** A small value added to the shadow depth comparison to avoid shadow acne caused by quantization errors [S1, unofficial]. Typical values depend on the scene scale and depth precision, often in the range of $10^{-4}$ to $10^{-2}$ [S1, unofficial].
*   **Shadow Map Resolution:** Width and height of the shadow map texture. Higher resolutions reduce aliasing but increase memory and sampling cost [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Shadow Acne:** Caused by quantization errors in depth comparison. Mitigated by bias [S1, unofficial].
*   **Peter Panning:** Caused by excessive bias, making shadows appear detached from objects.
*   **Perspective Distortion:** Depth values are non-linear in light space. Bias may need to be scaled with distance or depth value [S1, unofficial].
*   **View Frustum Culling:** The light's view frustum must encompass all potential shadow casters and receivers. If objects are outside the frustum, they won't cast shadows correctly [S1, unofficial].

## Subtopics

### [Shadow Receiving for Volumes](956-shadow-receiving-for-volumes.md)
Shadow Receiving for Volumes allows volumetric data to receive shadows cast by polygonal actors without requiring the volume to be rendered twice. It solves the performance bottleneck of traditional shadow pipelines where rendering a volume as a shadow caster or receiver incurs significant overhead due to double traversal. Unlike sibling methods that compute self-shadowing or light-space transmittance buffers, this method integrates shadow occlusion checks directly into the view-space ray traversal against a pre-computed shadow map. It is distinct from [Lookup Table Shadow Rendering](3588-lookup-table-shadow-rendering.md) and [Ray Marching with Shadow Mapping](1415-ray-marching-with-shadow-mapping.md) by relying on standard shadow map textures rather than 3D transmittance textures or light-strength buffers.

### [Variance Shadow Mapping for NeRFs](3561-variance-shadow-mapping-for-nerfs.md)
Variance Shadow Mapping for NeRFs is a technique to render cast shadows in neural radiance fields by leveraging depth statistics from a light-source viewpoint, inspired by traditional variance shadow mapping. It avoids the computational cost of cumulative transmittance along shadow rays by comparing depth inconsistencies between the camera and light views, producing soft shadows with continuous visibility values in $[0, 1]$ rather than the hard shadows produced by naive depth comparison.

### [Variance Shadow Mapping](5686-variance-shadow-mapping.md)
Variance Shadow Mapping (VSM) is a shadow mapping technique that smoothens depth values to calculate the probability of occlusion using Chebyshev's inequality, producing soft shadow edges instead of the abrupt 0/1 values of naive shadow mapping. It sets itself apart from standard shadow mapping by enabling efficient soft shadows with minimal computational overhead, requiring only the storage and filtering of two depth maps.

### [Shadow Caching](9725-shadow-caching.md)
Shadow Caching is a technique that exploits the inherent coherence of shadow rays for a point light source that are occluded by the same object. It improves performance by storing shadow test results in a spatial data structure, avoiding redundant ray casting for identical occlusion queries. Unlike Shadow Mapping, which stores depth values for binary occlusion decisions against opaque occluders, Shadow Caching stores the result of the shadow ray traversal itself.

### [PCF Shadow Lookup with Slope Scaling and Edge Fade](10994-pcf-shadow-lookup-with-slope-scaling-and-edge-fade.md)
This method computes sun shadow occlusion at a world position by performing Percentage-Closer Filtering (PCF) on a shadow map, augmented with slope-scaled depth bias and distance-based edge fading. It solves the twin problems of shadow acne/peter-panning (via normal-offset bias) and harsh, aliased shadow boundaries at the edges of the shadow map (via smooth fading). It differs from standard PCF Shadow Mapping by dynamically adjusting the depth bias based on surface orientation relative to the light and by softening the shadow intensity at the periphery of the shadow map to prevent hard cutoff artifacts.

### Reflective Shadow Maps
Reflective Shadow Maps perform a direct lighting pass of the scene and store the result in textures [S4, unofficial]. They render from the point of view of the light source, storing depth information and surface properties (albedo, normal, direct illumination) [S4, unofficial]. During final shading, texels of the reflective shadow map are treated as point lights to provide single bounce indirect illumination [S4, unofficial]. Optimization involves reversing processing: creating several lights based on the entire map and splatting them in screen-space [S4, unofficial]. Pitfalls include excessive performance cost if all texels are used, requiring heuristics to choose regions, and temporal stability issues/flickering if the number of lights is insufficient [S4, unofficial].

### Moment-Based Order-Independent Transparency
Moment-Based Order-Independent Transparency is a method for compact storage and fast reconstruction of the depth-dependent transmittance function per pixel, allowing compositing of transparent surfaces rendered in arbitrary order [S5, unofficial]. It works with the logarithm of the transmittance such that the function may be accumulated additively rather than multiplicatively [S5, unofficial]. Steps include performing an additive rendering pass for all transparent surfaces to yield moments, and using moment-based reconstruction algorithms to provide approximations to the original function for compositing in a second additive pass [S5, unofficial]. Variants utilize existing algorithms with four or six power moments, or new algorithms using eight power moments or up to four trigonometric moments [S5, unofficial]. It is completely order-independent, unlike methods requiring sorting, and works well for participating media as well as transparent surfaces [S5, unofficial].

## Operating characteristics
*   **Time Complexity:** The shadow map generation is $O(N)$ where $N$ is the number of polygons in the scene [S1, S2, unofficial]. The shading pass is $O(M)$ where $M$ is the number of fragments, with constant-time shadow map lookup [S1, unofficial].
*   **Memory Complexity:** $O(W \times H)$ for the shadow map texture, where $W$ and $H$ are the shadow map dimensions [S3, unofficial].
*   **Measured Performance:** On an RTX 3050 laptop, rendering 5,000,000 blocks with 4096x4096 shader resolution and shadow mapping achieved over 200fps [S2, unofficial].
*   **Strengths:** Efficient for hard shadows; widely supported; relatively simple to implement [S1, S2, S3, unofficial].
*   **Limitations:** Produces hard shadows unless extended (e.g., VSM, PCF); suffers from aliasing and precision issues (shadow acne, peter panning); requires careful bias tuning [S1, S2, unofficial].
*   **When to Use:** When hard shadows are acceptable, or when extended with soft shadow techniques; for scenes with opaque occluders.
*   **When Not to Use:** When soft shadows are required without extensions; for transparent occluders without additional techniques (e.g., Moment-Based Order-Independent Transparency) [S5, unofficial].

## Background
Shadow Mapping was introduced as a technique to efficiently render shadows by rendering the scene from the light's point of view to generate a depth map [S1, S2, S3, unofficial]. It is a fundamental technique in real-time graphics for adding depth and realism to scenes [S2, unofficial].

## Sources
[S1] Voxel Cone Tracing - GitHub, https://raw.githubusercontent.com/righier/gidemo/master/report.pdf, fetched 2026-10-27 (unofficial)
[S2] Creating a Voxel Engine with OpenGL | Walker McGilvary, https://theslabby.github.io/2023/12/24/start-voxelengine.html, fetched 2026-10-28 (unofficial)
[S3] GitHub Repo for VoxelEngine, https://github.com/TheSlabby/VoxelEngine, fetched 2026-10-29 (unofficial)
[S4] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
[S5] (PDF) Image-space visibility ordering for cell projection volume ..., https://www.researchgate.net/publication/8193687_Image-space_visibility_ordering_for_cell_projection_volume_rendering_of_unstructured_data, fetched 2026-10-29 (unofficial)
