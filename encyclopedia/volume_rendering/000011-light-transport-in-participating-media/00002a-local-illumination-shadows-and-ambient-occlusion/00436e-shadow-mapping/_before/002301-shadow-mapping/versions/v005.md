# Shadow Mapping

Also called: Depth Buffer Shadowing, Light Space Shadow Mapping

## Overview
Shadow Mapping is a technique to efficiently render shadows by rendering the scene from the light's point of view to generate a depth map (shadow map) [S1, S2, unofficial]. It determines if a point is occluded by another object from the light's perspective by comparing depth information from the light's perspective with depth from the camera's perspective [S2, unofficial]. It sets itself apart from its siblings by using a pre-computed depth buffer rather than ray casting or voxel traversal for visibility tests, offering high performance at the cost of resolution-dependent artifacts and bias tuning [S1, S2, unofficial].

## How it works
**Inputs:**
- Light source position $\mathbf{p}_L$ and direction $\mathbf{d}_L$ (for spotlights).
- Light aperture angle $\theta$ (for spotlights).
- Near and far plane distances $n, f$ for the light's frustum.
- Scene geometry (vertices).
- Fragment position in world space $\mathbf{p}_{world}$ (from camera rendering).

**Outputs:**
- Shadow map: A 2D texture storing depth values in light space.
- Shadow factor: A scalar value $0.0$ (shadow) or $1.0$ (lit) per fragment.

**Data Structures:**
- **Shadow Map:** A 2D texture (e.g., `GL_DEPTH_COMPONENT`) of resolution $W \times H$. Stores linear or logarithmic depth values.
- **Frame Buffer Object (FBO):** Attached to the shadow map texture, used to render the scene from the light's perspective.

**Steps:**

1.  **Generate Light View-Projection Matrix:**
    For a spotlight, construct the view matrix $V_L$ and projection matrix $P_L$.
    
    First, determine the "up" vector $\mathbf{u}$ to avoid gimbal lock if the light direction is parallel to the world up vector $\mathbf{k} = (0,0,1)$ [S1, unofficial]:
    $$
    \mathbf{u} = \begin{cases} 
    \text{normalize}(\mathbf{k} \times \mathbf{d}_L) & \text{if } |\mathbf{d}_L \cdot \mathbf{k}| < 0.99 \\
    \text{normalize}((1,0,0) \times \mathbf{d}_L) & \text{otherwise}
    \end{cases}
    $$
    
    The view matrix $V_L$ is constructed using `lookAt` with eye position $\mathbf{p}_L$, target $\mathbf{p}_L + \mathbf{d}_L$, and up vector $\mathbf{u}$ [S1, unofficial]:
    $$ V_L = \text{lookAt}(\mathbf{p}_L, \mathbf{p}_L + \mathbf{d}_L, \mathbf{u}) $$
    
    The projection matrix $P_L$ is a perspective projection with field of view $\theta$, aspect ratio $1.0$ (assuming square shadow map), near plane $n$, and far plane $f$ [S1, unofficial]:
    $$ P_L = \text{perspective}(\theta, 1.0, n, f) $$
    
    The combined light-space transformation matrix is $M_L = P_L V_L$ [S1, unofficial].

2.  **Render Shadow Map:**
    Bind the FBO attached to the shadow map texture. Clear the depth buffer. Render all scene geometry using $M_L$. This stores the minimum depth $z_{light}$ from the light source for each pixel in the shadow map [S1, S2, unofficial].

3.  **Sample Shadow Map (Fragment Shader):**
    For each fragment being shaded by the light:
    
    a. Transform the fragment's world position $\mathbf{p}_{world}$ to light clip space:
    $$ \mathbf{p}_{clip} = M_L \cdot \begin{bmatrix} \mathbf{p}_{world} \\ 1 \end{bmatrix} $$
    
    b. Perform perspective division to get normalized device coordinates (NDC):
    $$ \mathbf{p}_{ndc} = \frac{\mathbf{p}_{clip}}{p_{clip,w}} $$
    
    c. Map NDC $z$ from $[-1, 1]$ to texture space $[0, 1]$ and use $x, y$ for texture lookup [S1, unofficial]:
    $$ \mathbf{c} = \mathbf{p}_{ndc} \cdot \frac{1}{2} + \frac{1}{2} $$
    $$ \text{coords} = (\mathbf{c}_x, \mathbf{c}_y) $$
    $$ z_{frag} = \mathbf{c}_z $$
    
    d. Sample the stored depth from the shadow map:
    $$ z_{shadow} = \text{texture}(u_{shadowMap}, \text{coords}).r $$
    
    e. Compare fragment depth with shadow map depth, applying a bias $b$ to prevent self-shadowing artifacts (shadow acne) [S1, S2, unofficial]:
    $$ \text{shadowVal} = \begin{cases} 
    1.0 & \text{if } z_{frag} \ge z_{shadow} + b \\
    0.0 & \text{otherwise}
    \end{cases} $$

**Parameters:**
- **Bias ($b$):** A small positive value added to the shadow depth comparison to avoid shadow acne caused by quantization errors and precision issues [S1, unofficial]. Typical values depend on scene scale and depth buffer precision.
- **Resolution ($W \times H$):** Size of the shadow map. Higher resolution reduces aliasing but increases memory and fill rate cost.
- **Near/Far Planes ($n, f$):** Define the light's frustum. Tight frustum bounds improve depth precision [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Shadow Acne:** Caused by precision errors where a surface incorrectly shadows itself. Mitigated by bias [S1, unofficial].
- **Peter Panning:** Caused by excessive bias, making shadows appear detached from objects.
- **Perspective Aliasing:** Shadow map resolution is fixed in screen space, leading to jagged edges at oblique angles or far distances.
- **Frustum Culling:** Objects outside the light's view frustum are not rendered to the shadow map and may incorrectly appear lit or shadowed if not handled.

## Subtopics

### [Variance Shadow Mapping](5686-variance-shadow-mapping.md)
Variance Shadow Mapping (VSM) is a shadow mapping technique that smoothens depth values to calculate the probability of occlusion using Chebyshev's inequality, producing soft shadow edges instead of the abrupt 0/1 values of naive shadow mapping. It sets itself apart from standard shadow mapping by enabling efficient soft shadows with minimal computational overhead, requiring only the storage and filtering of two depth maps.

## Operating characteristics
- **Time Complexity:** $O(N)$ for shadow map generation (one render pass) and $O(1)$ per fragment for shadow testing (texture lookup and comparison). Total time is dominated by the shadow map render pass and the final shading pass.
- **Memory Complexity:** $O(W \cdot H)$ for the shadow map texture.
- **Measured Performance:** On an RTX 3050 laptop, rendering 5,000,000 blocks with 4096x4096 shadow map resolution achieved over 200 fps [S2, unofficial].
- **Strengths:** Efficient, widely supported, easy to implement. Good for static scenes or scenes with limited dynamic geometry.
- **Limitations:** Resolution-dependent artifacts (aliasing, blurring). Requires bias tuning. Not suitable for soft shadows without extensions like VSM. Performance degrades with very high resolution shadow maps.
- **When to use:** Real-time rendering where performance is critical and hard shadows are acceptable.
- **When not to use:** When soft shadows are required without additional techniques, or when high precision is needed for large scenes with small objects.

## Background
Shadow mapping was introduced by Lance Williams in 1978 [S1, S2, unofficial]. It is one of the most common shadowing techniques in real-time graphics due to its efficiency and simplicity. It is widely used in game engines and real-time rendering applications [S2, unofficial].

## Sources
[S1] Voxel Cone Tracing - GitHub, https://raw.githubusercontent.com/righier/gidemo/master/report.pdf, fetched 2026-10-27 (unofficial)
[S2] Creating a Voxel Engine with OpenGL | Walker McGilvary, https://theslabby.github.io/2023/12/24/start-voxelengine.html, fetched 2026-10-28 (unofficial)
