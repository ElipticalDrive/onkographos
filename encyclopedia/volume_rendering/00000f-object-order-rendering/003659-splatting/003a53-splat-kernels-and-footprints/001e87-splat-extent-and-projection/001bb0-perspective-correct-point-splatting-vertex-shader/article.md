# Perspective-Correct Point Splatting Vertex Shader

Also called: Point Splatting with Perspective Correction

## Overview
This method renders point clouds as splats (quads or points) in a vertex shader, applying size attenuation based on perspective projection to ensure the visual size of points corresponds to their world-space radius [S1, unofficial]. It differs from its siblings by explicitly calculating the point size in the vertex shader using the projection matrix and depth, rather than relying on hardware extensions or post-hoc footprint evaluation [S1, unofficial][S2, unofficial]. It builds on [Splat Kernels and Footprints](8724-splat-kernels-and-footprints.md).

## How it works
**Inputs:**
- `in_position`: The world-space position of the point.
- `particle_radius`: The world-space radius of the point splat.
- `in_color`: The color of the point.
- `view_proj_matrix`: The combined view-projection matrix.
- `fov_y`: The vertical field of view in radians.
- `viewport_size`: The dimensions of the viewport (specifically height).

**Outputs:**
- `gl_Position`: Clip-space position.
- `gl_PointSize`: The size of the point in screen space.
- `VertOut`: A struct passing `particle_radius` and `in_color` to the fragment shader.

**Data Structures:**
- `VertOut`:
  - `float particle_radius`
  - `vec3 in_color`

**Steps:**
1. Transform the input position `in_position` to clip space using the `view_proj_matrix` and assign the result to `gl_Position` [S1, unofficial].
2. Calculate `gl_PointSize` using the perspective-correct formula:
   $$ \text{gl\_PointSize} = \text{viewport\_height} \times \frac{1}{\tan(\text{fov\_y} / 2)} \times \frac{\text{particle\_radius}}{\text{gl\_Position.w}} $$
   This scales the point size inversely with depth (`gl_Position.w`) [S1, unofficial].
3. Pass the `particle_radius` and `in_color` to the fragment shader via the output struct `VertOut` [S1, unofficial].

**Parameters:**
- `fov_y`: The vertical field of view in radians, used to calculate perspective scaling [S1, unofficial].
- `viewport_size`: The dimensions of the viewport, specifically `viewport_size[1]` (height) is used for scaling [S1, unofficial].
- `particle_radius`: The world-space radius of the point splat [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Vertex Shader Input
in_position: vec3
particle_radius: float
in_color: vec3
view_proj_matrix: mat4
fov_y: float
viewport_height: float

# Vertex Shader Output
struct VertOut {
    float particle_radius;
    vec3 in_color;
};

VertOut main() {
    // Step 1: Transform to clip space
    vec4 clip_pos = view_proj_matrix * vec4(in_position, 1.0);
    gl_Position = clip_pos;

    // Step 2: Calculate perspective-correct point size
    #define TAN_HALF_FOV tan(fov_y / 2.0)
    float point_size = viewport_height * (1.0 / TAN_HALF_FOV) * (particle_radius / clip_pos.w);
    gl_PointSize = point_size;

    // Step 3: Pass data to fragment shader
    VertOut out;
    out.particle_radius = particle_radius;
    out.in_color = in_color;
    return out;
}
```

**Edge Cases and Pitfalls:**
- Precision: Division by `gl_Position.w` requires careful handling of near-plane clipping to avoid division by zero or extreme values [S1, unofficial].
- Boundaries: The formula assumes a standard perspective projection matrix where `gl_Position.w` represents the negative eye-space depth [S1, unofficial].

## Subtopics

### Point Scaling via OpenGL Point Parameter
A technique to scale points automatically according to distance from the eye point using the OpenGL point parameter extension, overcoming the limitation of standard `glPointSize` which does not support perspective scaling [S2, unofficial]. It provides a conservative lower bound for point sizing to prevent holes. The formula for the lower bound is $z^2 \ge d^2 \cdot k$, where $d$ is the distance to the eye, and $k = (1 + 2\tan^2(\theta/2))^{-1}$ [S2, unofficial]. The conservative point diameter is $D = \frac{H}{2\sqrt{2} d k \tan(\theta/2)}$ [S2, unofficial]. To match this derived diameter $D$, set $D_0 = 1$, $a = 0$, $b = 0$, and $c = \frac{2 k \tan^2(\theta/2)}{H^2}$ in the OpenGL point size formula $D = D_0 \sqrt{\frac{1}{a + b d + c d^2}}$ [S2, unofficial]. A pitfall is that the point parameter extension scales according to distance $d$, not the z-coordinate; using a lower bound for $z$ ensures the point size is never smaller than the true size, preventing holes [S2, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
- Time: $O(1)$ per vertex for the vertex shader calculations.
- Memory: Minimal, requiring only the vertex attributes and uniforms.

**Measured Performance:**
- Not given in the sources.

**Scaling:**
- Scales linearly with the number of points.

**Strengths and Limitations:**
- Strengths: Provides accurate perspective-correct sizing without relying on hardware extensions.
- Limitations: Requires explicit calculation in the vertex shader, which may be less flexible than hardware-based solutions.

**When to Use:**
- Use when precise perspective-correct point sizing is required and hardware extensions are not available or preferred.

**When Not to Use:**
- Do not use when hardware acceleration for point parameters is available and sufficient for the application.

## Background
Originated in the context of hardware-accelerated point rendering of isosurfaces [S2, unofficial]. It addresses the limitations of standard `glPointSize` by providing a method to scale points according to distance from the eye point [S2, unofficial].

## Sources
[S1] arminkz/PointCloudRendering, https://github.com/arminkz/PointCloudRendering, fetched 2026-10-28 (unofficial)
[S2] Hardware Accelerated Point Rendering of Isosurfaces, http://wscg.zcu.cz/wscg2003/Papers_2003/J61.pdf, fetched 2026-10-26 (unofficial)
