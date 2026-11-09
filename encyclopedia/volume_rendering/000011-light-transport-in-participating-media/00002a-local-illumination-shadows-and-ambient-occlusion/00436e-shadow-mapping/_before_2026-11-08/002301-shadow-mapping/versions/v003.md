# Shadow Mapping

Also called: Shadow Map, Depth Buffer Shadowing

## Overview
Shadow Mapping is a technique to efficiently render shadows by rendering the scene from the light's point of view to generate a depth map (shadow map) [S1, unofficial]. It determines if a point is occluded by another object from the light's perspective by comparing depth information from the light's perspective (stored in a framebuffer) with depth from the camera's perspective [S2, unofficial]. It sets itself apart from siblings like [Shadow Ray Casting](2796-shadow-ray-casting.md) by using a pre-rendered buffer for visibility tests rather than per-pixel ray traversal, and from [Two-Pass Shadow Casting (Kajiya and Von Herzen)](2196-two-pass-shadow-casting-kajiya-and-von-herzen.md) by relying on standard rasterization depth buffers rather than volume-specific accumulation.

## How it works
**Inputs and Outputs:**
*   **Inputs:** Scene geometry, light source parameters (position, direction, aperture angle for spotlights), camera parameters, shadow map resolution.
*   **Outputs:** A shadow map (depth texture) and a final image with shadowed lighting.

**Data Structures:**
*   **Shadow Map:** A 2D texture (or framebuffer object) storing depth values. For spotlights, a standard 2D texture is sufficient [S1, unofficial]. For point lights, a cube map would be required, but this implementation focuses on spotlights [S1, unofficial].
*   **Light Space Matrix:** A combined projection and view matrix ($M_{light}$) transforming world coordinates to light space.

**Steps:**

1.  **Generate Light Projection Matrix:**
    For a spotlight, calculate the view and projection matrices.
    *   Calculate the `up` vector to avoid gimbal lock when the light direction is parallel to the world up vector:
        $$ \text{up} = \begin{cases} \text{normalize}(\text{cross}(\text{vec3}(0,0,1), \mathbf{dir})) & \text{if } |\mathbf{dir} \cdot \text{vec3}(0,0,1)| < 0.99 \\ \text{normalize}(\text{cross}(\text{vec3}(1,0,0), \mathbf{dir})) & \text{otherwise} \end{cases} $$
    *   Create the view matrix using `glm::lookAt(lightPos, lightPos + dir, up)`.
    *   Create the projection matrix using `glm::perspective(angle, 1.0, near, far)`, where `angle` is the aperture, `near` is typically 0.01, and `far` is typically 3.0 for indoor scenes [S1, unofficial].
    *   Combine: $M_{light} = P_{light} \times V_{light}$.

2.  **Render Shadow Map:**
    *   Set up a separate Framebuffer Object (FBO) to capture depth information [S2, unofficial].
    *   Render the scene geometry using $M_{light}$, writing only depth values to the FBO. This generates the shadow map [S1, unofficial].

3.  **Sample Shadow Map during Rendering:**
    *   For each fragment in the main camera pass, transform its world position $\mathbf{p}$ into light space:
        $$ \mathbf{p}_{light} = M_{light} \times \mathbf{p} $$
    *   Perform perspective division to get normalized device coordinates (NDC) in light space:
        $$ \mathbf{p}_{ndc} = \frac{\mathbf{p}_{light}.xyz}{\mathbf{p}_{light}.w} $$
    *   Transform NDC to texture coordinates (0 to 1):
        $$ \mathbf{c} = \mathbf{p}_{ndc} \times 0.5 + 0.5 $$
    *   Sample the depth value from the shadow map at $\mathbf{c}.xy$:
        $$ d_{shadow} = \text{texture}(u_{shadowMap}, \mathbf{c}.xy).r $$
    *   Compare the fragment's light-space depth ($\mathbf{c}.z$) with the sampled shadow depth. Apply a bias to prevent shadow acne:
        $$ \text{shadowVal} = \begin{cases} 1.0 & \text{if } \mathbf{c}.z \ge d_{shadow} + \text{bias} \\ 0.0 & \text{otherwise} \end{cases} $$
    *   The bias value is added to avoid artifacts caused by quantization errors [S1, unofficial].

**Parameters:**
*   **Bias:** A small positive value added to the shadow depth comparison to prevent self-shadowing artifacts (shadow acne) due to floating-point precision errors [S1, unofficial].
*   **Near/Far Plane:** Defines the depth range of the light's view frustum. Typical values for indoor scenes are near=0.01, far=3.0 [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Shadow Acne:** Caused by precision errors where the fragment's depth is slightly less than the stored depth due to quantization. Mitigated by bias [S1, unofficial].
*   **Perspective Distortion:** Directional lights may suffer from perspective distortion at close distances. Spotlights are preferred for indoor scenes where perspective effects are significant [S1, unofficial].
*   **Cube Map Requirement:** Point lights require rendering to a cube map, which is more computationally expensive. This method assumes spotlights to avoid this complexity [S1, unofficial].

## Subtopics

### [Variance Shadow Mapping](5686-variance-shadow-mapping.md)
Uses Chebyshev's inequality on depth moments to produce soft shadows, differing from standard shadow mapping's binary occlusion.

## Operating characteristics
*   **Time Complexity:** $O(N)$ for rendering the shadow map (where $N$ is the number of polygons) and $O(M)$ for sampling during the main pass (where $M$ is the number of fragments).
*   **Memory Complexity:** $O(R^2)$ for the shadow map resolution $R$.
*   **Performance:** On an RTX 3050, rendering 5,000,000 blocks at 4096x4096 resolution achieved over 200 FPS with shadow mapping enabled [S2, unofficial].
*   **Strengths:** Efficient for static scenes; leverages hardware rasterization; simple to implement [S1, unofficial].
*   **Limitations:** Prone to aliasing (jagged edges) and shadow acne; requires bias tuning; perspective distortion for directional lights; not suitable for dynamic scenes without frequent shadow map updates [S1, unofficial][S2, unofficial].
*   **When to Use:** Indoor scenes with spotlights; static geometry; when performance is critical and soft shadows are not required (or handled by other means).
*   **When Not to Use:** Point lights (requires cube maps); large outdoor scenes with directional lights (perspective issues); dynamic scenes with frequent geometry changes (high update cost).

## Background
Shadow mapping was introduced by Lance Williams in 1978 [S1, unofficial]. It is a widely used technique in real-time graphics for rendering shadows. The method described here is based on implementations for voxel engines and indoor scenes, specifically using spotlights to avoid the complexity of cube maps [S1, unofficial][S2, unofficial].

## Sources
[S1] Voxel Cone Tracing - GitHub, https://raw.githubusercontent.com/righier/gidemo/master/report.pdf, fetched 2026-10-27 (unofficial)
[S2] Creating a Voxel Engine with OpenGL | Walker McGilvary, https://theslabby.github.io/2023/12/24/start-voxelengine.html, fetched 2026-10-28 (unofficial)
