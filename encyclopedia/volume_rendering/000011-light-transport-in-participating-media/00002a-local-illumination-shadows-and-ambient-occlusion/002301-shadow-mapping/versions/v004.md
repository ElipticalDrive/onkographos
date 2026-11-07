# Shadow Mapping

Also called: Shadow Map, Depth Buffer Shadowing

## Overview
Shadow Mapping is a technique to efficiently render shadows by rendering the scene from the light's point of view to generate a depth map (shadow map) [S1, unofficial]. This depth map determines if a point is occluded by another object from the light's perspective [S1, unofficial]. It sets itself apart from sibling methods like [Shadow Ray Casting](2796-shadow-ray-casting.md) by using a pre-rendered depth buffer comparison rather than explicit ray intersection tests per fragment, and from [Two-Pass Shadow Casting (Kajiya and Von Herzen)](2196-two-pass-shadow-casting-kajiya-and-von-herzen.md) by relying on a single depth pass per light rather than a specific two-pass volume integration approach.

## How it works
**Inputs and Outputs**
*   **Inputs:** Scene geometry, light position $\mathbf{p}_L$, light direction $\boldsymbol{\omega}_L$ (for spotlights), light aperture angle $\theta$, near/far plane distances $n, f$, and a bias value $b$.
*   **Outputs:** A shadow map (depth texture) and a binary occlusion factor $S \in \{0, 1\}$ for each fragment during the final lighting pass.

**Data Structures**
*   **Shadow Map:** A 2D texture (or framebuffer object) storing depth values. For spotlights, a standard 2D texture is sufficient [S1, unofficial]. The texture format typically uses a single channel (e.g., `GL_DEPTH_COMPONENT` or `GL_R32F`) to store the linear or non-linear depth value from the light's perspective.
*   **Light Space Coordinates:** Homogeneous coordinates $\mathbf{c}_{light} = (x, y, z, w)$ resulting from transforming world positions by the light's view-projection matrix.

**Steps**

1.  **Generate Light View-Projection Matrix**
    For a spotlight, construct the view and projection matrices.
    *   Calculate the up vector $\mathbf{u}$:
        $$ \mathbf{u} = \text{normalize}\left( \text{cross}\left( \begin{cases} (0,0,1) & \text{if } |\boldsymbol{\omega}_L \cdot (0,0,1)| < 0.99 \\ (1,0,0) & \text{otherwise} \end{cases}, \boldsymbol{\omega}_L \right) \right) $$
    *   Create the view matrix $V_L$ using `lookAt` with eye at $\mathbf{p}_L$, target at $\mathbf{p}_L + \boldsymbol{\omega}_L$, and up $\mathbf{u}$ [S1, unofficial].
    *   Create the projection matrix $P_L$ using perspective projection with field of view $\theta$, aspect ratio $1.0$ (assuming square shadow map), near $n$, and far $f$ [S1, unofficial].
    *   Compute $M_{vp} = P_L V_L$.

2.  **Render Shadow Map**
    *   Bind a Framebuffer Object (FBO) with a depth attachment [S2, unofficial].
    *   Set the viewport to the shadow map resolution.
    *   Draw the scene geometry using $M_{vp}$. This populates the depth buffer with the closest distance from the light to any surface along each ray [S1, unofficial].

3.  **Sample and Compare (Final Pass)**
    For each fragment in the main rendering pass:
    *   Transform the fragment's world position $\mathbf{p}$ to light space: $\mathbf{c}_{light} = M_{vp} \mathbf{p}$.
    *   Perform perspective division to get normalized device coordinates (NDC):
        $$ \mathbf{c}_{ndc} = \left( \frac{x_{light}}{w_{light}}, \frac{y_{light}}{w_{light}}, \frac{z_{light}}{w_{light}} \right) $$
    *   Convert NDC to texture coordinates $\mathbf{tc}$ (assuming OpenGL-style depth range $[0, 1]$):
        $$ \mathbf{tc} = \left( \frac{x_{ndc}}{2} + 0.5, \frac{y_{ndc}}{2} + 0.5 \right) $$
    *   Sample the stored depth from the shadow map: $d_{map} = \text{texture}(u_{shadowMap}, \mathbf{tc})$ [S1, unofficial].
    *   Compare the fragment's light-space depth $z_{ndc}$ with the sampled depth $d_{map}$ using a bias $b$ to prevent self-shadowing artifacts (shadow acne):
        $$ S = \begin{cases} 1.0 & \text{if } z_{ndc} \ge d_{map} + b \\ 0.0 & \text{otherwise} \end{cases} $$
    *   The value $S$ is the shadow factor (1.0 = lit, 0.0 = shadow) [S1, unofficial].

**Parameters**
*   **Bias ($b$):** A small positive value added to the shadow depth comparison to avoid shadow acne caused by quantization errors [S1, unofficial]. Typical values depend on scene scale and depth precision; not given in the sources.
*   **Near/Far Planes ($n, f$):** Define the depth range of the light's frustum. Example: $n=0.01, f=3.0$ [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Shadow Acne:** Caused by precision errors where a surface incorrectly shadows itself. Mitigated by the bias parameter [S1, unofficial].
*   **Perspective Distortion:** For directional lights, perspective effects are negligible at large distances, but for spotlights or point lights, perspective is significant [S1, unofficial].
*   **Point Lights:** Require a cube map texture and rendering to all 6 faces, which is more involved than the 2D texture approach for spotlights [S1, unofficial].

## Subtopics

### [Variance Shadow Mapping](5686-variance-shadow-mapping.md)
Variance Shadow Mapping (VSM) is a shadow mapping technique that smoothens depth values to calculate the probability of occlusion using Chebyshev's inequality, producing soft shadow edges instead of the abrupt 0/1 values of naive shadow mapping. It sets itself apart from standard shadow mapping by enabling efficient soft shadows with minimal computational overhead, requiring only the storage and filtering of two depth maps.

## Operating characteristics
*   **Time Complexity:** $O(N)$ for the shadow map pass (where $N$ is the number of visible triangles) and $O(1)$ per fragment for the sampling/comparison in the final pass.
*   **Memory Complexity:** $O(W \times H)$ for the shadow map texture, where $W, H$ are the resolution of the shadow map.
*   **Performance:** On an RTX 3050 laptop, rendering 5,000,000 blocks at 4096x4096 shader resolution achieved over 200 fps with shadow mapping enabled [S2, unofficial].
*   **Strengths:** Efficient for static scenes; no per-ray intersection tests required during the final lighting pass [S1, unofficial].
*   **Limitations:** Produces hard shadows by default (aliasing at edges); requires bias tuning to avoid artifacts; perspective shadow maps can suffer from resolution variance across the frustum.
*   **When to use:** When efficient hard shadows are needed for spotlights or directional lights in real-time rendering [S1, unofficial].
*   **When not to use:** When soft shadows are required without additional techniques (like VSM); for point lights if cube map overhead is prohibitive [S1, unofficial].

## Background
Shadow mapping was introduced as a standard technique for rendering shadows in computer graphics. It involves rendering the scene from the light's point of view to generate a depth map [S1, unofficial]. The technique is widely used in real-time engines to add visual depth and realism [S2, unofficial].

## Sources
[S1] Voxel Cone Tracing - GitHub, https://raw.githubusercontent.com/righier/gidemo/master/report.pdf, fetched 2026-10-27 (unofficial)
[S2] Creating a Voxel Engine with OpenGL | Walker McGilvary, https://theslabby.github.io/2023/12/24/start-voxelengine.html, fetched 2026-10-28 (unofficial)
