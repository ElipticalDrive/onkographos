# Perspective-Correct Point Splatting Vertex Shader

Also called: Point Splatting with Perspective Correction

## Overview
This method renders point clouds or voxel centers as splats (quads or points) in a vertex shader, applying size attenuation based on perspective projection to ensure the visual size of points corresponds to their world-space radius [S1, unofficial]. It solves the problem of points appearing incorrectly sized (too large or too small) at varying depths, which occurs when using fixed viewport sizes or naive depth scaling [S1, unofficial]. It sets itself apart from siblings like the [Centralized Conics Approach](1243-centralized-conics-approach.md) by operating at the primitive level (vertex/fragment) rather than computing exact geometric intersections, and from [Exact Half-Space Clipping](3395-exact-half-space-clipping.md) by relying on hardware-accelerated point rasterization rather than explicit clipping logic.

## How it works
**Inputs and Outputs:**
*   **Input:** `in_position` (world-space position of the point), `particle_radius` (world-space radius of the splat), `in_color` (color/opacity attributes).
*   **Uniforms:** `view_proj_matrix` (combined view-projection matrix), `fov_y` (vertical field of view in radians), `viewport_size` (viewport dimensions, specifically height).
*   **Output:** `gl_Position` (clip-space position), `gl_PointSize` (rasterized point diameter in pixels), `VertOut` (struct passing `particle_radius` and `in_color` to the fragment shader).

**Data Structures:**
*   `VertOut`: A struct containing at least `particle_radius` (float) and `in_color` (vec3 or vec4).

**Steps:**
1.  Transform the input position `in_position` to clip space using the `view_proj_matrix` and assign the result to `gl_Position` [S1, unofficial].
2.  Calculate the perspective-correct point size using the formula:
    $$ \text{gl\_PointSize} = \text{viewport\_size}[1] \times \left(\frac{1}{\tan(\text{fov\_y} / 2)}\right) \times \frac{\text{particle\_radius}}{\text{gl\_Position.w}} $$
    This scales the point size inversely with depth (`gl_Position.w`) [S1, unofficial].
3.  Pass the `particle_radius` and `in_color` to the fragment shader via the `VertOut` struct [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Vertex Shader Logic
# Inputs: in_position (vec3), particle_radius (float), in_color (vec4)
# Uniforms: view_proj_matrix (mat4), fov_y (float), viewport_height (float)

# Step 1: Transform to clip space
gl_Position = view_proj_matrix * vec4(in_position, 1.0)

# Step 2: Calculate perspective-correct point size
# viewport_size[1] is the height
# fov_y is in radians
cot_fov_half = 1.0 / tan(fov_y / 2.0)
gl_PointSize = viewport_height * cot_fov_half * (particle_radius / gl_Position.w)

# Step 3: Pass data to fragment shader
# Assuming a standard output struct
out_particle_radius = particle_radius
out_color = in_color
```

**Parameters:**
*   `fov_y`: The vertical field of view in radians [S1, unofficial].
*   `viewport_size`: The dimensions of the viewport; specifically `viewport_size[1]` (height) is used for scaling [S1, unofficial].
*   `particle_radius`: The world-space radius of the point splat [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Division by `gl_Position.w` can lead to precision issues if points are extremely close to the near plane or if `gl_Position.w` is zero or negative (behind the camera). [G] Ensure points are clipped before the near plane to avoid division by zero or negative sizes.
*   **Boundaries:** The formula assumes a standard perspective projection matrix. If the projection matrix is orthographic, `gl_Position.w` will be constant (typically 1.0 or -1.0), and the perspective correction will fail to scale with depth. [G] Use orthographic scaling logic if the projection is not perspective.

## Subtopics

### Point Scaling via OpenGL Point Parameter
A technique to scale points automatically according to distance from the eye point using the OpenGL point parameter extension, overcoming the limitation of standard `glPointSize` which does not support perspective scaling [S2, unofficial]. It uses a conservative lower bound for the z-coordinate to ensure points are never rendered smaller than their true size, preventing visual holes [S2, unofficial].

*   **Definition:** Uses `glPointParameter` (available since OpenGL 1.4) to define point size as a function of distance to the eye $d$ [S2, unofficial].
*   **Formula:** The OpenGL point size formula is:
    $$ D = D_0 \sqrt{\frac{1}{a + b \cdot d + c \cdot d^2}} $$
    where $D_0$ is the user-defined diameter, $a, b, c$ are constants, and $d$ is the distance to the eye [S2, unofficial].
    To match the derived conservative diameter $D = \frac{H}{2 \sqrt{2} d k \tan(\theta/2)}$, set $D_0 = 1$, $a = 0$, $b = 0$, and $c = \frac{2 k \tan^2(\theta/2)}{H^2}$, where $k = (1 + 2 \tan^2(\theta/2))^{-1}$ and $H$ is the viewport height [S2, unofficial].
*   **Pitfall:** The extension scales according to distance $d$, not the z-coordinate. Using a lower bound for $z$ (specifically $z^2 \ge d^2 k$) ensures the point size is never smaller than the true size, preventing holes [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per point for the vertex shader calculation. The cost is dominated by the rasterization of the point primitives.
*   **Memory Complexity:** $O(1)$ additional memory per point for the vertex output struct.
*   **Performance:** Hardware-accelerated point rendering is often faster than software rendering or complex geometric clipping approaches for large point clouds [S2, unofficial].
*   **Strengths:** Simple implementation; leverages GPU hardware for point rasterization; provides visually correct perspective scaling.
*   **Limitations:** Limited to point primitives (or quads mapped to points); does not handle complex occlusion or exact geometric intersections like conic approaches; precision issues near the near plane.
*   **When to use:** When rendering large point clouds or voxel centers where exact geometric fidelity is less critical than performance and visual correctness of size.
*   **When not to use:** When exact half-space clipping or conic intersection is required for high-fidelity volume rendering.

## Background
The method originates from the need to render point clouds and isosurfaces efficiently on hardware. The specific vertex shader approach is documented in community implementations like `arminkz/PointCloudRendering` [S1, unofficial]. The theoretical basis for perspective-correct point sizing using OpenGL extensions is detailed in "Hardware Accelerated Point Rendering of Isosurfaces" [S2, unofficial], which addresses the limitations of standard `glPointSize` and provides formulas for conservative point sizing to avoid artifacts.

## Sources
[S1] arminkz/PointCloudRendering, https://github.com/arminkz/PointCloudRendering, fetched 2026-10-28 (unofficial)
[S2] Hardware Accelerated Point Rendering of Isosurfaces, http://wscg.zcu.cz/wscg2003/Papers_2003/J61.pdf, fetched 2026-10-26 (unofficial)
